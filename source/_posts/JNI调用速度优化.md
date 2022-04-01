title: JNI调用速度优化
date: 2022-03-29 22:08:17
tags:
---


# FastJNI

最近在看JNI HOOK的时候看到了个叫做fastJNI的东西,它可以加速JNI方法的调用,比较有意思。

首先我们都知道RegisterNativeMethods用于动态注册JNI方法:

```
static const JNINativeMethod jniNativeMethod[] = {
        {"stringFromJNI", "()Ljava/lang/String;", (void *) (stringFromJNI)},
};

JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *javaVm, void *pVoid) {
    JNIEnv *jniEnv = nullptr;
    jint result = javaVm->GetEnv(reinterpret_cast<void **>(&jniEnv), JNI_VERSION_1_6); 
    if (result != JNI_OK) {
        return -1;
    }
    
    jclass jniClass = jniEnv->FindClass("me/linjw/demo/MainActivity");
    jniEnv->RegisterNatives(
        jniClass, 
        jniNativeMethod,
        sizeof(jniNativeMethod) / sizeof(JNINativeMethod)
    );
    return JNI_VERSION_1_6;
}
```

如果我们在方法签名的前面加上"!",就可以指定使用fastJNI的方式去调用这个native方法:

```
static const JNINativeMethod jniNativeMethod[] = {
        {"stringFromJNI", "!()Ljava/lang/String;", (void *) (stringFromJNI)},
};
```

查看RegisterNativeMethods可以知道,它实际上是给Native方法设置了kAccFastNative标志位:

```
// jni_internal.cc
static jint RegisterNativeMethods(JNIEnv* env, jclass java_class, const JNINativeMethod* methods,
                                jint method_count, bool return_errors) {
    ...
    for (jint i = 0; i < method_count; ++i) {
        const char* name = methods[i].name;
        const char* sig = methods[i].signature;
        const void* fnPtr = methods[i].fnPtr;

        ...
        bool is_fast = false;
        // Notes about fast JNI calls:
        //
        // On a normal JNI call, the calling thread usually transitions
        // from the kRunnable state to the kNative state. But if the
        // called native function needs to access any Java object, it
        // will have to transition back to the kRunnable state.
        //
        // There is a cost to this double transition. For a JNI call
        // that should be quick, this cost may dominate the call cost.
        //
        // On a fast JNI call, the calling thread avoids this double
        // transition by not transitioning from kRunnable to kNative and
        // stays in the kRunnable state.
        //
        // There are risks to using a fast JNI call because it can delay
        // a response to a thread suspension request which is typically
        // used for a GC root scanning, etc. If a fast JNI call takes a
        // long time, it could cause longer thread suspension latency
        // and GC pauses.
        //
        // Thus, fast JNI should be used with care. It should be used
        // for a JNI call that takes a short amount of time (eg. no
        // long-running loop) and does not block (eg. no locks, I/O,
        // etc.)
        //
        // A '!' prefix in the signature in the JNINativeMethod
        // indicates that it's a fast JNI call and the runtime omits the
        // thread state transition from kRunnable to kNative at the
        // entry.
        if (*sig == '!') {
            is_fast = true;
            ++sig;
        }
        ...
        m->RegisterNative(fnPtr, is_fast);
        ...
    }

    return JNI_OK;
}

// art_method.cc
void ArtMethod::RegisterNative(const void* native_method, bool is_fast) {
    CHECK(IsNative()) << PrettyMethod(this);
    CHECK(!IsFastNative()) << PrettyMethod(this);
    CHECK(native_method != nullptr) << PrettyMethod(this);
    if (is_fast) {
        SetAccessFlags(GetAccessFlags() | kAccFastNative);
    }
    SetEntryPointFromJni(native_method);
}
```

源码的注释里面也描述了fastJNI的原理:

1. java方法运行在kRunnable state，native方法运行在kNative state
2. java进入native方法，从kRunnable state切换到kNative state会消耗时间
3. 如果native方法需要调到java的代码，从kNative state切换回kRunnable state也会耗时
4. 如果在方法签名前面加上"!"可以将native方法定义成fastJNI方法
5. fastJNI方法运行在kRunnable state，避免了state的切换耗时

以我的理解是这样的,默认情况下虚拟机栈和本地方法栈在两个不同的state下,相当于退出和进入java虚拟机环境,所以会有一系列的环境的存储与恢复:

{% plantuml %}

card 普通JNI{
    frame kNativeState {
        node nativeStack [
            ---
            ---
            ---
            本地方法栈
        ]
    }

    frame kRunnableState {
        node javaStack [
            ---
            ---
            ---
            虚拟机栈
        ]
    }

    javaStack -> nativeStack : 进入JNI方法: 退出java虚拟机环境
    javaStack <- nativeStack : 调用java代码: 进入java虚拟机环境
}
{% endplantuml %}

虚拟机是c/c++写的,而fastJNI相当于在执行虚拟机栈的环境上直接调用了native方法,所以java和本地方法是直接相互调用的:

{% plantuml %}
card fastJNI {
    
    frame kRunnableState {
        node nativeStack [
            ---
            ---
            ---
            本地方法栈
        ]
        node javaStack [
            ---
            ---
            ---
            虚拟机栈
        ]
    }

    javaStack -> nativeStack : 进入JNI方法: 直接调用
    javaStack <- nativeStack : 调用java代码: 直接调用
}
{% endplantuml %}

JAVA GC的stop the work实际上只是停止了java虚拟机的世界,并没没有办法停止native层的代码。

普通jni会有java虚拟机环境的进出,单纯的执行native代码对虚拟机环境没有任何影响,所以只需要在进入虚拟机的时候判断是否已经停止。

但fastJNI由于native代码会直接调用运行java层的代码,所以stop the work的时候反而需要判断是否在fastJNI过程中,以避免stop the work的过程中java代码被native层执行。

因此fastJNI使用的时候需要注意:

> fastJNI会导致Java GC之类的线程挂起请求操作被推迟,所以fastJNI方法需要尽量的短小和不要在里面做一些阻塞操作

# @FastNative & @CriticalNative

fastJNI在安卓8.0之后就被废弃了:

```
// jni_internal.cc
static jint RegisterNativeMethods(JNIEnv* env, jclass java_class, const JNINativeMethod* methods,
                                jint method_count, bool return_errors) {
    ...
    for (jint i = 0; i < method_count; ++i) {
        const char* name = methods[i].name;
        const char* sig = methods[i].signature;
        const void* fnPtr = methods[i].fnPtr;

        ...
        if (*sig == '!') {
            is_fast = true;
            ++sig;
        }
        ...
        if (UNLIKELY(is_fast)) {
            // There are a few reasons to switch:
            // 1) We don't support !bang JNI anymore, it will turn to a hard error later.
            // 2) @FastNative is actually faster. At least 1.5x faster than !bang JNI.
            //    and switching is super easy, remove ! in C code, add annotation in .java code.
            // 3) Good chance of hitting DCHECK failures in ScopedFastNativeObjectAccess
            //    since that checks for presence of @FastNative and not for ! in the descriptor.
            LOG(WARNING) << "!bang JNI is deprecated. Switch to @FastNative for " << m->PrettyMethod();
            is_fast = false;
            // TODO: make this a hard register error in the future.
        }

        const void* final_function_ptr = m->RegisterNative(fnPtr, is_fast);
        ...
    }

    return JNI_OK;
}
```

注释上说高版本的安卓提供了@FastNative去替代fastJNI。我们从[官方文档](https://source.android.google.cn/devices/tech/dalvik/improvements?hl=zh-cn)上可以找到它和另外一个叫 @CriticalNative 的东西:

### 更快速的原生方法


使用 [@FastNative](https://android.googlesource.com/platform/libcore/+/master/dalvik/src/main/java/dalvik/annotation/optimization/FastNative.java) 和 [@CriticalNative](https://android.googlesource.com/platform/libcore/+/master/dalvik/src/main/java/dalvik/annotation/optimization/CriticalNative.java) 注解可以更快速地对 Java 原生接口 (JNI) 进行原生调用。这些内置的 ART 运行时优化可以加快 JNI 转换速度，并取代了现已弃用的 !bang JNI 标记。这些注解对非原生方法没有任何影响，并且仅适用于 bootclasspath 上的平台 Java 语言代码（无 Play 商店更新）。

@FastNative 注解支持非静态方法。如果某种方法将 jobject 作为参数或返回值进行访问，请使用此注解。

利用 @CriticalNative 注解，可更快速地运行原生方法，但存在以下限制：

- 方法必须是静态方法 - 没有参数、返回值或隐式 this 的对象。
- 仅将基元类型传递给原生方法。
- 原生方法在其函数定义中不使用 JNIEnv 和 jclass 参数。
- 方法必须使用 RegisterNatives 进行注册，而不是依靠动态 JNI 链接。

> @FastNative 和 @CriticalNative 注解在执行原生方法时会停用垃圾回收。不要与长时间运行的方法一起使用，包括通常很快但一般不受限制的方法。
> 
> 停顿垃圾回收可能会导致死锁。如果锁尚未得到本地释放（即尚未返回受管理代码），请勿在原生快速调用期间获取锁。此要求不适用于常规的 JNI 调用，因为 ART 将正执行的原生代码视为已暂停的状态。

@FastNative 可以使原生方法的性能提升高达 3 倍，而 @CriticalNative 可以使原生方法的性能提升高达 5 倍。例如，在 Nexus 6P 设备上测量的 JNI 转换如下：

|Java 原生接口 (JNI) 调用 |	执行时间（以纳秒计）|
|-|-|
|常规 JNI|115|
|!bang JNI|60|
|@FastNative|35|
|@CriticalNative|25|

### 使用@FastNative和@CriticalNative

这两个东西的效果这么好,Framework里面也大量用到了。那么当我们充分了解了它们的影响之后,可以在适当的情景下使用。

但是如果你直接import它们的话会发现,在Android Studio里面报红色的Error,找不到具体的定义:

```
import dalvik.annotation.optimization.CriticalNative;
import dalvik.annotation.optimization.FastNative;
```

原因是它们都是Hide的接口对应用层隐藏。网上有不少调用隐藏API的方式,但是可能这两个类的位置比较特别,我也没有能从隐藏接口里面找到它们。于是乎我用了一个比较取巧的方式,直接把它们的代码拷贝了下来,在自己的工程里面创建同样的package去放:

{% img /JNI调用速度优化/1.png %}

从结果来看,速度的确是有比较明显的优化的:

{% img /JNI调用速度优化/2.png %}

完整的DEMO可以到[Github](https://github.com/bluesky466/FastNativeDemo)上下载