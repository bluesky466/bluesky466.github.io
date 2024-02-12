title: Binder请求处理流程探究
date: 2024-02-12 19:15:02
tags:
    - 技术相关
    - Android
---

前段时间遇到个蛮有意思的bug,趁着春节有空刨根问底记录下来。

背景是我们有个调试工具运行在安卓机器上用于局域网内远程调试机器,其实就是用[nanohttpd](https://github.com/NanoHttpd/nanohttpd)接收http的请求进行转发。

其中有个功能是通过sse的方式实现往client的转发通知,我们的实现方式是client调用get请求,server返回一个`text/event-stream`类型的无限大response,它的body是PipedInputStream类型,当有消息需要转发的时候就往对应的PipedOutputStream写入:

```java
private fun onCreateSse(uuid: String): Response {
    val input = PipedInputStream()
    val output = PipedOutputStream(input)
    val handler = ISseHandler { event, data ->
        try {
            output.write("event: $event\ndata: ${data}\n\n".toByteArray())
        } catch (e: IOException) {
            notifyHandlers.remove(uuid)
        }
    }

    notifyHandlers[uuid] = handler
    return newFixedLengthResponse(Response.Status.OK, "text/event-stream", input, -1)
        .apply {
            addHeader("Cache-Control", "no-cache")
            addHeader("Access-Control-Allow-Origin", "*")
        }
}
```


但是有开发发现这个sse总是会在接收到某一个url的post请求后不久因为IOException而断开,一开始我以为是http server和client的socket io断开了,仔细看异常才发现是PipedInputStream.read抛出的`IOException("Pipe broken")`:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:libcore/ojluni/src/main/java/java/io/PipedInputStream.java
public synchronized int read()  throws IOException {
    ...
    readSide = Thread.currentThread();
    int trials = 2;
    while (in < 0) {
        ...
        if ((writeSide != null) && (!writeSide.isAlive()) && (--trials < 0)) {
            throw new IOException("Pipe broken"); //就是这里抛出异常了!!!
        }
        ...
        wait(1000);
        ...
    }
    int ret = buffer[out++] & 0xFF;
    if (out >= buffer.length) {
        out = 0;
    }
    if (in == out) {
        /* now empty */
        in = -1;
    }

    return ret;
}
```

从上面的代码可以看到PipedInputStream的原理其实很暴力,不断wait 1秒等待PipedOutputStream写入数据,当数据写入的时候会执行到PipedInputStream.receive,这里面会赋值writeSide和in:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:libcore/ojluni/src/main/java/java/io/PipedInputStream.java
synchronized void receive(byte b[], int off, int len)  throws IOException {
    checkStateForReceive();
    writeSide = Thread.currentThread();
    int bytesToTransfer = len;
    while (bytesToTransfer > 0) {
        if (in == out)
            awaitSpace();
        int nextTransferAmount = 0;
        if (out < in) {
            nextTransferAmount = buffer.length - in;
        } else if (in < out) {
            if (in == -1) {
                in = out = 0;
                nextTransferAmount = buffer.length - in;
            } else {
                nextTransferAmount = out - in;
            }
        }
        if (nextTransferAmount > bytesToTransfer)
            nextTransferAmount = bytesToTransfer;
        assert(nextTransferAmount > 0);
        System.arraycopy(b, off, buffer, in, nextTransferAmount);
        bytesToTransfer -= nextTransferAmount;
        off += nextTransferAmount;
        in += nextTransferAmount;
        if (in >= buffer.length) {
            in = 0;
        }
    }
}
```

从抛出异常的条件`if ((writeSide != null) && (!writeSide.isAlive()) && (--trials < 0))`看,应该就是写入端的线程已经退出了。但我们的写入端其实是注册了binder的回调给到其他进程,在binder回调里面写入的。难道是binder线程池的线程被回收了导致的?但是从复现手法来看只有接收到那一个url的post请求会导致断开,其他url的post请求则PipedInputStream能一直正常工作。按道理如果是binder线程回收的原因,应该是不会区分nanohttpd里的url处理才对。

于是我在`output.write`前添加堆栈打印确认pid然后在断开后用`kill -3`强制打印进程全部线程堆栈到`/data/anr`目录,发现写入的线程的确在断开后退出了,但是从堆栈上我看到一些奇怪的东西:

```
01-31 09:27:54.801 9805 11368 D testtest:	at xx.xx.xx.xx.HttpServer.lambda$vSE0qxyqGQt4NJp3RjyeRZx9JR8(Unknown Source:0)
...
01-31 09:27:54.801 9805 11368 D testtest:	at xx.xx.xx.xx.xx.BinderEventHelper$1.onEvent(BinderEventHelper.java:53)
01-31 09:27:54.801 9805 11368 D testtest:	at xx.xx.xx.xx.binder.aidl.IEventListener$Stub.onTransact(IEventListener.java:63)
01-31 09:27:54.801 9805 11368 D testtest:	at android.os.Binder.execTransactInternal(Binder.java:1179)
01-31 09:27:54.801 9805 11368 D testtest:	at android.os.Binder.execTransact(Binder.java: 1143)
01-31 09:27:54.801 9805 11368 D testtest:	at android.os.BinderProxy.transactNative(Native Method)
01-31 09:27:54.801 9805 11368 D testtest:	at android.os.BinderProxy.transact(BinderProxy.java:571)
01-31 09:27:54.801 9805 11368 D testtest:	at xx.xx.xx.xx.binder.aidl.IServer$Stub$Proxy.sendRequest(IServer.iava:167)
...
01-31 09:27:54.801 9805 11368 D testtest:	at xx.xx.xx.xx.HttpServer.serve(HttpServer.kt:116)
01-31 09:27:54.801 9805 11368 D testtest:	at org.nanohttpd.protocols.http.HTTPSession.execute(HTTPSession.java:142)
01-31 09:27:54.801 9805 11368 D testtest:	at org.nanohttpd.protocols.http.ClientHandler.run(ClientHandler.java:12)
01-31 09:27:54.801 9805 11368 D testtest:	at java.lang.Thread.run(Thread.java:920)
```

onEvent这个aidl回调居然不是在binder线程中回调的,而是在nanohttpd的线程中回调的。而这条线程则是在DefaultAsyncRunner里面直接new的Thread:

```java
// https://github.com/NanoHttpd/nanohttpd/blob/efb2ebf85a2b06f7c508aba9eaad5377e3a01e81/core/src/main/java/org/nanohttpd/protocols/http/threading/DefaultAsyncRunner.java
public void exec(ClientHandler clientHandler) {
    ++this.requestCount;
    this.running.add(clientHandler);
    createThread(clientHandler).start();
}

protected Thread createThread(ClientHandler clientHandler) {
    Thread t = new Thread(clientHandler);
    t.setDaemon(true);
    t.setName("NanoHttpd Request Processor (#" + this.requestCount + ")");
    return t;
}
```

这样的线程在执行完Runnable的run方法之后就退出了,无怪乎PipedInputStream.read里面wait 1秒之后会判断到writeSide线程退出而抛出IOException。

而再看仔细点,我们其实是在这个线程里使用aidl去调用其他进程的方法,只不过这个方法刚好会引发aidl的回调,而这个回调就在同一个线程回到了,所以也只有处理这一个url的请求的时候会引发IOException,毕竟其他请求不会调用到这个aidl的方法:

```
01-31 09:27:54.801 9805 11368 D testtest:	at xx.xx.xx.xx.binder.aidl.IEventListener$Stub.onTransact(IEventListener.java:63) // 这里是service端回调client端注册的linstener
01-31 09:27:54.801 9805 11368 D testtest:	at android.os.Binder.execTransactInternal(Binder.java:1179)
01-31 09:27:54.801 9805 11368 D testtest:	at android.os.Binder.execTransact(Binder.java: 1143)
01-31 09:27:54.801 9805 11368 D testtest:	at android.os.BinderProxy.transactNative(Native Method)
01-31 09:27:54.801 9805 11368 D testtest:	at android.os.BinderProxy.transact(BinderProxy.java:571)
01-31 09:27:54.801 9805 11368 D testtest:	at xx.xx.xx.xx.binder.aidl.IServer$Stub$Proxy.sendRequest(IServer.iava:167) // 这里是client端调用service端的方法
```

其实从这个结果来看是挺合常理的,我调用的方法内部会触发listener回调,那么这个回调和调用的方法在同一个线程执行。但问题是现在使用的是aidl跨进程调用的另外一个进程的方法,回调也是另外一个进程回调回来的,安卓是如何实现调用和回调在同一个线程的呢?

# binder的请求流程

由于堆栈上只有java的信息,要想解答这个问题我们还需要深入看看BinderProxy.transactNative这个native方法是如何实现跨进程调用的:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/core/jni/android_util_Binder.cpp
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
{
    ...
    IBinder* target = getBPNativeData(env, obj)->mObject.get();
    ...
    status_t err = target->transact(code, *data, reply, flags);
    ...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/native/libs/binder/BpBinder.cpp
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
	...
	status = IPCThreadState::self()->transact(binderHandle(), code, data, reply, flags);
	...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    ...
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr); // 写入数据
    ...
    // 等待响应
    if (reply) {
        err = waitForResponse(reply);
    } else {
        Parcel fakeReply;
        err = waitForResponse(&fakeReply);
    }
    ...
}
```

可以看到BinderProxy.transactNative最终是调用到IPCThreadState去实现的,从IPCThreadState的名字还有获取的方式`IPCThreadState::self()`可以知道它是和线程绑定的。而它的writeTransactionData负责发送请求,时间就是往mOut里面写入BC\_TRANSACTION这个cmd和各种调用参数打包成的binder\_transaction\_data:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
        tr.offsets_size = 0;
        tr.data.ptr.offsets = 0;
    } else {
        return (mLastError = err);
    }

    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
```

service端接收到BC\_TRANSACTION这个cmd就会回调到aidl Stub的onTransact然后返回,client则是在IPCThreadState::waitForResponse里面等待返回:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();

        IF_LOG_COMMANDS() {
            alog << "Processing waitForResponse Command: "
                << getReturnString(cmd) << endl;
        }

        switch (cmd) {
        case BR_ONEWAY_SPAM_SUSPECT:
            ...
        case BR_TRANSACTION_COMPLETE:
            ...
        case BR_DEAD_REPLY:
            ...
        case BR_FAILED_REPLY:
            ...
        case BR_FROZEN_REPLY:
            ...
        case BR_ACQUIRE_RESULT:
            ...
        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                if (err != NO_ERROR) goto finish;

                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer);
                    } else {
                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
                        freeBuffer(nullptr,
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t));
                    }
                } else {
                    freeBuffer(nullptr,
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t));
                    continue;
                }
            }
            goto finish;

        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }
	...
}
```

正常情况会直接读取到BR\_REPLY,然后就填充返回数据回到java层,但是如果service端的onTransact里面又回调回client端的方法,由于`IPCThreadState::self()`会获取到同一个IPCThreadState,然后写入的cmd是BC\_TRANSACTION而不是响应的BR\_REPLY。这个时候就会去到default的executeCommand里面处理回调到client端listener Stub的onTransact:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;

    switch ((uint32_t)cmd) {
    ...
    case BR_TRANSACTION:
        {
            ...

        }
        break;
    ...
    default:
        ALOGE("*** BAD COMMAND %d received from Binder driver\n", cmd);
        result = UNKNOWN_ERROR;
        break;
    }

    if (result != NO_ERROR) {
        mLastError = result;
    }

    return result;
}
```

所以整个流程就是在IPCThreadState::transact里面调用service端方法,然后读取service端的返回发现读取出来的是回调client的listener于是进行回调。这样一来的确调用service端方法和回调client的listener都是在IPCThreadState::transact这个函数里面完成的就在同一个线程。

而处理完listener的回调之后由于waitForResponse里面是个`while(1)`,所以就会继续去读取调用service端方法的返回了:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) { // 这里是个while(1)
        ...
        switch (cmd) {
        ...
        case BR_REPLY:
            {
                ... // 2.再读取service方法的返回
            }
            goto finish; // 3.最后退出waitForResponse

        default:
            err = executeCommand(cmd); // 1.先处理BR_TRANSACTION回调client的listener
            if (err != NO_ERROR) goto finish;
            break;
        }
    }
	...
}
```

顺带一提,service端binder线程池里最终也是通过executeCommand去处理BC\_TRANSACTION这个cmd的:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/native/libs/binder/ProcessState.cpp
class PoolThread : public Thread
{
public:
    explicit PoolThread(bool isMain)
        : mIsMain(isMain)
    {
    }

protected:
    virtual bool threadLoop()
    {
        IPCThreadState::self()->joinThreadPool(mIsMain);
        return false;
    }

    const bool mIsMain;
};

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/native/libs/binder/IPCThreadState.cpp
void IPCThreadState::joinThreadPool(bool isMain)
{
    ...
    result = getAndExecuteCommand();
    ...
}

status_t IPCThreadState::getAndExecuteCommand()
{
    ...
	result = executeCommand(cmd);
	...
}
```

再结合我之前的另外一篇[binder机制深入探究](https://blog.islinjw.cn/2019/06/22/binder%E6%9C%BA%E5%88%B6%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%A9%B6/)博客对数据传输的分析,整个binder的跨进程请求流程就清晰了。