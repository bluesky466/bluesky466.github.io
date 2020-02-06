title: Android Input系统(二) 事件的分发流程
date: 2020-02-06 21:23:45
tags:
    - 技术相关
    - Android
---

[上篇笔记](http://blog.islinjw.cn/2020/01/23/Android-Input%E7%B3%BB%E7%BB%9F-%E4%B8%80-%E4%BA%8B%E4%BB%B6%E7%9A%84%E8%8E%B7%E5%8F%96%E6%B5%81%E7%A8%8B/)说到InputReader将事件传到InputDispatcher:: mInboundQueue里面并且唤醒了InputDispatcherThread。现在我们来继续看看InputDispatcherThread被唤醒之后干了什么:

首先会用haveCommandsLocked判断是否有Command,如果没有的话才去分发事件,如果有的话不分发,而是在下面的runCommandsLockedInterruptible里面执行Command:

```
void InputDispatcher::dispatchOnce() {
    nsecs_t nextWakeupTime = LONG_LONG_MAX;
    {
        AutoMutex _l(mLock);
        mDispatcherIsAliveCondition.broadcast();

        if (!haveCommandsLocked()) {
            dispatchOnceInnerLocked(&nextWakeupTime);
        }

        if (runCommandsLockedInterruptible()) {
            nextWakeupTime = LONG_LONG_MIN;
        }
    }
    ...
}
```

也就是说在dispatchOnce里面其实主要是做两件事情,执行指令或者分发事件,而且指令的优先级会比较高。由于上一篇笔记里面讲到的时候事件唤醒了InputDispatcherThread,所以我们这里先讲事件的分发:

```
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    if (! mPendingEvent) {
        if (mInboundQueue.isEmpty()) {
            ...
        } else {
            mPendingEvent = mInboundQueue.dequeueAtHead();
            ...
        }
        ...
    }

    switch (mPendingEvent->type) {
        ...

    case EventEntry::TYPE_KEY: {
        KeyEntry* typedEntry = static_cast<KeyEntry*>(mPendingEvent);
        ...
        done = dispatchKeyLocked(currentTime, typedEntry, &dropReason, nextWakeupTime);
        break;
    }

    case EventEntry::TYPE_MOTION: {
        MotionEntry* typedEntry = static_cast<MotionEntry*>(mPendingEvent);
        ...
        done = dispatchMotionLocked(currentTime, typedEntry,
                &dropReason, nextWakeupTime);
        break;
    }
    ...
    }

    if (done) {
        ...
        releasePendingEventLocked();
        ...
    }
}

void InputDispatcher::releasePendingEventLocked() {
    if (mPendingEvent) {
		...
		releaseInboundEventLocked(mPendingEvent);
		mPendingEvent = NULL;
    }
}
```

从这里可以看到如果mPendingEvent是正在处理的事件,如果是null的话就会从mInboundQueue里面拿队列头的事件,然后在后面判断事件类型,如果是按键事件就调用dispatchKeyLocked,如果是触摸事件就调用dispatchMotionLocked,然后如果完成了的话就会在releasePendingEventLocked里面将mPendingEvent设置回null。

这里我们只拿按键事件处理来分析,触摸事件的原理大同小异:

```

bool InputDispatcher::dispatchKeyLocked(nsecs_t currentTime, KeyEntry* entry,
        DropReason* dropReason, nsecs_t* nextWakeupTime) {
          ...
          Vector<InputTarget> inputTargets;
          int32_t injectionResult = findFocusedWindowTargetsLocked(currentTime,
                  entry, inputTargets, nextWakeupTime);
          ...

          addMonitoringTargetsLocked(inputTargets);

          dispatchEventLocked(currentTime, entry, inputTargets);
}
```

dispatchKeyLocked里面最主要是两个步骤,一是找到焦点窗口,将它放到inputTargets里面:

```
int32_t InputDispatcher::findFocusedWindowTargetsLocked(nsecs_t currentTime,
        const EventEntry* entry, Vector<InputTarget>& inputTargets, nsecs_t* nextWakeupTime) {
    ...

    injectionResult = INPUT_EVENT_INJECTION_SUCCEEDED;
    addWindowTargetLocked(mFocusedWindowHandle,
            InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS, BitSet32(0),
            inputTargets);
    ...
    return injectionResult;
}

void InputDispatcher::addWindowTargetLocked(const sp<InputWindowHandle>& windowHandle,
        int32_t targetFlags, BitSet32 pointerIds, Vector<InputTarget>& inputTargets) {
    inputTargets.push();

    const InputWindowInfo* windowInfo = windowHandle->getInfo();
    InputTarget& target = inputTargets.editTop();
    target.inputChannel = windowInfo->inputChannel;
    target.flags = targetFlags;
    target.xOffset = - windowInfo->frameLeft;
    target.yOffset = - windowInfo->frameTop;
    target.scaleFactor = windowInfo->scaleFactor;
    target.pointerIds = pointerIds;
}
```

然后去dispatchEventLocked里面会将事件进行分发,这块我们之后详细聊。这里可以看到dispatchEventLocked并不是只将消息分发给当前焦点窗口,而是会分发给inputTargets里面的所有监听者,另外的一些特殊需求需要监听事件的时候就能把自己注册到mMonitoringChannels里面,分发事件的时候会把mMonitoringChannels里面的channel放到inputTargets里面:

```
void InputDispatcher::addMonitoringTargetsLocked(Vector<InputTarget>& inputTargets) {
    for (size_t i = 0; i < mMonitoringChannels.size(); i++) {
        inputTargets.push();

        InputTarget& target = inputTargets.editTop();
        target.inputChannel = mMonitoringChannels[i];
        target.flags = InputTarget::FLAG_DISPATCH_AS_IS;
        target.xOffset = 0;
        target.yOffset = 0;
        target.pointerIds.clear();
        target.scaleFactor = 1.0f;
    }
}
```

# InputChannel通信

在继续讲事件分发之前我们先来看看InputDispatcher是怎么和窗口通信的。

在应用层新的顶层窗口需要被注册到WMS中是在ViewRootImpl.setView实现的,我们从这里开始一路往下追,可以看到最后会调用到InputDispatcher::registerInputChannel注册通信的channel__放到InputDispatcher::mConnectionsByFd__这个KeyedVector里面:

```
// frameworks/base/core/java/android/view/ViewRootImpl.java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
	...
	mInputChannel = new InputChannel();
	...
	res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
	...
	mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
	...
}

// frameworks/base/services/core/java/com/android/server/wm/Session.java
public class Session extends IWindowSession.Stub
        implements IBinder.DeathRecipient {
    ...
	@Override
	public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
	        int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
	        Rect outOutsets, InputChannel outInputChannel) {
	    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
	            outContentInsets, outStableInsets, outOutsets, outInputChannel);
	}
	...
}

// frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
public int addWindow(Session session, IWindow client, int seq,
        WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
        Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
        InputChannel outInputChannel) {
	...
	final WindowState win = new WindowState(this, session, client, token, parentWindow,
	        appOp[0], seq, attrs, viewVisibility, session.mUid,
	        session.mCanAddInternalSystemWindow);
	...
    win.openInputChannel(outInputChannel);
	...
}

// frameworks/base/services/core/java/com/android/server/wm/WindowState.java
void openInputChannel(InputChannel outInputChannel) {
    ...
    String name = getName();
    InputChannel[] inputChannels = InputChannel.openInputChannelPair(name);
    mInputChannel = inputChannels[0];
    mClientChannel = inputChannels[1];
    mInputWindowHandle.inputChannel = inputChannels[0];
    if (outInputChannel != null) {
        mClientChannel.transferTo(outInputChannel);
        mClientChannel.dispose();
        mClientChannel = null;
    } else {
        ...
    }
    mService.mInputManager.registerInputChannel(mInputChannel, mInputWindowHandle);
}

// frameworks/base/services/core/java/com/android/server/input/InputManagerService.java
public void registerInputChannel(InputChannel inputChannel,
        InputWindowHandle inputWindowHandle) {
    ...
    nativeRegisterInputChannel(mPtr, inputChannel, inputWindowHandle, false);
}

// frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
static void nativeRegisterInputChannel(JNIEnv* env, jclass /* clazz */,
        jlong ptr, jobject inputChannelObj, jobject inputWindowHandleObj, jboolean monitor) {
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);

    sp<InputChannel> inputChannel = android_view_InputChannel_getInputChannel(env,
            inputChannelObj);
    ...
    status_t status = im->registerInputChannel(
            env, inputChannel, inputWindowHandle, monitor);
    ...
}

// frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
status_t NativeInputManager::registerInputChannel(JNIEnv* /* env */,
        const sp<InputChannel>& inputChannel,
        const sp<InputWindowHandle>& inputWindowHandle, bool monitor) {
    return mInputManager->getDispatcher()->registerInputChannel(
            inputChannel, inputWindowHandle, monitor);
}

// frameworks/native/services/inputflinger/InputDispatcher.cpp
status_t InputDispatcher::registerInputChannel(const sp<InputChannel>& inputChannel,
        const sp<InputWindowHandle>& inputWindowHandle, bool monitor) {
		...
        sp<Connection> connection = new Connection(inputChannel, inputWindowHandle, monitor);

        int fd = inputChannel->getFd();
        mConnectionsByFd.add(fd, connection);
        ...

        // registerInputChannel里面传入的monitor是false --> nativeRegisterInputChannel(mPtr, inputChannel, inputWindowHandle, false);
        // 所以这个流程不会将窗口的channel放到mMonitoringChannels里面
        if (monitor) {
            mMonitoringChannels.push(inputChannel);
        }
        ...
}
```

时序图如下:

{% img /AndroidInput系统二/1.png %}

值得注意的是这个流程里InputDispatcher::registerInputChannel的monitor是false(原因看注释),所以不会将channel放到mMonitoringChannels,要不然分发消息的时候就不止会分发到焦点窗口而是分发到所有窗口了。

细心的同学可能会发现ViewRootImpl.setView里面new的InputChannel到了WindowState.openInputChannel里面就不再继续往下传了,而是保存到mClientChannel。而继续往下传的其实是mInputChannel:

```
// frameworks/base/services/core/java/com/android/server/wm/WindowState.java
void openInputChannel(InputChannel outInputChannel) {
    ...
    String name = getName();
    InputChannel[] inputChannels = InputChannel.openInputChannelPair(name);
    mInputChannel = inputChannels[0];
    mClientChannel = inputChannels[1];
    mInputWindowHandle.inputChannel = inputChannels[0];
    if (outInputChannel != null) {
        mClientChannel.transferTo(outInputChannel);
        mClientChannel.dispose();
        mClientChannel = null;
    } else {
        ...
    }
    mService.mInputManager.registerInputChannel(mInputChannel, mInputWindowHandle);
}
```

这两个都是是啥?我们直接来看看InputChannel.openInputChannelPair的代码:

```
// frameworks/base/core/java/android/view/InputChannel.java
public static InputChannel[] openInputChannelPair(String name) {
    ...
    return nativeOpenInputChannelPair(name);
}

// frameworks/base/core/jni/android_view_InputChannel.cpp
static jobjectArray android_view_InputChannel_nativeOpenInputChannelPair(JNIEnv* env,
        jclass clazz, jstring nameObj) {
    ...
    sp<InputChannel> serverChannel;
    sp<InputChannel> clientChannel;
    status_t result = InputChannel::openInputChannelPair(name, serverChannel, clientChannel);
	...
    env->SetObjectArrayElement(channelPair, 0, serverChannelObj);
    env->SetObjectArrayElement(channelPair, 1, clientChannelObj);
    return channelPair;
}

// frameworks/native/libs/input/InputTransport.cpp
status_t InputChannel::openInputChannelPair(const String8& name,
        sp<InputChannel>& outServerChannel, sp<InputChannel>& outClientChannel) {
    int sockets[2];
    if (socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets)) {
        ...
    }

    int bufferSize = SOCKET_BUFFER_SIZE;
    setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));

    String8 serverChannelName = name;
    serverChannelName.append(" (server)");
    outServerChannel = new InputChannel(serverChannelName, sockets[0]);

    String8 clientChannelName = name;
    clientChannelName.append(" (client)");
    outClientChannel = new InputChannel(clientChannelName, sockets[1]);
    return OK;
}
```

最后追下来,它们其实是两个socket,0是服务端,1是客户端。然后再让我们品味一下这个代码:

```
// frameworks/base/services/core/java/com/android/server/wm/WindowState.java
void openInputChannel(InputChannel outInputChannel) {
    ...
    String name = getName();
    InputChannel[] inputChannels = InputChannel.openInputChannelPair(name);
    mInputChannel = inputChannels[0];
    mClientChannel = inputChannels[1];
    mInputWindowHandle.inputChannel = inputChannels[0];
    if (outInputChannel != null) {
        mClientChannel.transferTo(outInputChannel);
        mClientChannel.dispose();
        mClientChannel = null;
    } else {
        ...
    }
    mService.mInputManager.registerInputChannel(mInputChannel, mInputWindowHandle);
}
```

它将服务端socket注册给InputDispatcher,于是InputDispatcher就可以往服务端写入数据传给WindowState。而客户端的socket和ViewRootImpl传下来的outInputChannel绑定,也就是说可以往客户端socket里面写入数据通知ViewRootImpl。没错,事件机制的消息传输靠的是socket!

大概的原理是这样的:

{% img /AndroidInput系统二/2.png %}

# 焦点窗口确定机制

好了,上一小节里面我们知道了应用启动的时候最终会调用InputDispatcher::registerInputChannel往mConnectionsByFd添加一个socket连接,那InputDispatcher又是怎样确定事件分发的时候要分发给谁的呢?

应该有同学记得事件分发时findFocusedWindowTargetsLocked里面这个mFocusedWindowHandle:

```
bool InputDispatcher::dispatchKeyLocked(nsecs_t currentTime, KeyEntry* entry,
        DropReason* dropReason, nsecs_t* nextWakeupTime) {
          ...
          Vector<InputTarget> inputTargets;
          int32_t injectionResult = findFocusedWindowTargetsLocked(currentTime,
                  entry, inputTargets, nextWakeupTime);
          ...

          addMonitoringTargetsLocked(inputTargets);

          dispatchEventLocked(currentTime, entry, inputTargets);
}

int32_t InputDispatcher::findFocusedWindowTargetsLocked(nsecs_t currentTime,
        const EventEntry* entry, Vector<InputTarget>& inputTargets, nsecs_t* nextWakeupTime) {
    ...

    injectionResult = INPUT_EVENT_INJECTION_SUCCEEDED;
    addWindowTargetLocked(mFocusedWindowHandle,
            InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS, BitSet32(0),
            inputTargets);
    ...
    return injectionResult;
}

void InputDispatcher::addWindowTargetLocked(const sp<InputWindowHandle>& windowHandle,
        int32_t targetFlags, BitSet32 pointerIds, Vector<InputTarget>& inputTargets) {
    inputTargets.push();

    const InputWindowInfo* windowInfo = windowHandle->getInfo();
    InputTarget& target = inputTargets.editTop();
    target.inputChannel = windowInfo->inputChannel;
    target.flags = targetFlags;
    target.xOffset = - windowInfo->frameLeft;
    target.yOffset = - windowInfo->frameTop;
    target.scaleFactor = windowInfo->scaleFactor;
    target.pointerIds = pointerIds;
}
```

我们来看看它是怎么被赋值的,当焦点窗口改变的时候WindowManagerService.mInputMonitor的setInputFocusL和updateInputWindowsLw会被调用,我们从这里开始追踪:

```
// frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
@Override
public int addWindow(Session session, IWindow client, int seq,
            WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
            InputChannel outInputChannel) {
	...
	if (focusChanged) {
        mInputMonitor.setInputFocusLw(mCurrentFocus, false /*updateInputWindows*/);
    }
    mInputMonitor.updateInputWindowsLw(false /*force*/);
	...
}

// frameworks/base/services/core/java/com/android/server/wm/InputMonitor.java
void updateInputWindowsLw(boolean force) {
	...
    mUpdateInputForAllWindowsConsumer.updateInputWindows(inDrag);
	...
}

// frameworks/base/services/core/java/com/android/server/wm/InputMonitor.java
private void updateInputWindows(boolean inDrag) {
	...
	mService.mInputManager.setInputWindows(mInputWindowHandles, mFocusedInputWindowHandle);
	...
}

// frameworks/base/services/core/java/com/android/server/input/InputManagerService.java
public void setInputWindows(InputWindowHandle[] windowHandles,
        InputWindowHandle focusedWindowHandle) {
    ...
    nativeSetInputWindows(mPtr, windowHandles);
}

// frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
static void nativeSetInputWindows(JNIEnv* env, jclass /* clazz */,
        jlong ptr, jobjectArray windowHandleObjArray) {
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);

    im->setInputWindows(env, windowHandleObjArray);
}

// frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
void NativeInputManager::setInputWindows(JNIEnv* env, jobjectArray windowHandleObjArray) {
    ...
    mInputManager->getDispatcher()->setInputWindows(windowHandles);
    ...
}

// frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::setInputWindows(const Vector<sp<InputWindowHandle> >& inputWindowHandles) {
	...
    mWindowHandles = inputWindowHandles;

    sp<InputWindowHandle> newFocusedWindowHandle;
    ...
    for (size_t i = 0; i < mWindowHandles.size(); i++) {
        const sp<InputWindowHandle>& windowHandle = mWindowHandles.itemAt(i);
        ...
        if (windowHandle->getInfo()->hasFocus) {
            newFocusedWindowHandle = windowHandle;
        }
        ...
        mFocusedWindowHandle = newFocusedWindowHandle;
    }
    ...
}
```

这个流程主要是更新窗口的焦点状态,然后最终调用setInputWindows,更新mWindowHandles和mFocusedWindowHandle,他们一个代表所有window的handler一个代码焦点window的handler。

时序图如下:

{% img /AndroidInput系统二/3.png %}

# 事件分发

于是我们就到了最后的具体的事件分发流程里面,大家下看看代码:

```
void InputDispatcher::dispatchEventLocked(nsecs_t currentTime,
        EventEntry* eventEntry, const Vector<InputTarget>& inputTargets) {
    ...
    for (size_t i = 0; i < inputTargets.size(); i++) {
        const InputTarget& inputTarget = inputTargets.itemAt(i);

        ssize_t connectionIndex = getConnectionIndexLocked(inputTarget.inputChannel);
        if (connectionIndex >= 0) {
            sp<Connection> connection = mConnectionsByFd.valueAt(connectionIndex);
            prepareDispatchCycleLocked(currentTime, connection, eventEntry, &inputTarget);
        }
        ...
    }
}

void InputDispatcher::prepareDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget) {
    ...
    enqueueDispatchEntriesLocked(currentTime, connection, eventEntry, inputTarget);
}

void InputDispatcher::enqueueDispatchEntriesLocked(nsecs_t currentTime,
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget) {
    bool wasEmpty = connection->outboundQueue.isEmpty();
    ...
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_IS);
    ...
    if (wasEmpty && !connection->outboundQueue.isEmpty()) {
        startDispatchCycleLocked(currentTime, connection);
    }
}

void InputDispatcher::enqueueDispatchEntryLocked(
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget,
        int32_t dispatchMode) {
    ...
    inputTargetFlags = (inputTargetFlags & ~InputTarget::FLAG_DISPATCH_MASK) | dispatchMode;

    DispatchEntry* dispatchEntry = new DispatchEntry(eventEntry,
            inputTargetFlags, inputTarget->xOffset, inputTarget->yOffset,
            inputTarget->scaleFactor);

    switch (eventEntry->type) {
        case EventEntry::TYPE_KEY: {
            KeyEntry* keyEntry = static_cast<KeyEntry*>(eventEntry);
            dispatchEntry->resolvedAction = keyEntry->action;
            dispatchEntry->resolvedFlags = keyEntry->flags;
            ...
            break;
        }
        ...
    }
    ...
    connection->outboundQueue.enqueueAtTail(dispatchEntry);
    ...
}

void InputDispatcher::startDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection) {
    ...
    while (connection->status == Connection::STATUS_NORMAL
            && !connection->outboundQueue.isEmpty()) {
        DispatchEntry* dispatchEntry = connection->outboundQueue.head;
        dispatchEntry->deliveryTime = currentTime;
        ...
        EventEntry* eventEntry = dispatchEntry->eventEntry;
        switch (eventEntry->type) {
            case EventEntry::TYPE_KEY: {
                KeyEntry* keyEntry = static_cast<KeyEntry*>(eventEntry);

                // Publish the key event.
                status = connection->inputPublisher.publishKeyEvent(dispatchEntry->seq,
                        keyEntry->deviceId, keyEntry->source,
                        dispatchEntry->resolvedAction, dispatchEntry->resolvedFlags,
                        keyEntry->keyCode, keyEntry->scanCode,
                        keyEntry->metaState, keyEntry->repeatCount, keyEntry->downTime,
                        keyEntry->eventTime);
                break;
            }
            ...
        }
        ...
        connection->outboundQueue.dequeue(dispatchEntry);
        ...
        connection->waitQueue.enqueueAtTail(dispatchEntry);
        ...
    }
}
```

上面的代码简单来说就是:

1. 遍历inputTargets拿到对应的connection
2. 在enqueueDispatchEntryLocked里面将事件放到connection->outboundQueue里面
3. 在startDispatchCycleLocked里面用connection->inputPublisher.publishKeyEvent去将消息通过socket传到应用层
4. 从connection->outboundQueue里面移出事件放到connection->waitQueue里面等待事件完成。

然后就是等待事件完成了,由于上层的事件分发机制大家都比较熟悉了我这里就不讲了。

我们回过头来看InputDispatcher::registerInputChannel,里面注册了handleReceiveCallback回调:

```
status_t InputDispatcher::registerInputChannel(const sp<InputChannel>& inputChannel,
        const sp<InputWindowHandle>& inputWindowHandle, bool monitor) {
        ...
        mLooper->addFd(fd, 0, ALOOPER_EVENT_INPUT, handleReceiveCallback, this);
        ...
}
```

当上层完成了事件的处理之后就会发送消息调用handleReceiveCallback:

```
int InputDispatcher::handleReceiveCallback(int fd, int events, void* data) {
    InputDispatcher* d = static_cast<InputDispatcher*>(data);
    ...
    d->finishDispatchCycleLocked(currentTime, connection, seq, handled);
    ...
    d->runCommandsLockedInterruptible();
    ...
}
```

这里会先调用InputDispatcher::finishDispatchCycleLocked去往mCommandQueue里面加入一个执行InputDispatcher:: doDispatchCycleFinishedLockedInterruptible的Command:

```
void InputDispatcher::finishDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection, uint32_t seq, bool handled) {
    ...
    onDispatchCycleFinishedLocked(currentTime, connection, seq, handled);
}

void InputDispatcher::onDispatchCycleFinishedLocked(
        nsecs_t currentTime, const sp<Connection>& connection, uint32_t seq, bool handled) {
    CommandEntry* commandEntry = postCommandLocked(
            & InputDispatcher::doDispatchCycleFinishedLockedInterruptible);
    commandEntry->connection = connection;
    commandEntry->eventTime = currentTime;
    commandEntry->seq = seq;
    commandEntry->handled = handled;
}

InputDispatcher::CommandEntry* InputDispatcher::postCommandLocked(Command command) {
    CommandEntry* commandEntry = new CommandEntry(command);
    mCommandQueue.enqueueAtTail(commandEntry);
    return commandEntry;
}
```

然后InputDispatcher::runCommandsLockedInterruptible会执行这个Command:

```
bool InputDispatcher::runCommandsLockedInterruptible() {
    if (mCommandQueue.isEmpty()) {
        return false;
    }

    do {
        CommandEntry* commandEntry = mCommandQueue.dequeueAtHead();

        Command command = commandEntry->command;
        (this->*command)(commandEntry);

        commandEntry->connection.clear();
        delete commandEntry;
    } while (! mCommandQueue.isEmpty());
    return true;
}
```

也就是说InputDispatcher:: doDispatchCycleFinishedLockedInterruptible会被调用,然后在里面会将消息移出connection->waitQueue:

```
void InputDispatcher::doDispatchCycleFinishedLockedInterruptible(
        CommandEntry* commandEntry) {
    sp<Connection> connection = commandEntry->connection;
    ...
    DispatchEntry* dispatchEntry = connection->findWaitQueueEntry(seq);
    ...
    if (dispatchEntry == connection->findWaitQueueEntry(seq)) {
        connection->waitQueue.dequeue(dispatchEntry);
        ...
    }
}
```

到这里整个消息的分发流程就完成了。
