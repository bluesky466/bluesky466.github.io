title: 安卓音视频播放 - NuPlayer
date: 2019-01-19 15:47:06
tags:
  - 技术相关
  - Android
  - 音视频
---

系列文章:

- [安卓音视频播放 - 总体架构](http://blog.islinjw.cn/2019/01/17/%E5%AE%89%E5%8D%93%E9%9F%B3%E8%A7%86%E9%A2%91%E6%92%AD%E6%94%BE-%E6%80%BB%E4%BD%93%E6%9E%B6%E6%9E%84/)
- [安卓音视频播放 - AwesomePlayer](http://blog.islinjw.cn/2019/01/17/%E5%AE%89%E5%8D%93%E9%9F%B3%E8%A7%86%E9%A2%91%E6%92%AD%E6%94%BE-AwesomePlayer/)
- [安卓音视频播放 - NuPlayer](http://blog.islinjw.cn/2019/01/19/%E5%AE%89%E5%8D%93%E9%9F%B3%E8%A7%86%E9%A2%91%E6%92%AD%E6%94%BE-NuPlayer/)

这里有一点需要提一下,不像StagefrightPlayerFactory直接就创建出了StagefrightPlayer, NuPlayerFactory创建出来的是NuPlayerDriver,不过NuPlayerDriver内部也是封装了NuPlayer,对NuPlayer进行调用就是了:

```
class NuPlayerFactory : public MediaPlayerFactory::IFactory {
	...

    virtual sp<MediaPlayerBase> createPlayer(pid_t pid) {
        ALOGV(" create NuPlayer");
        return new NuPlayerDriver(pid);
    }
};
```

接下来就要介绍NuPlayer了,NuPlayer的特点在于它用了Handler机制去实现子线程解码。对的,就是我们熟悉的Handler机制.只不过它是在C/C++的实现,但是原理和java层的是一样的。

看下接口声明就会感觉似曾相识了:

```
struct ALooper : public RefBase {
	...
	status_t start(bool runOnCallingThread = false,bool canCallJava = false,int32_t priority = PRIORITY_DEFAULT);
	...
    status_t stop();
    ...
    void post(const sp<AMessage> &msg, int64_t delayUs);
    ...
    bool loop();
    ...
}


struct AHandler : public RefBase {
...
protected:
    virtual void onMessageReceived(const sp<AMessage> &msg) = 0;
...
}

struct AMessage : public RefBase {
    AMessage();
    AMessage(uint32_t what, const sp<const AHandler> &handler);
	...
    void setTarget(const sp<const AHandler> &handler);

    void clear();

    void setInt32(const char *name, int32_t value);
    void setInt64(const char *name, int64_t value);
    void setSize(const char *name, size_t value);
    void setFloat(const char *name, float value);
    void setDouble(const char *name, double value);
    void setPointer(const char *name, void *value);
    void setString(const char *name, const char *s, ssize_t len = -1);
    void setString(const char *name, const AString &s);
    void setObject(const char *name, const sp<RefBase> &obj);
    void setBuffer(const char *name, const sp<ABuffer> &buffer);
    void setMessage(const char *name, const sp<AMessage> &obj);
    ...
    status_t post(int64_t delayUs = 0);
    ...
}
```

而Looper就是在NuPlayerDriver中创建并启动的

```

NuPlayerDriver::NuPlayerDriver(pid_t pid)
    : mState(STATE_IDLE),
      mIsAsyncPrepare(false),
      mAsyncResult(UNKNOWN_ERROR),
      mSetSurfaceInProgress(false),
      mDurationUs(-1),
      mPositionUs(-1),
      mSeekInProgress(false),
      mLooper(new ALooper),
      mPlayerFlags(0),
      mAtEOS(false),
      mLooping(false),
      mAutoLoop(false) {
    ALOGV("NuPlayerDriver(%p)", this);
    mLooper->setName("NuPlayerDriver Looper");

    mLooper->start(
            false, /* runOnCallingThread */
            true,  /* canCallJava */
            PRIORITY_AUDIO);

    mPlayer = new NuPlayer(pid);
    mLooper->registerHandler(mPlayer);

    mPlayer->setDriver(this);
}

NuPlayerDriver::~NuPlayerDriver() {
    ALOGV("~NuPlayerDriver(%p)", this);
    mLooper->stop();
}
```

我们的NuPlayer其实是一个AHandler:

```
struct NuPlayer : public AHandler {
	...
    virtual void onMessageReceived(const sp<AMessage> &msg);
    ...
}
```

它通过AMessage::post方法将操作放到子线程中,这操作简直不能再熟悉,甚至就没有细讲的必要:

```
void NuPlayer::setDataSourceAsync(const sp<IStreamSource> &source) {
    sp<AMessage> msg = new AMessage(kWhatSetDataSource, this);

    sp<AMessage> notify = new AMessage(kWhatSourceNotify, this);

    msg->setObject("source", new StreamingSource(notify, source));
    msg->post();
}

...

void NuPlayer::prepareAsync() {
    (new AMessage(kWhatPrepare, this))->post();
}

...

void NuPlayer::onMessageReceived(const sp<AMessage> &msg) {
	...
	switch (msg->what()) {
		case kWhatSetDataSource:
        {
            ALOGV("kWhatSetDataSource");

            CHECK(mSource == NULL);

            status_t err = OK;
            sp<RefBase> obj;
            CHECK(msg->findObject("source", &obj));
            if (obj != NULL) {
                Mutex::Autolock autoLock(mSourceLock);
                mSource = static_cast<Source *>(obj.get());
            } else {
                err = UNKNOWN_ERROR;
            }

            CHECK(mDriver != NULL);
            sp<NuPlayerDriver> driver = mDriver.promote();
            if (driver != NULL) {
                driver->notifySetDataSourceCompleted(err);
            }
            break;
        }

        case kWhatPrepare:
        {
            mSource->prepareAsync();
            break;
        }		
        ...
	}
	...
}
```

# NuPlayer::Source

NuPlayer::Source顾名思义,是数据源的意思,它复制从音视频源读取数据。

数据源在java层调用setDataSource方法之后,传递到NuPlayer都会打包成不同的NuPlayer::Source

```
void NuPlayer::setDataSourceAsync(const sp<IMediaHTTPService> &httpService, const char *url, const KeyedVector<String8, String8> *headers) {
    sp<AMessage> msg = new AMessage(kWhatSetDataSource, this);
    size_t len = strlen(url);

    sp<AMessage> notify = new AMessage(kWhatSourceNotify, this);

    sp<Source> source;
    if (IsHTTPLiveURL(url)) {
        source = new HTTPLiveSource(notify, httpService, url, headers);
    } else if (!strncasecmp(url, "rtsp://", 7)) {
        source = new RTSPSource(notify, httpService, url, headers, mUIDValid, mUID);
    } else if ((!strncasecmp(url, "http://", 7) || !strncasecmp(url, "https://", 8)) && ((len >= 4 && !strcasecmp(".sdp", &url[len - 4])) || strstr(url, ".sdp?"))) {
        source = new RTSPSource(notify, httpService, url, headers, mUIDValid, mUID, true);
    } else {
        sp<GenericSource> genericSource = new GenericSource(notify, mUIDValid, mUID);
        status_t err = genericSource->setDataSource(httpService, url, headers);

        if (err == OK) {
            source = genericSource;
        } else {
            ALOGE("Failed to set data source!");
        }
    }
    msg->setObject("source", source);
    msg->post();
}

void NuPlayer::setDataSourceAsync(int fd, int64_t offset, int64_t length) {
    sp<AMessage> msg = new AMessage(kWhatSetDataSource, this);

    sp<AMessage> notify = new AMessage(kWhatSourceNotify, this);

    sp<GenericSource> source = new GenericSource(notify, mUIDValid, mUID);

    status_t err = source->setDataSource(fd, offset, length);

    if (err != OK) {
        ALOGE("Failed to set data source!");
        source = NULL;
    }

    msg->setObject("source", source);
    msg->post();
}

void NuPlayer::setDataSourceAsync(const sp<DataSource> &dataSource) {
    sp<AMessage> msg = new AMessage(kWhatSetDataSource, this);
    sp<AMessage> notify = new AMessage(kWhatSourceNotify, this);

    sp<GenericSource> source = new GenericSource(notify, mUIDValid, mUID);
    status_t err = source->setDataSource(dataSource);

    if (err != OK) {
        ALOGE("Failed to set data source!");
        source = NULL;
    }

    msg->setObject("source", source);
    msg->post();
}

void NuPlayer::setDataSourceAsync(const sp<IStreamSource> &source) {
    sp<AMessage> msg = new AMessage(kWhatSetDataSource, this);

    sp<AMessage> notify = new AMessage(kWhatSourceNotify, this);

    msg->setObject("source", new StreamingSource(notify, source));
    msg->post();
}
```

上面的HTTPLiveSource、RTSPSource、GenericSource、StreamingSource都继承NuPlayer::Source

```
struct NuPlayer::HTTPLiveSource : public NuPlayer::Source {
	...
}

struct NuPlayer::RTSPSource : public NuPlayer::Source {
	...
}

struct NuPlayer::GenericSource : public NuPlayer::Source {
	...
}

struct NuPlayer::StreamingSource : public NuPlayer::Source {
	...
}
```

{% img /安卓音视频播放-NuPlayer/1.png %}

可以看到NuPlayer根据音视频源的类型,创建了不同的NuPlayer::Source,然后放到了一个what=kWhatSetDataSource的AMessage中post了出去。

让我们跟踪下NuPlayer这个Handler是怎么处理kWhatSetDataSource消息的:

```
void NuPlayer::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        case kWhatSetDataSource:
        {
        	...
        	sp<RefBase> obj;
        	CHECK(msg->findObject("source", &obj));
        	...
        	mSource = static_cast<Source *>(obj.get());
        	...
        }
    ...
}
```
其实就是赋值了一下mSource

## NuPlayer::Source准备数据

首先在调用prepare方法之后NuPlayer会发送kWhatPrepare消息,在NuPlayer::onMessageReceived里面会调用NuPlayer::Source::prepareAsync方法:

```
void NuPlayer::prepareAsync() {
    (new AMessage(kWhatPrepare, this))->post();
}

void NuPlayer::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        ...
        case kWhatPrepare:
        {
            mSource->prepareAsync();
            break;
        }
        ...
    }
}

```

NuPlayer::Source::prepareAsync是让NuPlayer::Source去准备好数据源,例如通过网络请求获取或者打开文件获取。

NuPlayer::Source内部基本也是通过Handler机制异步去加载数据的,这里只举一个NuPlayer::GenericSource的例子:


```
void NuPlayer::GenericSource::prepareAsync() {
    if (mLooper == NULL) {
        mLooper = new ALooper;
        mLooper->setName("generic");
        mLooper->start();

        mLooper->registerHandler(this);
    }

    sp<AMessage> msg = new AMessage(kWhatPrepareAsync, this);
    msg->post();
}

void NuPlayer::GenericSource::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
      case kWhatPrepareAsync:
      {
          onPrepareAsync();
          break;
      }
      ....
}

void NuPlayer::GenericSource::onPrepareAsync() {
	...
	if (!mUri.empty()) {
		const char* uri = mUri.c_str();
		...
		mDataSource = DataSource::CreateFromURI(
                   mHTTPService, uri, &mUriHeaders, &contentType,
                   static_cast<HTTPBase *>(mHttpSource.get()));
	} else {
		mIsWidevine = false;
		mDataSource = new FileSource(mFd, mOffset, mLength);
		mFd = -1;
	}
	...
	finishPrepareAsync();
}
```

准备好数据之后会调用finishPrepareAsync用构造的时候传给NuPlayer::Source的kWhatSourceNotify消息复制出一个新的kWhatPrepared消息反向通知NuPlayer,这是一种标准的原型模式:

```

void NuPlayer::GenericSource::finishPrepareAsync() {
	...
	notifyPrepared();
	...
}

void NuPlayer::Source::notifyPrepared(status_t err) {
    sp<AMessage> notify = dupNotify();
    notify->setInt32("what", kWhatPrepared);
    notify->setInt32("err", err);
    notify->post();
}

struct NuPlayer::Source : public AHandler {
	...
	Source(const sp<AMessage> &notify)
       : mNotify(notify) {
	}
	...
	sp<AMessage> dupNotify() const { return mNotify->dup(); }
	...
}
```

然后NuPlayer::Source其实也充当了Demux的功能,它生命了一个dequeueAccessUnit纯虚方法,这个方法就是从数据源分离获取音频或者视频数据:

```
virtual status_t dequeueAccessUnit( bool audio, sp<ABuffer> *accessUnit) = 0;
```

{% img /安卓音视频播放-NuPlayer/2.png %}

# NuPlayer::Decoder

NuPlayer::Decoder是NuPlayer的解码模块,然我们来看看它是怎么创建的吧。

```
void NuPlayer::start() {
    (new AMessage(kWhatStart, this))->post();
}

void NuPlayer::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        ...
        case kWhatStart:
        {
            ...
            onStart();
            ...
        }
        ...
    }
}

//startPositionUs 有个默认值-1
void NuPlayer::onStart(int64_t startPositionUs) {
	...
	sp<AMessage> notify = new AMessage(kWhatRendererNotify, this);
	...
    mRenderer = new Renderer(mAudioSink, notify, flags);
    ...
    postScanSources();
}

```

在onStart里面创建了Renderer,然后调用postScanSources

```

void NuPlayer::postScanSources() {
    if (mScanSourcesPending) {
        return;
    }

    sp<AMessage> msg = new AMessage(kWhatScanSources, this);
    msg->setInt32("generation", mScanSourcesGeneration);
    msg->post();

    mScanSourcesPending = true;
}

void NuPlayer::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        ...
        case kWhatScanSources:
        {
            ...
            if (mSurface != NULL) {
                if (instantiateDecoder(false, &mVideoDecoder) == -EWOULDBLOCK) {
                    rescan = true;
                }
            }

            if (mAudioSink != NULL && mAudioDecoder == NULL) {
                if (instantiateDecoder(true, &mAudioDecoder) == -EWOULDBLOCK) {
                    rescan = true;
                }
            }
            ...
        }
        ...
    }
}

status_t NuPlayer::instantiateDecoder(bool audio, sp<DecoderBase> *decoder, bool checkAudioModeChange) {
	...
	sp<AMessage> format = mSource->getFormat(audio);
	...
	if (audio) {
		...
		*decoder = new Decoder(notify, mSource, mPID, mRenderer);
		...
	} else {
		...
		*decoder = new Decoder(notify, mSource, mPID, mRenderer, mSurface, mCCDecoder);
		...
	}
	(*decoder)->init();
    (*decoder)->configure(format);
	...
}

void NuPlayer::Decoder::onConfigure(const sp<AMessage> &format) {
	...
	AString mime;
    CHECK(format->findString("mime", &mime));
    ...
	mCodec = MediaCodec::CreateByType(
            mCodecLooper, mime.c_str(), false /* encoder */, NULL /* err */, mPid);
	...
}
```

从这里可以看到NuPlayer::Decoder实际上是通过MediaCodec去进行音视频的解码的。

MediaCodec是安卓提供的,访问底层编解码器的接口。其实最后也是依赖OpenMax的。

# NuPlayer::Render

NuPlayer::Render顾名思义是做渲染的,但是经过代码分析,其实它的逻辑只是做音视频同步,然后音频渲染会交给MediaPlayerBase::AudioSink,而视频渲染会交回给NuPlayer::Decoder再交给MediaCodec。


当NuPlayer::Decoder从NuPlayer::Source拿到数据并解码之后,会调用NuPlayer::Renderer::queueBuffer方法将解码之后的数据丢给NuPlayer::Renderer

```
bool NuPlayer::Decoder::handleAnOutputBuffer(
        size_t index,
        size_t offset,
        size_t size,
        int64_t timeUs,
        int32_t flags) {
	sp<ABuffer> buffer;
	mCodec->getOutputBuffer(index, &buffer); // 从MediaCodec获取解码之后的数据
	...
	// 注意这里,这个reply用于让mRenderer回调NuPlayer::Decoder视频绘制.
	// 不过很多人都会忽略这行注释吧,没关系,读到后面你们还会返回来看的...
	sp<AMessage> reply = new AMessage(kWhatRenderBuffer, this); 
	...
	mRenderer->queueBuffer(mIsAudio, buffer, reply);
	...
}
```

NuPlayer::Renderer会往消息队列丢入kWhatQueueBuffer消息:

```
void NuPlayer::Renderer::queueBuffer(
        bool audio,
        const sp<ABuffer> &buffer,
        const sp<AMessage> &notifyConsumed) {
    sp<AMessage> msg = new AMessage(kWhatQueueBuffer, this);
    msg->setInt32("queueGeneration", getQueueGeneration(audio));
    msg->setInt32("audio", static_cast<int32_t>(audio));
    msg->setBuffer("buffer", buffer);
    msg->setMessage("notifyConsumed", notifyConsumed);
    msg->post();
}

void NuPlayer::Renderer::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        case kWhatQueueBuffer:
        {
            onQueueBuffer(msg);
            break;
        }
        ...
}

void NuPlayer::Renderer::onQueueBuffer(const sp<AMessage> &msg) {
    int32_t audio;
    CHECK(msg->findInt32("audio", &audio));
    ...
    sp<ABuffer> buffer;
    CHECK(msg->findBuffer("buffer", &buffer));
    
    sp<AMessage> notifyConsumed;
    CHECK(msg->findMessage("notifyConsumed", &notifyConsumed));

    QueueEntry entry;
    entry.mBuffer = buffer;
    entry.mNotifyConsumed = notifyConsumed;
    
    ...
    if (audio) {
        Mutex::Autolock autoLock(mLock);
        mAudioQueue.push_back(entry);
        postDrainAudioQueue_l();
    } else {
        mVideoQueue.push_back(entry);
        postDrainVideoQueue();
    }
    ...
}
```

这里会判断是从AudioDecoder传来的音频数据,还是从VideoDecoder传来的视频数据。音频数据会丢到mAudioQueue而视频数据会丢到mVideoQueue,然后调用postDrainAudioQueue_l或者postDrainVideoQueue通过Handler机制发送音频处理消息或者视频处理消息

## 音频处理

让我们先看看音频部分的处理

```
void NuPlayer::Renderer::postDrainAudioQueue_l(int64_t delayUs) {
	...
	sp<AMessage> msg = new AMessage(kWhatDrainAudioQueue, this);
    msg->setInt32("drainGeneration", mAudioDrainGeneration);
    msg->post(delayUs);
}

void NuPlayer::Renderer::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        case kWhatDrainAudioQueue:
        {
            if (onDrainAudioQueue()) {
            	...
            }
            ...
        }
        ...
}

bool NuPlayer::Renderer::onDrainAudioQueue() {
	...
	while (!mAudioQueue.empty()) {
        QueueEntry *entry = &*mAudioQueue.begin();
        ...
        ssize_t written = mAudioSink->write(entry->mBuffer->data() + entry->mOffset,
                                            copy, false /* blocking */);
        ...
   }
   ...
}
```

可以看到这里NuPlayer::Renderer会从mAudioQueue拿音频数据然后写入mAudioSink。mAudioSink内部就会调用声音输出设备如喇叭等去播放了。

## 视频处理

接着看看视频处理

```
void NuPlayer::Renderer::postDrainVideoQueue() {
	sp<AMessage> msg = new AMessage(kWhatDrainVideoQueue, this);
	...
	msg->post(postDelayUs);
	...
}

void NuPlayer::Renderer::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        case kWhatDrainVideoQueue:
        {
            ...
            onDrainVideoQueue();
            ...
        }
        ...
}

void NuPlayer::Renderer::onDrainVideoQueue() {
	...
	QueueEntry *entry = &*mVideoQueue.begin();
	...
	
    entry->mNotifyConsumed->setInt64("timestampNs", realTimeUs * 1000ll);
    entry->mNotifyConsumed->setInt32("render", !tooLate);
    entry->mNotifyConsumed->post();
	...
}
```

视频处理这里其实还会做一些数值计算,主要用于视频的平滑播放,这里就忽略了。然后就调用了mNotifyConsumed的post方法。这个mNotifyConsumed是啥呢?大家可以往上拉回到NuPlayer::Decoder::handleAnOutputBuffer给NuPlayer::Renderer丢入解码后的音视频数据那里,估计很多人都没有注意到。

总之,它会给NuPlayer::Decoder发一个kWhatRenderBuffer消息,然后就会让NuPlayer::Decoder去渲染视频画面了,不过它也是交给MediaCodec去渲染而已:

```
void NuPlayer::Decoder::onMessageReceived(const sp<AMessage> &msg) {
    ALOGV("[%s] onMessage: %s", mComponentName.c_str(), msg->debugString().c_str());

    switch (msg->what()) {
        case kWhatRenderBuffer:
        {
            if (!isStaleReply(msg)) {
                onRenderBuffer(msg);
            }
            break;
        }
        ...
}

void NuPlayer::Decoder::onRenderBuffer(const sp<AMessage> &msg) {
	...
   err = mCodec->renderOutputBufferAndRelease(bufferIx, timestampNs);
   ...
}
```

这个mCodec大家可能都忘了是啥,其实在NuPlayer::Decoder那节有说过的,就是MediaCodec:

```
void NuPlayer::Decoder::onConfigure(const sp<AMessage> &format) {
	...
	AString mime;
    CHECK(format->findString("mime", &mime));
	...
	mCodec = MediaCodec::CreateByType(
            mCodecLooper, mime.c_str(), false /* encoder */, NULL /* err */, mPid);
	...
}
```

# 整体架构图

所以NuPlayer的整个架构图如下:

{% img /安卓音视频播放-NuPlayer/3.png %}

可以看出来, NuPlayer的核心功能是依赖MediaCodec去实现的
