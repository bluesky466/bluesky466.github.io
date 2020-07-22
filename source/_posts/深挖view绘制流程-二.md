title: 深挖view绘制流程(二)
date: 2020-07-22 23:11:36
tags:
  - 技术相关
  - Android
---
# Choreographer

Choreographer的中文翻译是编舞者、舞蹈编导的意思，为什么起这个名字呢？因为view的刷新和舞蹈一样是需要按着节拍来的，Choreographer就是根据VSync信号这个节拍来安排view的刷新动作。

它使用ThreadLocal单例模式，每个线程都有自己的Choreographer，靠Looper去同步：

```java
public final class Choreographer {
    ...
    private static final ThreadLocal<Choreographer> sThreadInstance = new ThreadLocal<Choreographer>() {
        @Override
        protected Choreographer initialValue() {
            Looper looper = Looper.myLooper();
            if (looper == null) {
                throw new IllegalStateException("The current thread must have a looper!");
            }
            return new Choreographer(looper, VSYNC_SOURCE_APP);
        }
    };
    ..
    public static Choreographer getInstance() {
        return sThreadInstance.get();
    }
}
```

而且Choreographer实际上不仅仅是控制view刷新，作为一个舞蹈编导需要编排多个人的动作，它也需要控制多种类型的事件的处理。

它内部有4条CallbackQueue,分别控制input、animation、traversal和commit:

```java
public static final int CALLBACK_INPUT = 0;
public static final int CALLBACK_ANIMATION = 1;
public static final int CALLBACK_TRAVERSAL = 2;
public static final int CALLBACK_COMMIT = 3;
private static final int CALLBACK_LAST = CALLBACK_COMMIT;

private Choreographer(Looper looper, int vsyncSource) {
    ...
    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
    for (int i = 0; i <= CALLBACK_LAST; i++) {
        mCallbackQueues[i] = new CallbackQueue();
    }
}
```

ViewRootImpl在requestLayout的时候就是丢到了CALLBACK\_TRAVERSAL类型的CallbackQueue里面:

```java
@Override
public void requestLayout() {
    ...
    scheduleTraversals();
    ..
}

void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        ...
    }
}
```

Choreographer会找到对应的CallbackQueue然后使用addCallbackLocked将他们按时间顺序插入:

```java
public void postCallback(int callbackType, Runnable action, Object token) {
    postCallbackDelayed(callbackType, action, token, 0);
}

public void postCallbackDelayed(int callbackType,
        Runnable action, Object token, long delayMillis) {
    ...
    postCallbackDelayedInternal(callbackType, action, token, delayMillis);
}

private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {
    ...
    final long now = SystemClock.uptimeMillis();
    final long dueTime = now + delayMillis;
    mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
    ...
}

private final class CallbackQueue {
    ...
    public void addCallbackLocked(long dueTime, Object action, Object token) {
        CallbackRecord callback = obtainCallbackLocked(dueTime, action, token);
        CallbackRecord entry = mHead;
        if (entry == null) {
            mHead = callback;
            return;
        }
        if (dueTime < entry.dueTime) {
            callback.next = entry;
            mHead = callback;
            return;
        }
        while (entry.next != null) {
            if (dueTime < entry.next.dueTime) {
                callback.next = entry.next;
                break;
            }
            entry = entry.next;
        }
        entry.next = callback;
    }
    ...
}
```

从上面代码我们可以看出来CallbackQueue是个单链表，而Choreographer里维护了四条CallbackQueue用于不同类的回调:

{% img /深挖view绘制流程二/1.png %}


插入CallbackQueue之后Choreographer就会向DisplayEventReceiver请求一个Vsync信号的监听:

```java
private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {
    ...
    scheduleFrameLocked(now);
    ...
}

private void scheduleFrameLocked(long now) {
    ...
        scheduleVsyncLocked();
    ...
}

private void scheduleVsyncLocked() {
    mDisplayEventReceiver.scheduleVsync();
}
```

DisplayEventReceiver的监听原理我们等下再看，总之调用scheduleVsync之后DisplayEventReceiver会监听一次Vsync信号，然后在接收到信号的时候回调onVsync，而Choreographer有个FrameDisplayEventReceiver内部类继承了DisplayEventReceiver并且实现了Runnable接口，它在onVsync里面就会通过Handler机制将自己同步到Looper线程去执行run方法，去调用Choreographer.doFrame:

```java
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
    ...
    @Override
    public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
        ...
        mFrame = frame;
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }

    @Override
    public void run() {
        mHavePendingVsync = false;
        doFrame(mTimestampNanos, mFrame);
    }
}
```

而Choreographer.doFrame里面就会去回调之前post的callback了:

```java
void doFrame(long frameTimeNanos, int frame) {
    ...
    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
        AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

        mFrameInfo.markInputHandlingStart();
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

        mFrameInfo.markAnimationsStart();
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

        mFrameInfo.markPerformTraversalsStart();
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    } finally {
        AnimationUtils.unlockAnimationClock();
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    ...
}
```

# DisplayEventReceiver

java层的DisplayEventReceiver基本就是个壳，都是通过jni调到native层，由native层c++的NativeDisplayEventReceiver去干活:

```java
public DisplayEventReceiver(Looper looper, int vsyncSource) {
    ...
    mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this), mMessageQueue,
            vsyncSource);
    ...
}

private void dispose(boolean finalized) {
    ...
    nativeDispose(mReceiverPtr);
    mReceiverPtr = 0;
    ...
}

 public void scheduleVsync() {
    ...
    nativeScheduleVsync(mReceiverPtr);
    ...
}
```

jni层是这样的，沟通了java层的DisplayEventReceiver和native层的NativeDisplayEventReceiver:

```c++
// 动态注册JNI回调
static const JNINativeMethod gMethods[] = {
    { "nativeInit",
            "(Ljava/lang/ref/WeakReference;Landroid/os/MessageQueue;I)J",
            (void*)nativeInit },
    { "nativeDispose",
            "(J)V",
            (void*)nativeDispose },
    { "nativeScheduleVsync", "(J)V",
            (void*)nativeScheduleVsync }
};

static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
        jobject messageQueueObj, jint vsyncSource) {
    ...

    sp<NativeDisplayEventReceiver> receiver = new NativeDisplayEventReceiver(env,
            receiverWeak, messageQueue, vsyncSource);
    ...
    return reinterpret_cast<jlong>(receiver.get());
}

static void nativeDispose(JNIEnv* env, jclass clazz, jlong receiverPtr) {
    NativeDisplayEventReceiver* receiver =
            reinterpret_cast<NativeDisplayEventReceiver*>(receiverPtr);
    receiver->dispose();
    ..
}

static void nativeScheduleVsync(JNIEnv* env, jclass clazz, jlong receiverPtr) {
    sp<NativeDisplayEventReceiver> receiver =
            reinterpret_cast<NativeDisplayEventReceiver*>(receiverPtr);
    status_t status = receiver->scheduleVsync();
    ...
}
```

我们现在开始看看scheduleVsync里面具体干了些啥，由于NativeDisplayEventReceiver是继承了DisplayEventDispatcher，而且没有重写该方法，所以我们要实际应该去看DisplayEventDispatcher::scheduleVsync。

```c++
class NativeDisplayEventReceiver : public DisplayEventDispatcher {
public:
    NativeDisplayEventReceiver(JNIEnv* env,
            jobject receiverWeak, const sp<MessageQueue>& messageQueue, jint vsyncSource);

    void dispose();

protected:
    virtual ~NativeDisplayEventReceiver();

private:
    jobject mReceiverWeakGlobal;
    sp<MessageQueue> mMessageQueue;
    DisplayEventReceiver mReceiver;

    virtual void dispatchVsync(nsecs_t timestamp, int32_t id, uint32_t count);
    virtual void dispatchHotplug(nsecs_t timestamp, int32_t id, bool connected);
};
```

```c++
status_t DisplayEventDispatcher::scheduleVsync() {
    if (!mWaitingForVsync) {
        ...
        status_t status = mReceiver.requestNextVsync();
        ...

        mWaitingForVsync = true;
    }
    return OK;
}
```

可以看到DisplayEventDispatcher::scheduleVsync又是调用mReceiver.requestNextVsync请求下一个VSync信号，这个mReceiver是DisplayEventReceiver:

```c++
DisplayEventReceiver mReceiver;
```

所以我们就继续追踪:

```c++
status_t DisplayEventReceiver::requestNextVsync() {
    if (mEventConnection != NULL) {
        mEventConnection->requestNextVsync();
        return NO_ERROR;
    }
    return NO_INIT;
}

DisplayEventReceiver::DisplayEventReceiver(ISurfaceComposer::VsyncSource vsyncSource) {
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    if (sf != NULL) {
        mEventConnection = sf->createDisplayEventConnection(vsyncSource);
        if (mEventConnection != NULL) {
            mDataChannel = std::make_unique<gui::BitTube>();
            mEventConnection->stealReceiveChannel(mDataChannel.get());
        }
    }
}
```

这里打开了一个ComposerService连接，然后实际是向这个服务请求VSync信号。Composer是作曲家的意思，实际上和之前java层的编舞者是对应的。作曲家作曲，谱写节奏，编舞者根据节奏指挥舞蹈。

上面讲的DisplayEventReceiver感觉就是个套娃架构，一层套一层。而且类的名字又很接近，所以直接追踪代码的确比较晕，看下面的时序图的话会好一些:

{% img /深挖view绘制流程二/2.png %}


# VSync信号的读取

ComposerService实际上是指的SurfaceFlinger服务的client端包装类:

```c++
/*static*/ sp<ISurfaceComposer> ComposerService::getComposerService() {
    ComposerService& instance = ComposerService::getInstance();
    ...
    if (instance.mComposerService == NULL) {
        ComposerService::getInstance().connectLocked();
        ...
    }
    return instance.mComposerService;
}

void ComposerService::connectLocked() {
    const String16 name("SurfaceFlinger");
    while (getService(name, &mComposerService) != NO_ERROR) {
        usleep(250000);
    }
    ...
}
```

所以createDisplayEventConnection最终调用到SurfaceFlinger::createDisplayEventConnection，在这个方法用mEventThread去createEventConnection，最终创建一个Connection:

```c++
sp<IDisplayEventConnection> SurfaceFlinger::createDisplayEventConnection(
        ISurfaceComposer::VsyncSource vsyncSource) {
    ...
    return mEventThread->createEventConnection();
    ...
}

sp<EventThread::Connection> EventThread::createEventConnection() const {
    return new Connection(const_cast<EventThread*>(this));
}
```

然后Connection是个安卓里面的只能指针类型(RefBase)它在第一次引用计数的时候会调用onFirstRef，在这里Connection会将自己注册到EventThread的mDisplayEventConnections列表里:

```c++
void EventThread::Connection::onFirstRef() {
    // NOTE: mEventThread doesn't hold a strong reference on us
    mEventThread->registerDisplayEventConnection(this);
}

status_t EventThread::registerDisplayEventConnection(
        const sp<EventThread::Connection>& connection) {
    ...
    mDisplayEventConnections.add(connection);
    ...
    return NO_ERROR;
}
```

requestNextVsync最终是会调用到Connection::requestNextVsync，而这里除了会调用到SurfaceFlinger::resyncWithRateLimit去请求VSync信号之外还会将设置Connection的count:

```c++
void EventThread::Connection::requestNextVsync() {
    mEventThread->requestNextVsync(this);
}

void EventThread::requestNextVsync(
        const sp<EventThread::Connection>& connection) {
    Mutex::Autolock _l(mLock);

    mFlinger.resyncWithRateLimit();

    if (connection->count < 0) {
        connection->count = 0;
        mCondition.broadcast();
    }
}
```

从注释可以看出来这个count是用来标志这次应用进程VSync信号的请求是一次性的，还是多次的:

```c++
// count >= 1 : continuous event. count is the vsync rate
// count == 0 : one-shot event that has not fired
// count ==-1 : one-shot event that fired this round / disabled
int32_t count;
```

然后mCondition.broadcast()就会唤醒EventThread的waitForEvent流程，这个流程相对比较复杂，我先将删除注释之后的完整代码贴出来，然后再详细解释:

```c++
001 Vector< sp<EventThread::Connection> > EventThread::waitForEvent(
002        DisplayEventReceiver::Event* event)
003 {
004    Mutex::Autolock _l(mLock);
005    Vector< sp<EventThread::Connection> > signalConnections;
006
007    do {
008        bool eventPending = false;
009        bool waitForVSync = false;
010
011        size_t vsyncCount = 0;
012        nsecs_t timestamp = 0;
013        for (int32_t i=0 ; i<DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES ; i++) {
014            timestamp = mVSyncEvent[i].header.timestamp;
015            if (timestamp) {
016                if (mInterceptVSyncs) {
017                    mFlinger.mInterceptor.saveVSyncEvent(timestamp);
018                }
019                *event = mVSyncEvent[i];
020                mVSyncEvent[i].header.timestamp = 0;
021                vsyncCount = mVSyncEvent[i].vsync.count;
022                break;
023            }
024        }
025
026        if (!timestamp) {
027            eventPending = !mPendingEvents.isEmpty();
028            if (eventPending) {
029                *event = mPendingEvents[0];
030                mPendingEvents.removeAt(0);
031            }
032        }
033
034        size_t count = mDisplayEventConnections.size();
035        for (size_t i=0 ; i<count ; i++) {
036            sp<Connection> connection(mDisplayEventConnections[i].promote());
037            if (connection != NULL) {
038                bool added = false;
039                if (connection->count >= 0) {
040                    waitForVSync = true;
041                    if (timestamp) {
042                        if (connection->count == 0) {
043                            connection->count = -1;
044                            signalConnections.add(connection);
045                            added = true;
046                        } else if (connection->count == 1 ||
047                                (vsyncCount % connection->count) == 0) {
048                            signalConnections.add(connection);
049                            added = true;
050                        }
051                    }
052                }
053
054                if (eventPending && !timestamp && !added) {
055                    signalConnections.add(connection);
056                }
057            } else {
058                mDisplayEventConnections.removeAt(i);
059                --i; --count;
060            }
061        }
062
063        if (timestamp && !waitForVSync) {
064            disableVSyncLocked();
065        } else if (!timestamp && waitForVSync) {
066            enableVSyncLocked();
067        }
068
069        if (!timestamp && !eventPending) {
070            if (waitForVSync) {
071                bool softwareSync = mUseSoftwareVSync;
072                nsecs_t timeout = softwareSync ? ms2ns(16) : ms2ns(1000);
073                if (mCondition.waitRelative(mLock, timeout) == TIMED_OUT) {
074                    if (!softwareSync) {
075                        ALOGW("Timed out waiting for hw vsync; faking it");
076                    }
077                    mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
078                    mVSyncEvent[0].header.id = DisplayDevice::DISPLAY_PRIMARY;
079                    mVSyncEvent[0].header.timestamp = systemTime(SYSTEM_TIME_MONOTONIC);
080                    mVSyncEvent[0].vsync.count++;
081                }
082            } else {
083                mCondition.wait(mLock);
084            }
085        }
086    } while (signalConnections.isEmpty());
087
088    return signalConnections;
089 }
```

首先一开始waitForEvent是阻塞在083行的mCondition.wait(mLock)这里等待，EventThread::requestNextVsync会将它唤醒，然后在013~024检查唤醒之前是否已经有VSync信号，我们先假设没有，那么timestamp就一直是0。

然后继续跑到039行，这个的count在EventThread::requestNextVsync已经被设置>=0了，所以会进去将waitForVSync设置成true。而041行由于timestamp是0所以不会进去。

我们假设mPendingEvents也是空的，于是eventPending也是false， 接着就继续跑到073等待VSync信号了。

这里用mCondition.waitRelative等待一段时间，其实是在等待之前调用SurfaceFlinger::resyncWithRateLimit请求的屏幕硬件VSync信号，如果信号到来的话EventThread::onVSyncEvent会被调用，然后唤醒线程:

```c++
void EventThread::onVSyncEvent(nsecs_t timestamp) {
    Mutex::Autolock _l(mLock);
    mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
    mVSyncEvent[0].header.id = 0;
    mVSyncEvent[0].header.timestamp = timestamp;
    mVSyncEvent[0].vsync.count++;
    mCondition.broadcast();
}
```

如果一直没有到来的话，等待时间结束，返回TIMED\_OUT的话也会用软件模拟一个VSync信号。

然后就会继续do-while循环跑到013~024将timestamp和参数出参event的内容给设置了，接着在041行的timestamp判断不为0，于是就会将Connection放到signalConnections，最后在while里面判断到signalConnections不为空退出循环。

# VSync信号的发送

于是乎threadLoop就能拿到event和Connection列表，将事件分发出去:

```c++
bool EventThread::threadLoop() {
    DisplayEventReceiver::Event event;
    Vector< sp<EventThread::Connection> > signalConnections;
    signalConnections = waitForEvent(&event);

    // dispatch events to listeners...
    const size_t count = signalConnections.size();
    for (size_t i=0 ; i<count ; i++) {
        const sp<Connection>& conn(signalConnections[i]);
        // now see if we still need to report this event
        status_t err = conn->postEvent(event);
        ...
    }
    return true;
}
```

Connection::postEvent实际是调用DisplayEventReceiver::sendEvents往Channel里面发送消息:

```c++
status_t EventThread::Connection::postEvent(
        const DisplayEventReceiver::Event& event) {
    ssize_t size = DisplayEventReceiver::sendEvents(&mChannel, &event, 1);
    return size < 0 ? status_t(size) : status_t(NO_ERROR);
}
```

还记得在app端createDisplayEventConnection的代码吗？我们得到Connection之后调用了stealReceiveChannel方法，它就将c/s两端的通信链路打通了:

```c++
DisplayEventReceiver::DisplayEventReceiver(ISurfaceComposer::VsyncSource vsyncSource) {
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    if (sf != NULL) {
        mEventConnection = sf->createDisplayEventConnection(vsyncSource);
        if (mEventConnection != NULL) {
            mDataChannel = std::make_unique<gui::BitTube>();
            mEventConnection->stealReceiveChannel(mDataChannel.get());
        }
    }
}

status_t EventThread::Connection::stealReceiveChannel(gui::BitTube* outChannel) {
    outChannel->setReceiveFd(mChannel.moveReceiveFd());
    return NO_ERROR;
}
```

所以s端写入event之后c端就能读取到。这个fd的监听是在DisplayEventDispatcher::initialize里面写入的,它往mLooper里面add了DataChannel的Fd:

```c++
status_t DisplayEventDispatcher::initialize() {
    ...
    int rc = mLooper->addFd(mReceiver.getFd(), 0, Looper::EVENT_INPUT,
            this, NULL);
    ...
}

int DisplayEventReceiver::getFd() const {
    if (mDataChannel == NULL)
        return NO_INIT;

    return mDataChannel->getFd();
}
```

所以当消息到来之后DisplayEventDispatcher::handleEvent就会被调用然后再使用dispatchVsync去发送VSync事件

```c++
int DisplayEventDispatcher::handleEvent(int, int events, void*) {
    ...
    nsecs_t vsyncTimestamp;
    int32_t vsyncDisplayId;
    uint32_t vsyncCount;
    if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount)) {
        ...
        dispatchVsync(vsyncTimestamp, vsyncDisplayId, vsyncCount);
    }

    return 1; // keep the callback
}
```

这个在dispatchVsync子类NativeDisplayEventReceiver中实现，它会使用jni回调java层的DisplayEventReceiver.dispatchVsync:

```c++
gDisplayEventReceiverClassInfo.dispatchVsync = GetMethodIDOrDie(env,
            gDisplayEventReceiverClassInfo.clazz, "dispatchVsync", "(JII)V");


void NativeDisplayEventReceiver::dispatchVsync(nsecs_t timestamp, int32_t id, uint32_t count) {
    JNIEnv* env = AndroidRuntime::getJNIEnv();
  ...
    env->CallVoidMethod(receiverObj.get(),
                        gDisplayEventReceiverClassInfo.dispatchVsync, timestamp, id, count);
  ...
}
```

而java层的dispatchVsync再调onVsync方法

```java
@SuppressWarnings("unused")
private void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame) {
    onVsync(timestampNanos, builtInDisplayId, frame);
}
```

而这个onVsync在我们在第一节Choreographer流程里面有讲到，它会调到Choreographer.doFrame去回调注册的Callback:

```java
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
    ...
    @Override
    public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
        ...
        mFrame = frame;
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }

    @Override
    public void run() {
        mHavePendingVsync = false;
        doFrame(mTimestampNanos, mFrame);
    }
}
```

# 总结

所以整个调用关系如下图:

{% img /深挖view绘制流程二/3.png %}

ViewRootImpl将Callback丢到Choreographer之后，通过FrameDisplayEventReceiver调到Native层的NativeDisplayEventReceiver。

然后通过Connect调用到SurfaceFlinger进程的EventThread，在这里向SurfaceFlinger请求一次VSync信号并且等待信号到来之后通过DataChannel回调到应用进程的NativeDisplayEventReceiver。

之后再通过jni调回java层FrameDisplayEventReceiver和Choreographer去执行post的Callback去执行view的布局绘制。