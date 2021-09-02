title: 记一个主线程卡死却没有ANR的BUG
date: 2021-09-01 22:29:18
tags: 
    - 技术相关
    - Android
---

今天测试报了个BUG，分析了一波顺利解决问题。但是感觉中间的一些思路、技巧和知识点比较有意思，所以记录下来。

# 问题定位与分析

首先这个问题是是个概率性问题，在压测整机复位功能的时候出现的。我负责的某个服务在开机的时候会自启动，测试发现某一次复位完成开机之后功能没有办法正常使用，立马叫我过去看。

1. 首先我到的时候现场是还在的，由于这是个Service，ui上看不出异常。所以adb 连接上机器之后使用PS命令查看进程，发现服务的进程是存在的
2. 其次查看log，没有发现任何的异常打印或者奔溃重启的痕迹
3. 接着查找关键日志发现异常，这个服务在子线程做完一些初始化操作之后会同步回主线程打开功能:

```java
Log.d(TAG, "child thread finish");
mHandler.sendEmptyMessage(MSG_START_FUNCTION);
```

子线程的打印找到了，而且它的下一行就是用Handler发送Message，但是主线程接的打印没有找到。

由于这部分的代码十分简单，不存在什么bug，除非Handler机制出问题了。由于我们的机器还在研发阶段，系统哥调试的时候不小心改出什么奇怪的问题也是可能的，但是我们不能一上来就这么想，要不然把问题转给系统哥也会一脸懵逼无从入手。

由于Handler的Message是逐个执行的，所以如果某个Message堵死了也会造成后面的Message没法处理。由于这次是主线程的Handler，如果我们的主线程卡死了也会出现这种问题。

但是主线程卡死的话已经十几分钟过去了也没有出现ANR，/data/anr/下面也是空的。不过我们可以使用kill -3 \<pid\>命令强制输出trace文件，查看应用当前所有线程的调用栈。然后分析主线程现在是个啥情况:

```
"main" prio=5 tid=1 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x7137cc28 self=0xe3f82a10
  | sysTid=1208 nice=0 cgrp=default sched=0/0 handle=0xf09e6470
  | state=S schedstat=( 456814323 745320630 635 ) utm=40 stm=5 core=0 HZ=100
  | stack=0xff1c8000-0xff1ca000 stackSize=8192KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x017f64da> (a java.lang.Object)
  at java.lang.Object.wait(Object.java:442)
  at java.lang.Object.wait(Object.java:568)
  at h.a.a.a.a.l.q.f(:4)
  - locked <0x017f64da> (a java.lang.Object)
  at h.a.a.a.a.l.q.e(:2)
  at d.d.a.d.f.d.b(:3)
  at d.d.a.d.f.a.run(lambda:-1)
  at android.os.Handler.handleCallback(Handler.java:938)
  at android.os.Handler.dispatchMessage(Handler.java:99)
  at android.os.Looper.loop(Looper.java:223)
  at android.app.ActivityThread.main(ActivityThread.java:7666)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:592)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:952)
```

很给力，立马就验证了问题，主线程果然卡死在Object.wait了。

但是代码里面搜索了一圈并没有直接使用这个wait方法，倒是有个第三方库的类似的操作可能会用到它。由于之前一直没报过这种问题，应该是小概率实际所以我们必须给出实锤并且解决，要不然问题的回归比较难。

代码被混淆了，虽然我们可以用mapping.txt文件来还原，但是由于这个项目的配套还不成熟，版本号机制都还没有加上去，所以找到对应的版本和mapping.txt文件比较困难。

于是我选择将apk从机器里面adb pull扣出来用[jadx](https://github.com/skylot/jadx)反编译找到h.a.a.a.a.l.q.f这个方法看看:

{% img /一个主线程卡死却没有ANR的BUG/1.png %}

可以看到一些字符串和大概的代码逻辑。和之前猜测的第三方库做对比，发现的确我们的猜测是正确的，然后一步步对应整理回整个堆栈。发现的确是第三方库的某个方法wait一直阻塞住了主线程。这应该是第三方库的bug，幸好它有个重载方法可以传入超时时间，所以我们添加了个3秒的超时时间，超时之后重试去解决问题。另外在主线程等待也不是个好的习惯，我们可以将它挪到子线程中。

# ANR原理

虽然问题解决了，但是其实还有些知识点比较有意思值得去深究。我们都知道不能在主线程不能做耗时操作，要不然会ANR。但是这个问题主线程都阻塞十几分钟了，就算我们的是Service也应该最多200s后(后台服务)就会ANR，为啥就是没有ANR呢？

我恢复堆栈之后发现，这个wait的阻塞是在Application.onCreate的时候调用的，也就是说Application.onCreate的卡顿并不会导致ANR。

我们来回顾下ANR的4种类型:

**1. KeyDispatchTimeout** : input事件在5S内没有处理完成发生ANR

**2. ServiceTimeout** : bind,create,start,unbind等操作,前台Service在20s内,后台Service在200s内没有处理完成发生ANR

**3. BroadcastTimeout** : BroadcastReceiver onReceiver处理事务时前台广播在10S内,后台广播在60s内. 没有处理完成发生ANR

**4. ProcessContentProviderPublishTimedOutLocked** : ContentProvider publish在10s内没有处理完成发 生ANR

的确上面Service、Broadcast、ContentProvider的ANR原因都是对应组件的生命周期回调超时，他们ANR的计算并没有包括Application.onCreate因为这个回调是进程的初始化，并不在四大组件中。

另外一个知识点是并没有Activity生命周期的ANR，也就是说我们在Activity的onCreate、onStart这些生命周期中阻塞并不会造成ANR。

Activity的ANR都是input事件例如按键和触摸消息处理耗时导致的。

## 定时炸弹机制

ServiceTimeout、BroadcastTimeout、ProcessContentProviderPublishTimedOutLocked的原理都是类似的

1. 在处理前使用Handler.sendMessageDelayed发送一个ANR消息
2. 在处理完成之后使用Handler.removeMessages删除ANR消息

这里可以类比成一个定时炸弹，在处理前埋下定时炸弹，只要没有再规定的时间内完成处理并且拆除炸弹，就会爆炸。

我们这里只举一个Service的例子。在AMS里面调用Service.onCreate之前会sendMessageDelayed一个SERVICE\_TIMEOUT\_MSG的Message:

```java
// AMS start service核心代码
private final void realStartServiceLocked(ServiceRecord r, ProcessRecord app, boolean execInFg) throws RemoteException {
    ...
    // 在bumpServiceExecutingLocked里面会发送SERVICE_TIMEOUT_MSG
    bumpServiceExecutingLocked(r, execInFg, "create");
    ...
    // 异步调用Service.onCreate
    app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                    app.getReportedProcState());
    ...
}

// 下面的代码追踪bumpServiceExecutingLocked是如何发生SERVICE_TIMEOUT_MSG的
private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
    ...
    scheduleServiceTimeoutLocked(r.app);
    ...
}

private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
    ...
    scheduleServiceTimeoutLocked(r.app);
    ...
}

void scheduleServiceTimeoutLocked(ProcessRecord proc) {
    if (proc.executingServices.size() == 0 || proc.thread == null) {
        return;
    }
    Message msg = mAm.mHandler.obtainMessage(
            ActivityManagerService.SERVICE_TIMEOUT_MSG);
    msg.obj = proc;
    mAm.mHandler.sendMessageDelayed(msg,
            proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
}
```

而在ActivityThread里面Service.onCreate调用完成之后会通知AMS:

```java
private void handleCreateService(CreateServiceData data) {
    ...
    // 创建service
    service = packageInfo.getAppFactory().instantiateService(cl, data.info.name, data.intent);
    // 调用onCreate生命周期
    service.onCreate();
    ...
    // 告诉AMS,Service.onCreate已经调用完成
    ActivityManager.getService().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
  ...
}
```

AMS就会再serviceDoneExecutingLocked里面拆炸弹:

```java
private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying, boolean finishing) {
    ...
    // 删除SERVICE_TIMEOUT_MSG
    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
    ...
}
```

这种机制打个比方就是歹徒(AMS)在你家装了个定时炸弹，然后威胁你去干一件事，你必须在规定时间内完成然后告诉他停止计时，要不然就会把你家炸上天(ANR)

## KeyDispatchTimeout原理

Activity的ANR并不是通过上面所说的埋定时炸弹的方式实现的，它有另外一套逻辑。

前面我们也有说Activity的生命周期是不会触发ANR的，它的ANR实际上是在处理input事件的时候产生的。例如在按键消息或者触摸消息处理里面耗时太久。

input事件的底层分发逻辑以前写过[两篇博客](https://blog.islinjw.cn/2020/01/23/Android-Input%E7%B3%BB%E7%BB%9F-%E4%B8%80-%E4%BA%8B%E4%BB%B6%E7%9A%84%E8%8E%B7%E5%8F%96%E6%B5%81%E7%A8%8B/)感兴趣的同学可以详细了解下。我们这篇来补充上input事件分发的ANR检测原理。

相关代码在native层的InputDispatcher.cpp里面，每个input事件都会唤醒Dispatcher线程进行分发处理，我们以按键消息为例:

```cpp
void InputDispatcher::dispatchOnce() {
    ...
    dispatchOnceInnerLocked(&nextWakeupTime);
    ...
}

void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    ...
    // Ready to start a new event.
  // If we don't already have a pending event, go grab one.
    if (! mPendingEvent) {
        ...
        resetANRTimeoutsLocked();
    }
    ...

    switch (mPendingEvent->type) {
        ...
        case EventEntry::TYPE_KEY: {
            ...
            done = dispatchKeyLocked(currentTime, typedEntry, &dropReason, nextWakeupTime);
            ...
        }
        ...
    }
    ...
}
```

假设我们的应用接收到了它的第一个input事件KEY\_DOWN。可以看到dispatchOnceInnerLocked里面判断如果是一个新的事件，就调用resetANRTimeoutsLocked清除ANR的标记，然后使用dispatchKeyLocked进行分发。

resetANRTimeoutsLocked里面最重要的一步是将mInputTargetWaitCause设置成INPUT\_TARGET\_WAIT\_CAUSE\_NONE:

```cpp
void InputDispatcher::resetANRTimeoutsLocked() {
    ...
    mInputTargetWaitCause = INPUT_TARGET_WAIT_CAUSE_NONE;
    ...
}
```

dispatchKeyLocked里面回去获取当前的焦点windows分发按键消息:

```cpp
bool InputDispatcher::dispatchKeyLocked(nsecs_t currentTime, KeyEntry* entry,
        DropReason* dropReason, nsecs_t* nextWakeupTime) {
    ...
    int32_t injectionResult = findFocusedWindowTargetsLocked(currentTime,
            entry, inputTargets, nextWakeupTime);
    ...
}

int32_t InputDispatcher::findFocusedWindowTargetsLocked(nsecs_t currentTime,
        const EventEntry* entry, Vector<InputTarget>& inputTargets, nsecs_t* nextWakeupTime) {
    ...
    reason = checkWindowReadyForMoreInputLocked(currentTime,
            mFocusedWindowHandle, entry, "focused");
    if (!reason.isEmpty()) {
        injectionResult = handleTargetsNotReadyLocked(currentTime, entry,
                mFocusedApplicationHandle, mFocusedWindowHandle, nextWakeupTime, reason.string());
        ...
    }
    ...
}
```

由于是一个新的事件，所以windows没有正在处理的消息。checkWindowReadyForMoreInputLocked拿到的reson是empty的，不会进入handleTargetsNotReadyLocked，而是正常向这个window分发。

如果应用处理KEY\_DOWN卡死了，那么在用户抬起手指触发KEY\_UP事件的时候mPendingEvent则不为NULL，__不会__清除ANR标记，而且checkWindowReadyForMoreInputLocked返回的reason不是empty，就会进入handleTargetsNotReadyLocked方法:

```cpp
int32_t InputDispatcher::handleTargetsNotReadyLocked(nsecs_t currentTime,
        const EventEntry* entry,
        const sp<InputApplicationHandle>& applicationHandle,
        const sp<InputWindowHandle>& windowHandle,
        nsecs_t* nextWakeupTime, const char* reason) {
    ...
    if (mInputTargetWaitCause != INPUT_TARGET_WAIT_CAUSE_APPLICATION_NOT_READY) {
        ...
        mInputTargetWaitCause = INPUT_TARGET_WAIT_CAUSE_APPLICATION_NOT_READY;
        ...
        mInputTargetWaitTimeoutTime = currentTime + timeout;
        ...
    }
    ...
    if (currentTime >= mInputTargetWaitTimeoutTime) {
        onANRLocked(currentTime, applicationHandle, windowHandle,
                entry->eventTime, mInputTargetWaitStartTime, reason);
        ...
    } else {
        *nextWakeupTime = mInputTargetWaitTimeoutTime;
        ...
    }
 }
```

我们看到这个方法里面判断mInputTargetWaitCause不是INPUT\_TARGET\_WAIT\_CAUSE\_APPLICATION\_NOT\_READY（因为KEY\_DOWN已经在resetANRTimeoutsLocked里面将它设置成INPUT\_TARGET\_WAIT\_CAUSE\_NONE了），所以会进入if里面设置mInputTargetWaitTimeoutTime和mInputTargetWaitCause。

后面的"currentTime >= mInputTargetWaitTimeoutTime"判断因为是刚设置的mInputTargetWaitTimeoutTime所以不会进入，而是会去到else里面设置nextWakeupTime，然后线程会睡眠。也就是说这个KEY\_UP时间会被延迟timeout时间再执行。

等时间到了线程被唤醒的时候mInputTargetWaitCause，已经是INPUT\_TARGET\_WAIT\_CAUSE\_APPLICATION\_NOT\_READY了，所以不会被修改，然后"currentTime >= mInputTargetWaitTimeoutTime"判断会成功进入onANRLocked触发应用的ANR。

简单来讲就是KEY\_UP事件到来的时候发现之前上个事件还没有处理完，于是延迟5s再来看看，如果这个时候上个事件依然没有处理完，则触发ANR。

这种机制有个特点就是假设你在KEY\_UP里面卡死了，但是界面是没有任何动画，也不去触发input事件。那么虽然主线程卡死了，但是无论过多久都不会报ANR。如果这个时候你再去触发input事件(例如触摸或者按键)，就会发现过多5秒就出现ANR了。

同样打个比方这种机制就像一个暴躁的恐怖分子(Input事件)去找神父(FocusWindow)忏悔,如果发现神父已经在接客了,就会过一会再来看看,如果到时候神父还是没空,就会引爆炸弹一了百了(ANR)。

# 感想

随着年纪的增长，脑子就像个长时间运行的硬盘，塞满了各种有用的没用的东西。加载速度和检索的命中率越来越低。就像是以前明明有去专门看过ANR的原理，但是看到这个问题我的第一反应也是主线程不可能卡死要不然就ANR了。所以除了各种死记硬背的八股文知识，我认为更应该重视调试技巧和解决问题能力，这才是老年程序员的核心竞争力。


