title: JNI内存管理
date: 2020-04-08 20:29:57
tags:
    - 技术相关
    - Android
---

面试的时候遇到一些候选人的简历上写着熟悉jni,但是问的时候才发现对jni的了解仅仅是停留在java和c的方法是如何相互调用上。其实这远远称不上熟悉,这篇博客就来讲讲jni面试中经常还会问到的内存管理问题。

首先我们知道java和c的对象是不能直接共用的,例如字符串我们不能直接返回char*,而需要创建一个jstring对象:

```
std::string hello = "hello world";
jstring jstr = env->NewStringUTF(hello.c_str());
```
那问题就来了,这个jstr是我们用env去new出来的。那我们需要手动去delete吗,不delete会不会造成内存泄露?

如果需要的话,当我们需要将这个jstr返回给java层使用的时候又要怎么办呢?不delete就内存泄露,delete就野指针:

```
extern "C" JNIEXPORT jstring JNICALL
Java_me_linjw_ndkdemo_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject thiz/* this */) {
    std::string hello = "hello world";
    jstring jstr = env->NewStringUTF(hello.c_str());
    return jstr;
}
```

其实jni为了解决这个问题,设计了三种引用类型:

- 局部引用
- 全局引用
- 弱全局引用

# 局部引用

我们先从局部引用讲起,其实我们这里通过NewStringUTF创建的jstring就是局部引用,那它有什么特点呢?

我们在c层大多数调用jni方法创建的引用都是局部引用,它会别存放在一张局部引用表里。它的内存有四种释放方式:

1. 程序员可以手动调用DeleteLocalRef去释放
2. c层方法执行完成返回java层的时候,jvm会遍历局部引用表去释放
3. 使用PushLocalFrame/PopLocalFrame创建/销毁局部引用栈帧的时候,在PopLocalFrame里会释放帧内创建的引用
4. 如果使用AttachCurrentThread附加原生线程,在调用DetachCurrentThread的时候会释放该线程创建的局部引用

所以上面的问题我们就能回答了, jstr可以不用手动delete,可以等方法结束的时候jvm自己去释放(当然如果返回之后在java层将这个引用保存了起来,那也是不会立马释放内存的)

但是这样是否就意味着我们可以任性的去new对象,不用考虑任何东西呢?其实也不是,局部引用表是有大小限制的,如果new的内存太多的话可能造成局部引用表的内存溢出,例如我们在for循环里面不断创建对象:

```
std::string hello = "hello world";
for(int i = 0 ; i < 9999999 ; i ++) {
    env->NewStringUTF(hello.c_str());
}
```

这就会引起local reference table overflow:

{% img /JNI内存管理/1.png %}

所以在使用完之后一定记得调用DeleteLocalRef去释放它。

有些同学可能会说,怎么可能会有人真的直接就在循环里不断创建对象呢。其实这种溢出大多数情况发生在被循环调用的方法里面:

```
void func(JNIEnv *env) {
    std::string hello = "hello world";
    env->NewStringUTF(hello.c_str());
}

...

for(int i = 0 ; i < 9999999 ; i ++) {
    func(env);
}
```

作为一个安全的程序员,在对象不再使用的时候,立马使用DeleteLocalRef去将其释放是一个很好的习惯。

## 局部引用栈帧

如上面所说我们可能在某个函数中创建了局部引用,然后这个函数在循环中被调用,就容易出现溢出。

但是如果方法里面创建了多个局部引用,在return之前一个个去释放会显得十分繁琐:

```
void func(JNIEnv *env) {
    ...
    jstring jstr1 = env->NewStringUTF(str1.c_str());
    jstring jstr2 = env->NewStringUTF(str2.c_str());
    jstring jstr3 = env->NewStringUTF(str3.c_str());
    jstring jstr4 = env->NewStringUTF(str4.c_str());
    ...
    env->DeleteLocalRef(jstr1);
    env->DeleteLocalRef(jstr2);
    env->DeleteLocalRef(jstr3);
    env->DeleteLocalRef(jstr4);
}
```

这个时候可以考虑使用局部引用栈帧:

```
void func(JNIEnv *env) {
    env->PushLocalFrame(4);
    ...
    jstring jstr1 = env->NewStringUTF(str1.c_str());
    jstring jstr2 = env->NewStringUTF(str2.c_str());
    jstring jstr3 = env->NewStringUTF(str3.c_str());
    jstring jstr4 = env->NewStringUTF(str4.c_str());
	...
    env->PopLocalFrame(NULL);
}
```

我们在方法开头PushLocalFrame,结尾PopLocalFrame,这样整个方法就在一个局部引用帧里面,而在PopLocalFrame就会将该帧里面创建的局部引用全部释放。

有的同学可能会想到一种场景,如果需要将某个局部引用当初返回值返回怎么办?用局部引用帧会不会造成野指针?

其实jni也考虑到了这中情况,所以PopLocalFrame有一个参数:

```
jobject PopLocalFrame(jobject result)
```

这个result参数可以传入你的返回值引用,这样的话这个局部引用就会在去到父帧里面,这样就能直接返回了:

```
jstring func(JNIEnv *env) {
    env->PushLocalFrame(4);
    ...
    jstring jstr1 = env->NewStringUTF(str1.c_str());
    jstring jstr2 = env->NewStringUTF(str2.c_str());
    jstring jstr3 = env->NewStringUTF(str3.c_str());
    jstring jstr4 = env->NewStringUTF(str4.c_str());
	...
    return (jstring)env->PopLocalFrame(jstr4);
}
```

PS: 就算使用了result参数,局部引用帧里面的引用也是会失效的,所以不能直接将它返回,而是需要用PopLocalFrame为它创建的新引用,这个引用才在父帧里面。

## 多线程下的局部引用

前面三种情况我们好理解,但是第四种情况又是什么意思呢?

> 3.如果使用AttachCurrentThread附加原生线程,在调用DetachCurrentThread的时候会释放该线程创建的局部引用

我们使用JNIEnv这个数据结构去调用jni的方法创建局部引用,但是JNIEnv将用于线程本地存储,所以我们不能在线程之间共享它。

如果是java层创建的线程,那调到c层会自然传入一个JNIEnv指针,但是如果是我们在c层自己新建的线程,我们要怎么拿的这个线程的JNIEnv呢?

在讲解之前还有一个知识点要先交代,除了JNIEnv其实jni还有个很重要的数据结构JavaVM,理论上每个进程可以有多个JavaVM,但Android只允许有一个,所以JavaVM是可以在多线程间共享的。

我们在java层使用System.loadLibrary方法加载so的时候,c层的JNI_OnLoad方法会被调用,我们可以在拿到JavaVM指针并将它保存起来:

```
JavaVM* g_Vm;

JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    g_Vm = vm;
    return JNI_VERSION_1_4;
}
```

之后可以在线程中使用它的AttachCurrentThread方法附加原生线程,然后在线程结束的时候使用DetachCurrentThread去解除附加:

```
pthread_t g_pthread;
JavaVM* g_vm;

void* ThreadRun(void *data) {
    JNIEnv* env;
    g_vm->AttachCurrentThread(&env, nullptr);
    ...
    g_vm->DetachCurrentThread();
}

JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    g_vm = vm;
    return JNI_VERSION_1_4;
}

...

pthread_create(&g_pthread, NULL, ThreadRun, (void *) 1);
```

所以在AttachCurrentThread和DetachCurrentThread之间JNIEnv都是有效的,我们可以使用它去创建局部引用,而在DetachCurrentThread之后JNIEnv就失效了,同时我们用它创建的局部引用也会被回收。

# 全局引用

假设我们需要使用监听者模式在c层保存java对象的引用,并启动线程执行操作,在适当的时候通知java层。我们需要怎么做,一种<font color='red'>__错误__</font>的做法是直接将传入的jobject保存到全局变量:

```
jobject g_listener;

extern "C" JNIEXPORT void JNICALL
Java_me_linjw_ndkdemo_MainActivity_registerListener(
        JNIEnv *env,
        jobject thiz,
        jobject listener) {
    g_listener = listener; // 错误的做法!!!
}
```

原因是这里传进来的jobject其实也是局部引用,而局部引用是不能跨线程使用的。我们应该将它转换成全局引用去保存:

```
jobject g_listener;

extern "C" JNIEXPORT void JNICALL
Java_me_linjw_ndkdemo_MainActivity_registerListener(
        JNIEnv *env,
        jobject thiz,
        jobject listener) {
    g_listener = env->NewGlobalRef(listener);
}
```

顾名思义,全局引用就是全局存在的引用,只有在我们调用DeleteGlobalRef之后它才会失效。

然后这样又出现了个问题,按道理这个g_listener和listener应该指向的是同一个java对象,但是如果我们这样去判断的话是错误的:

```
if(g_listener == listener) {
	...
}
```

它们的值是不会相等的,如果要判断两个jobject是否指向同一个java对象要需要用IsSameObject去判断:

```
if(env->IsSameObject(g_listener, listener)) {
	...   
}
```

# 弱全局引用

弱全局引用和全局引用类似,可以在跨线程使用,它使用NewGlobalWeakRef创建,使用DeleteGlobalWeakRef释放。

但是弱全局引用是会被gc回收的,所以在使用的时候我们需要先判断它是否已经被回收:

```
if(!env->IsSameObject(g_listener, NULL)) {
	...   
}
```

JNI中的NULL引用指向JVM中的null对象。