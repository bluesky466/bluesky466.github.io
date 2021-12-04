title: Bitmap内存回收机制
date: 2021-12-04 14:09:55
tags:
    - 技术相关
    - Android
---

Bitmap可以说是安卓里面最常见的内存消耗大户了，我们开发过程中遇到的oom问题很多都是由它引发的。谷歌官方也一直在迭代它的像素内存管理策略。从 Android 2.3.3以前的分配在native上，到2.3-7.1之间的分配在java堆上，又到8.0之后的回到native上。几度变迁，它的回收方法也在跟着变化。

# Android 2.3.3以前

2.3.3以前Bitmap的像素内存是分配在natvie上，而且不确定什么时候会被回收。根据[官方文档](https://developer.android.com/topic/performance/graphics/manage-memory)的说法我们需要手动调用Bitmap.recycle()去回收:

> 在 Android 2.3.3（API 级别 10）及更低版本上，位图的后备像素数据存储在本地内存中。它与存储在 Dalvik 堆中的位图本身是分开的。本地内存中的像素数据并不以可预测的方式释放，可能会导致应用短暂超出其内存限制并崩溃。

> 在 Android 2.3.3（API 级别 10）及更低版本上，建议使用 `recycle()`。如果您在应用中显示大量位图数据，则可能会遇到 `OutOfMemoryError` 错误。利用 `recycle()` 方法，应用可以尽快回收内存。

> **注意**：只有当您确定位图已不再使用时才应该使用 `recycle()`。如果您调用 `recycle()` 并在稍后尝试绘制位图，则会收到错误：`"Canvas: trying to use a recycled bitmap"`。

# Android 3.0~Android 7.1

虽然3.0~7.1的版本Bitmp的像素内存是分配在java堆上的，但是实际是在natvie层进行decode的，而且会在native层创建一个c++的对象和java层的Bitmap对象进行关联。

从BitmapFactory的源码我们可以看到它一路调用到nativeDecodeStream这个native方法:

```c++
// BitmapFactory.java
public static Bitmap decodeFile(String pathName, Options opts) {
	...
	stream = new FileInputStream(pathName);
	bm = decodeStream(stream, null, opts);
	...
	return bm;
}

public static Bitmap decodeStream(InputStream is, Rect outPadding, Options opts) {
	...
	bm = decodeStreamInternal(is, outPadding, opts);
	...
	return bm;
}

private static Bitmap decodeStreamInternal(InputStream is, Rect outPadding, Options opts) {
	...
	return nativeDecodeStream(is, tempStorage, outPadding, opts);
}
```

nativeDecodeStream实际上会通过jni创建java堆的内存,然后读取io流解码图片将像素数据存到这个java堆内存里面:

```c++

// BitmapFactory.cpp
static jobject nativeDecodeStream(JNIEnv* env, jobject clazz, jobject is, jbyteArray storage,
        jobject padding, jobject options) {
    ...
    bitmap = doDecode(env, bufferedStream, padding, options);
    ...
    return bitmap;
}

static jobject doDecode(JNIEnv* env, SkStreamRewindable* stream, jobject padding, jobject options) {
	...
	// outputAllocator是像素内存的分配器,会在java堆上创建内存给像素数据,可以通过BitmapFactory.Options.inBitmap复用前一个bitmap像素内存
	SkBitmap::Allocator* outputAllocator = (javaBitmap != NULL) ?
            (SkBitmap::Allocator*)&recyclingAllocator : (SkBitmap::Allocator*)&javaAllocator;
    ...
    // 将内存分配器设置给解码器
    decoder->setAllocator(outputAllocator);
    ...
    //解码
    if (decoder->decode(stream, &decodingBitmap, prefColorType, decodeMode)
                != SkImageDecoder::kSuccess) {
        return nullObjectReturn("decoder->decode returned false");
    }
    ...
	return GraphicsJNI::createBitmap(env, javaAllocator.getStorageObjAndReset(),
            bitmapCreateFlags, ninePatchChunk, ninePatchInsets, -1);
}

// Graphics.cpp
jobject GraphicsJNI::createBitmap(JNIEnv* env, android::Bitmap* bitmap,
        int bitmapCreateFlags, jbyteArray ninePatchChunk, jobject ninePatchInsets,
        int density) {

    // java层的Bitmap对象实际上是natvie层new出来的
    // native层也会创建一个android::Bitmap对象与java层的Bitmap对象绑定
    // bitmap->javaByteArray()代码bitmap的像素数据其实是存在java层的byte数组中
    jobject obj = env->NewObject(gBitmap_class, gBitmap_constructorMethodID,
            reinterpret_cast<jlong>(bitmap), bitmap->javaByteArray(),
            bitmap->width(), bitmap->height(), density, isMutable, isPremultiplied,
            ninePatchChunk, ninePatchInsets);
    ...
    return obj;
}
```

我们可以看最后会调用javaAllocator.getStorageObjAndReset()创建一个android::Bitmap类型的native层Bitmap对象，然后通过jni调用java层的Bitmap构造函数去创建java层的Bitmap对象，同时将native层的Bitmap对象保存到mNativePtr:

```java

// Bitmap.java
// Convenience for JNI access
private final long mNativePtr;

/**
 * Private constructor that must received an already allocated native bitmap
 * int (pointer).
 */
// called from JNI
Bitmap(long nativeBitmap, byte[] buffer, int width, int height, int density,
        boolean isMutable, boolean requestPremultiplied,
        byte[] ninePatchChunk, NinePatch.InsetStruct ninePatchInsets) {
    ...
    mNativePtr = nativeBitmap;
    ...
}
```

从上面的源码我们也能看出来，Bitmap的像素是存在java堆的，所以如果bitmap没有人使用了，垃圾回收器就能自动回收这块的内存，但是在native创建出来的nativeBitmap要怎么回收呢？从Bitmap的源码我们可以看到在Bitmap构造函数里面还会创建一个BitmapFinalizer去管理nativeBitmap:

```java
/**
 * Private constructor that must received an already allocated native bitmap
 * int (pointer).
 */
// called from JNI
Bitmap(long nativeBitmap, byte[] buffer, int width, int height, int density,
        boolean isMutable, boolean requestPremultiplied,
        byte[] ninePatchChunk, NinePatch.InsetStruct ninePatchInsets) {
    ...
    mNativePtr = nativeBitmap;
    mFinalizer = new BitmapFinalizer(nativeBitmap);
    ...
}
```

BitmapFinalizer的原理十分简单。Bitmap对象被销毁的时候BitmapFinalizer也会同步被销毁，然后就可以在BitmapFinalizer.finalize()里面销毁native层的nativeBitmap:

```java
private static class BitmapFinalizer {
    private long mNativeBitmap;
    ...
    BitmapFinalizer(long nativeBitmap) {
        mNativeBitmap = nativeBitmap;
    }
    ...
    @Override
    public void finalize() {
        try {
            super.finalize();
        } catch (Throwable t) {
            // Ignore
        } finally {
            setNativeAllocationByteCount(0);
            nativeDestructor(mNativeBitmap);
            mNativeBitmap = 0;
        }
    }
}
```

# Android 8.0之后

8.0以后像素内存又被放回了native上，所以依然需要在java层的Bitmap对象回收之后同步回收native的内存。

虽然BitmapFinalizer同样可以实现，但是Java的finalize方法实际上是不推荐使用的，所以谷歌也换了NativeAllocationRegistry去实现:

```java
/**
 * Private constructor that must received an already allocated native bitmap
 * int (pointer).
 */
// called from JNI
Bitmap(long nativeBitmap, int width, int height, int density,
        boolean isMutable, boolean requestPremultiplied,
    ...
    mNativePtr = nativeBitmap;
    long nativeSize = NATIVE_ALLOCATION_SIZE + getAllocationByteCount();
    NativeAllocationRegistry registry = new NativeAllocationRegistry(
        Bitmap.class.getClassLoader(), nativeGetNativeFinalizer(), nativeSize);
    registry.registerNativeAllocation(this, nativeBitmap);
}
```

NativeAllocationRegistry底层实际上使用了sun.misc.Cleaner,可以为对象注册一个清理的Runnable。当对象内存被回收的时候jvm就会调用它。

```java
import sun.misc.Cleaner;

public Runnable registerNativeAllocation(Object referent, Allocator allocator) {
    ...
    CleanerThunk thunk = new CleanerThunk();
    Cleaner cleaner = Cleaner.create(referent, thunk);
    ..
}

private class CleanerThunk implements Runnable {
    ...
    public void run() {
        if (nativePtr != 0) {
            applyFreeFunction(freeFunction, nativePtr);
        }
        registerNativeFree(size);
    }
    ...
}
```

这个Cleaner的原理也很暴力,首先它是一个虚引用,registerNativeAllocation实际上创建了一个Bitmap的虚引用:

```java
// Cleaner.java
public class Cleaner extends PhantomReference {
    ...
    public static Cleaner create(Object ob, Runnable thunk) {
        ...
        return add(new Cleaner(ob, thunk));
    }
    ...
    private Cleaner(Object referent, Runnable thunk) {
        super(referent, dummyQueue);
        this.thunk = thunk;
    }
    ...
    public void clean() {
        ...
        thunk.run();
        ...
    }
    ...
}
```

虚引用的话我们都知道需要配合一个ReferenceQueue使用，当对象的引用被回收的时候，jvm就会将这个虚引用丢到ReferenceQueue里面。而ReferenceQueue在插入的时候居然通过instanceof判断了下是不是Cleaner:

```java
// ReferenceQueue.java
private boolean enqueueLocked(Reference<? extends T> r) {
    ...
    if (r instanceof Cleaner) {
        Cleaner cl = (sun.misc.Cleaner) r;
        cl.clean();
        ...
    }
    ...
}
```

也就是说Bitmap对象被回收,就会触发Cleaner这个虚引用被丢入ReferenceQueue，而ReferenceQueue里面会判断丢进来的虚引用是不是Cleaner，如果是就调用Cleaner.clean()方法。而clean方法内部就会再去执行我们注册的清理的Runnable。
