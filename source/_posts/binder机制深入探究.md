title: binder机制深入探究
date: 2019-06-22 22:40:05
tags:
  - 技术相关
  - Android
---

之前有写过一篇笔记[《Android温故而知新 - AIDL》](http://blog.islinjw.cn/2017/09/26/Android%E6%B8%A9%E6%95%85%E8%80%8C%E7%9F%A5%E6%96%B0-AIDL/)从应用层分析了aidl的数据是怎么传递的,还有一篇[《Android跨进程抛异常的原理》](http://blog.islinjw.cn/2018/11/10/Android%E8%B7%A8%E8%BF%9B%E7%A8%8B%E6%8A%9B%E5%BC%82%E5%B8%B8%E7%9A%84%E5%8E%9F%E7%90%86/)分析了异常是怎样做到跨进程的。最近准备一个培训的时候又去看了下binder底层的实现原理，这里也记录下来做个笔记。

回顾下应用层的这张示意图:

{% img /binder深入探究/1.png %}

客户端用一个id指定想要调用的方法,并将参数序列化传给binder驱动。binder驱动将数据传到服务端,服务端将参数解序列化,并且调用指定的方法。再将返回值传给binder驱动。binder驱动最后将返回值传会客户端。

这就是应用层看到的binder跨进程调用的流程。那数据在binder驱动里面是怎么传递的呢？我们接下来就来一起看看。

# binder数据的一次复制

由于不同进程之间的内存是相互隔离的,一般情况下不能直接访问其他进程的数据。所以普通的ipc机制,数据先要从进程A内存复制到内核内存中,然后再复制到进程B内存,总共经历了两次复制:

{% img /binder深入探究/2.png %}

相信大家都应该有听过Binder机制传输数据只需要一次复制,那它又是怎么做到的呢?

## mmap

这里要先介绍mmap这个系统调用,mmap系统调用使得进程之间通过映射同一个普通文件实现共享内存。普通文件被映射到进程地址空间后，进程可以像访问普通内存一样对文件进行访问,往这块内存读取数据就是向文件读写数据,而不必再调用read(),write()等操作。

相信通过文件进行跨进程通信的原理大家都能理解吧:

A进程往一个文件写入数据,B进程再从这个文件将数据读取出来。

如果使用mmap系统调用的话,文件在用户看来就是一段内存,我们直接通过指针往这块内存赋值或者读取就能实现文件的读写。两个进程就能通过这种方式做跨进程通信了。

那有人就会说了,mmap实际上是一种文件读写的简化操作,用它做跨进程通信会导致频繁读写文件,效率不会很低吗？

其实mmap除了可以使用普通文件以提供内存映射IO,或者是特殊文件以提供匿名内存映射.也就是说如果我们使用的是特殊的文件的话,映射的是一块匿名的内存区域,是不涉及文件IO的。

## binder\_mmap

binder\_mmap就是安卓为了binder通讯机制专门写的一套逻辑。当使用mmap对"/dev/binder"这个文件进行映射的时候,系统会调用到注册好的binder\_mmap方法。

这个方法会在用户空间和binder内核空间各开辟一块相同大小的虚拟内存,它们的虚拟内存地址可能不一样,但他们指向的是同一块物理内存:

{% img /binder深入探究/3.png %}

也就是说Binder驱动通过指针往这块内存区域赋值,用户进程也能直接通过指针读取出来,返回来用户进程往这块内存写入的数据,binder驱动也能直接用指针读取出来。

## 一次复制的过程

当两个进程通过Binder机制进行通信,如果进程A想向进程B传输数据。当进程A将想要传输的数据告诉binder驱动,Binder驱动就会将它复制到进程B在Binder内核空间所对应的虚拟内存。这样进程B就能在自己的用户空间使用内存访问读取到传过来的数据了。

{% img /binder深入探究/4.png %}

# Binder机制的数据大小限制

我们都知道使用Intent传递数据的大小是有限制的,所以我们不能通过它去传大数据。

那为什么会有这个限制呢?

首先我们使用Intent.putExtra保存数据,然后将数据发给AMS的过程是基于binder通讯机制的。它的数据复制过程就是上一节的内容。

但是上一节没有讲到的是mmap在创建内存映射的时候需要指定映射内存的大小。也就是说我们映射出来的内存不是无限大的,是有确定大小的。这个大小在不同的进程中会有同,比如ServiceManager进程的限制是128K,而由Zygote进程fork出来的进程的大小限制是1M-8K。

由于应用层的应用都是由Zygote进程fork出来的,所以我们的应用的binder内存限制是1M-8K:

```
// frameworks/native/libs/binder/ProcessState.cpp

#define BINDER_VM_SIZE ((1 * 1024 * 1024) - sysconf(_SC_PAGE_SIZE) * 2)

ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver))
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
       ...
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        ...
}
```

那是不是说Intent能传递的数据的最大大小就是1M-8K了呢,实际上最大的大小比这个值会小一些,因为binder驱动还需要用这块内存去传一些其他的数据去指定服务端和调用的方法。

然后有没有人觉得这个值很奇怪呢？为什么不是1M？为什么要减去8K?

其实安卓源码里面最开始这个值的确是1M来着,是在后面才去掉的这8K,我们可以看看它的log:

```
commit c0c1092183ceb38dd4d70d2732dd3a743fefd567
Author: Rebecca Schultz Zavin <rebecca@android.com>
Date:   Fri Oct 30 18:39:55 2009 -0700

    Modify the binder to request 1M - 2 pages instead of 1M.  The backing store
    in the kernel requires a guard page, so 1M allocations fragment memory very
    badly.  Subtracting a couple of pages so that they fit in a power of
    two allows the kernel to make more efficient use of its virtual address space.

    Signed-off-by: Rebecca Schultz Zavin <rebecca@android.com>

diff --git a/libs/binder/ProcessState.cpp b/libs/binder/ProcessState.cpp
index d7daf7342..2d4e10ddd 100644
--- a/libs/binder/ProcessState.cpp
+++ b/libs/binder/ProcessState.cpp
@@ -41,7 +41,7 @@
 #include <sys/mman.h>
 #include <sys/stat.h>

-#define BINDER_VM_SIZE (1*1024*1024)
+#define BINDER_VM_SIZE ((1*1024*1024) - (4096 *2))

 static bool gSingleProcess = false;
```

log上解释了,减去这8K的原因是为了优化内存调用。

这里具体解释下就是,Linux的内存管理是以内存页为单位去管理的,一个内存页是4K的大小,然后计算机读取2的n次方的内存是最高效的,所以这个1M的内存大小并没有什么毛病。

但是问题出在Linux会给内存自动添加一个保护页,如果我们指定1M大小的内存的话实际上计算机在加载内存的时候需要加载1M加1页的内存,十分零散,不高效。

所以这里减去两页,也就是8K。那所有的数据加起来不足1M,每次加载内存的时候只需要直接按1M去高效加载就可以了。

# Binder的注册于查找

还有一个问题是binder驱动是怎么找到我们客户端想要调用的服务端的？

这要分两种情况,普通的服务比如我们写的service是由AMS管理的,而系统服务如AMS则是由ServiceManager管理的。

## bindSerivce过程

先来看看普通服务的管理逻辑,具体就是bindService的流程。

当有Context.bindService被调用的时候,应用会通过Binder通信向AMS请求一个服务,AMS内部维护了一个ServiceMap,当接到这个请求之后会通过Intent去这里查找对应的ServiceRecord,如果查找不到就会启动这个Service,并且获得这个Service.onBind方法返回的Binder,然后将它保存到ServiceMap中,再传给请求服务的进程。这个进程内部会去调用onServiceConnected。

{% img /binder深入探究/5.gif %}

然后等到下个客户端请求同一个服务的时候,AMS就用Intent能从ServiceMap中查到这个服务,于是就不需要再调用服务的onBind了,可以直接返回给客户端了。

{% img /binder深入探究/6.gif %}

## ServiceManager

普通的应用是通过AMS去查询Service的Binder的,但是我们知道应用和AMS之间也是通过Binder机制通信的,那AMS的Binder又是从哪里获取的呢?

答案就是ServiceManager。

系统服务进程会调用ServiceManager.addService,将服务注册到ServiceManger中。客户端调用Context.getSystemService的时候最终会调用到ServiceManager.getService获取到注册的系统服务。

其他进程和ServiceManager进程也是通过Binder机制来通信的,那么这就有个鸡生蛋蛋生鸡的问题了。ServiceManger进程的Binder又是怎么拿到的呢？

ServiceManager进程的Binder在Binder驱动中比较特殊。它的id是0,其他应用可以通过0这个具体的id(其实在源码里面是叫handler,但为了好理解这里就说id吧)去拿到ServiceManager进程的Binder

{% img /binder深入探究/7.png %}

### Context.getSystemService的原理

那是不是每一次调用Context.getSystemService都需要调用ServiceManager.getService跨进程从ServiceManager查询系统服务呢?

我们的用户应用默默为我们做了一层缓存,只有第一次查询的时候才需要调用ServiceManager.getService,之后就记录了下来,下次再查同一个服务,会从缓存中直接返回,不需要再调用ServiceManager.getService。

追踪ContextImpl.getSystemService，发现它是调用了SystemServiceRegistry.getSystemService:

```
class ContextImpl extends Context {
    ...
    @Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
    ...
```

然后我们就去查看SystemServiceRegistry.getSystemService的源码,发现它从SYSTEM_SERVICE_FETCHERS中查找到一个ServiceFetcher,然后通过这个ServiceFetcher获取:

```
final class SystemServiceRegistry {
    ...

    public static Object getSystemService(ContextImpl ctx, String name) {
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        return fetcher != null ? fetcher.getService(ctx) : null;
    }
    ...
}
```

所以这个SYSTEM_SERVICE_FETCHERS和ServiceFetcher又是怎么回事呢:

```

    static {
        ...
        registerService(Context.COUNTRY_DETECTOR, CountryDetector.class,
                new StaticServiceFetcher<CountryDetector>() {
            @Override
            public CountryDetector createService() {
                IBinder b = ServiceManager.getService(Context.COUNTRY_DETECTOR);
                return new CountryDetector(ICountryDetector.Stub.asInterface(b));
            }});
        ...
    }
    ...
    private static <T> void registerService(String serviceName, Class<T> serviceClass,
            ServiceFetcher<T> serviceFetcher) {
        ...
        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
    }
    ...
    static abstract class StaticServiceFetcher<T> implements ServiceFetcher<T> {
        private T mCachedInstance;

        @Override
        public final T getService(ContextImpl unused) {
            synchronized (StaticServiceFetcher.this) {
                if (mCachedInstance == null) {
                    mCachedInstance = createService();
                }
                return mCachedInstance;
            }
        }

        public abstract T createService();
    }
    ...
}
```

其实这是一个单例模式的变种,只有在第一次查询这个服务的时候ServiceFetcher会判断这个服务是否已经获取过,如果没有才调用createService去从ServiceManager查询,否则直接返回。

这部分内容感兴趣的同学可以《Android源码设计模式解析与实战》第二版。它的第一个源码设计模式讲的就是Context.getSystemService。