title: Jni多线程与类加载
date: 2022-06-13 20:48:04
tags:
    - 技术相关
    - Android
    - C/C++
---

在JNI中我们可以通过JNIEnv的FindClass方法查找到java的类然后进行类似反射的调用。

如果通过java代码调用的Jni函数,此时c的函数与Java运行在同一个线程中。无论是在主线程还是java启动的子线程中FindClass都能正常工作。

# native子线程加载不了自定义的Class

但如果是通过pthread\_create之类的方法在native层创建了子线程,则在这个子线程中FindClass方法查不到我们Apk中定义的class。会返回0并且在Java层抛出ClassNotFoundException:

```
Process: me.linjw.demo.jni, PID: 2759
java.lang.ClassNotFoundException: Didn't find class "me.linjw.demo.jni.MainActivity" on path: DexPathList[[directory "."],nativeLibraryDirectories=[/system/lib64, /vendor/lib64, /system/lib64, /vendor/lib64]]
       at dalvik.system.BaseDexClassLoader.findClass(BaseDexClassLoader.java:125)
       at java.lang.ClassLoader.loadClass(ClassLoader.java:379)
       at java.lang.ClassLoader.loadClass(ClassLoader.java:312)
```

我们可以通过JNIEnv的ExceptionClear方法清除java层出现的Exception,然后对返回的jclass进行判空处理防止应用崩溃:

```
jclass clazz = env->FindClass(className);
if(clazz != 0) {
    ...
}
env->ExceptionClear();
```

从上面的异常日志可以看到这里是通过BaseDexClassLoader而不是应用层常见的PathClassLoader去加载class的。我们从官方的[JNI tips](https://developer.android.com/training/articles/perf-jni#faq_FindClass)文档里面可以得到回答:

```
This usually does what you want. You can get into trouble if you create a thread yourself (perhaps by calling pthread_create and then attaching it with AttachCurrentThread). Now there are no stack frames from your application. If you call FindClass from this thread, the JavaVM will start in the "system" class loader instead of the one associated with your application, so attempts to find app-specific classes will fail.
```

实际上JNIEnv从也是通过classloader去加载类的,如果一个Jni的方法是由java调用下来的那么它将沿用java层的classloader,这个classloader是在loadLibrary的时候设置进去的:

```
public final class System {
    ...
    public static void loadLibrary(String libname) {
        Runtime.getRuntime().loadLibrary0(Reflection.getCallerClass(), libname);
    }
    ...
}

public class Runtime {
    ...
    void loadLibrary0(Class<?> fromClass, String libname) {
        ClassLoader classLoader = ClassLoader.getClassLoader(fromClass);
        loadLibrary0(classLoader, fromClass, libname);
    }
    ...
    private synchronized void loadLibrary0(ClassLoader loader, Class<?> callerClass, String libname) {
        ...
        String error = nativeLoad(filename, loader, callerClass);
        ...
    }
    ...
}
```

但如果是native创建的子线程那么它默认是和java虚拟机没有关联的,所以也就没有JNIEnv和对应的classloader。例如我们通过JavaVM的GetEnv方法是不能获取到JNIEnv的:

```
jint result = javaVM->GetEnv((void **) &env, JNI_VERSION_1_4);
// result 等于JNI_EDETACHED(-2) 
```

我们需要手动调用JavaVM的AttachCurrentThread方法将将native线程和java虚拟机相关联。在关联上java虚拟机的时候获取到的classloader实际上是系统classloader,也就是这里的BaseDexClassLoader而不是我们应用的PathClassLoader。

因此它并不能加载我们在apk里面定义的MainActivity等类,但是如果是一些系统的类比如java.lang.String、android.util.Log、android.app.Activity是可以加载到的:

```
3332  3359 D JniDemo : java/lang/String : 9
3332  3359 D JniDemo : android/util/Log : 17
3332  3359 D JniDemo : android/app/Activity : 37
3332  3359 D JniDemo : me/linjw/demo/jni/MainActivity : 0
```

# 解决方法

正常情况下我们都推荐在java层创建子线程去调用jni方法实现并发。但是有些特殊的情况可能的确需要在native中创建子线程访问java代码。

有的同学可能会说既然在native子线程中加载不到这个类,那么我们能不能在java线程中先加载出来在native子线程中使用呢?

答案是可以的,但是如果直接将jclass保存到全局引用会出现异常:

```
06-11 16:50:16.656  3507  3507 F DEBUG   : Abort message: 'java_vm_ext.cc:534] JNI DETECTED ERROR IN APPLICATION: use of deleted local reference 0x75'
```

我之前写过一篇[JNI内存管理](https://blog.islinjw.cn/2020/04/08/JNI%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/)的笔记有讲到相关的知识点,在java线程中FindClass得到的jclass是局部引用,局部引用在退出jni函数回到java代码的时候就被回收了。我们需要创建全局引用或者弱全局引用去保存:

```
clazzMainActivity = (jclass) env->NewGlobalRef(clazz);
```

之后我们就能在子线程中使用这个jclass通过类似反射的操作调用java代码了:

```
jfieldID field = env->GetStaticFieldID(clazzMainActivity, "DATA_IN_JAVA", "I");
int data = env->GetStaticIntField(clazzMainActivity, field);
LOGD("data in threadFunc : %d", data);

// 日志如下
// 3427  3427 D JniDemo : data : 123
```

完整Demo代码已上传[Github](https://github.com/bluesky466/JNIClassLoaderDemo/blob/master/app/src/main/cpp/native-lib.cpp)