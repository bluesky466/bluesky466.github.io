title: 安卓音视频播放 - AwesomePlayer
date: 2019-01-17 23:31:12
tags:
  - 技术相关
  - Android
---
# 音视频播放基础流程

在讲具体的实现之前我们看一下音视频播放的基础流程:

{% img /安卓音视频播放-AwesomePlayer/1.png %}

流程很简单,就是将复用的音视频流解复用出编码后的音频流和编码后的视频流。然后通过音频解码解出PCM数据给音频设备去播放,通过视频解码解出YUV数据给视频设备去播放。

# StagefrightPlayer

[上一篇文章](http://blog.islinjw.cn/2019/01/17/%E5%AE%89%E5%8D%93%E9%9F%B3%E8%A7%86%E9%A2%91%E6%92%AD%E6%94%BE%E6%9E%B6%E6%9E%84-%E4%B8%80/)有讲到MediaPlayerService会通过MediaPlayerFactory创建Player,其中一个创建的就是StagefrightPlayer.但它实际上是一个空壳,只是简单的调用AwesomePlayer的实现而已:


```
//StagefrightPlayer.h
class StagefrightPlayer : public MediaPlayerInterface {
	...
private:
	AwesomePlayer *mPlayer;
	...
}

//StagefrightPlayer.cpp
status_t StagefrightPlayer::pause() {
    ALOGV("pause");

    return mPlayer->pause();
}

bool StagefrightPlayer::isPlaying() {
    ALOGV("isPlaying");
    return mPlayer->isPlaying();
}

status_t StagefrightPlayer::seekTo(int msec) {
    ALOGV("seekTo %.2f secs", msec / 1E3);

    status_t err = mPlayer->seekTo((int64_t)msec * 1000);

    return err;
}
...
```


所以我们直接看AwesomePlayer的实现。

# 多线程架构

音视频的处理一般都很耗时,所以AwesomePlayer开了一个子线程去工作,防止阻塞住MediaPlayerService的主线程。

具体的架构如下(这幅图是在这篇[博客](https://www.cnblogs.com/shakin/p/4463639.html)抄来的,这篇文章写得的确不错,大家感兴趣可以去仔细读一下:

{% img /安卓音视频播放-AwesomePlayer/2.png %}

首先AwesomePlayer内部有个TimedEventQueue对象,所有的操作都会封装成一个个的Event,丢到这个队列里。然后TimedEventQueue创建了一个子线程,不断从队列中拿出Event来执行。


例如prepare操作最后会调到prepareAsync_l,这里面就是创建了个Event,通过postEvent丢到队列里:

```
status_t AwesomePlayer::prepareAsync_l() {
    ...

    if (!mQueueStarted) {
        mQueue.start();
        mQueueStarted = true;
    }

	...
    mAsyncPrepareEvent = new AwesomeEvent(
            this, &AwesomePlayer::onPrepareAsyncEvent);

    mQueue.postEvent(mAsyncPrepareEvent);

    return OK;
}

```

AwesomeEvent继承TimedEventQueue::Event,实现了fire方法,回调了注册的方法:

```
struct AwesomeEvent : public TimedEventQueue::Event {
    AwesomeEvent(
            AwesomePlayer *player,
            void (AwesomePlayer::*method)())
        : mPlayer(player),
          mMethod(method) {
    }
    ...
    virtual void fire(TimedEventQueue *queue, int64_t /* now_us */) {
        (mPlayer->*mMethod)();
    }
    ...
};
```

TimedEventQueue::start创建了一个子线程,调用TimedEventQueue::threadEntry方法,这里面有个死循环一直在从Event队列中拿出Event,执行fire方法:

```
void TimedEventQueue::start() {
    if (mRunning) {
        return;
    }

    mStopped = false;

    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE);

    pthread_create(&mThread, &attr, ThreadWrapper, this);

    pthread_attr_destroy(&attr);

    mRunning = true;
}

void *TimedEventQueue::ThreadWrapper(void *me) {

    androidSetThreadPriority(0, ANDROID_PRIORITY_FOREGROUND);

    static_cast<TimedEventQueue *>(me)->threadEntry();

    return NULL;
}

void TimedEventQueue::threadEntry() {
    ...
    for (;;) {
        ...
        event = removeEventFromQueue_l(eventID);

        if (event != NULL) {
            // Fire event with the lock NOT held.
            event->fire(this, now_us);
        }
    }
}
```


# Demux

我们先来看看prepare回调的时候实际是调用了AwesomePlayer::beginPrepareAsync\_l()方法,在这里会实际的去设置数据源,然后初始化Demux、视频解码器和音频解码器:

```
void AwesomePlayer::onPrepareAsyncEvent() {
    Mutex::Autolock autoLock(mLock);
    beginPrepareAsync_l();
}


void AwesomePlayer::beginPrepareAsync_l() {
    ...
    status_t err = finishSetDataSource_l();
    ...
    status_t err = initVideoDecoder();
    ...
    status_t err = initAudioDecoder();
}
```


先来看看AwesomePlayer::finishSetDataSource_l实际上是为音视频源找到对应的MediaExtractor,这个MediaExtractor的功能就是实现播放器的基础流程中的Demux,分解出视频流和音频流:

{% img /安卓音视频播放-AwesomePlayer/3.png %}

代码如下:

```

status_t AwesomePlayer::finishSetDataSource_l() {
    ...
    extractor = MediaExtractor::Create(dataSource, sniffedMIME.empty() ? NULL : sniffedMIME.c_str());
    ...
    status_t err = setDataSource_l(extractor);
    ...
}


status_t AwesomePlayer::setDataSource_l(const sp<MediaExtractor> &extractor) {
    ...
    for (size_t i = 0; i < extractor->countTracks(); ++i) {
        sp<MetaData> meta = extractor->getTrackMetaData(i);

        const char *_mime;
        CHECK(meta->findCString(kKeyMIMEType, &_mime));

        String8 mime = String8(_mime);
        ...
        if (!haveVideo && !strncasecmp(mime.string(), "video/", 6)) {
            setVideoSource(extractor->getTrack(i));
            ...
        } else if (!haveAudio && !strncasecmp(mime.string(), "audio/", 6)) {
            setAudioSource(extractor->getTrack(i));
            ...
        }
        ...
    }
    ...
}
```


MediaExtractor::Create的实现也是蛮粗暴的,判断媒体类型,然后创建不同的MediaExtractor,如MPEG4Extractor、MP3Extractor等:

```

sp<MediaExtractor> MediaExtractor::Create(const sp<DataSource> &source, const char *mime) {
	..
	MediaExtractor *ret = NULL;
	    if (!strcasecmp(mime, MEDIA_MIMETYPE_CONTAINER_MPEG4)
	            || !strcasecmp(mime, "audio/mp4")) {
	        ret = new MPEG4Extractor(source);
	    } else if (!strcasecmp(mime, MEDIA_MIMETYPE_AUDIO_MPEG)) {
	        ret = new MP3Extractor(source, meta);
	    } else if (!strcasecmp(mime, MEDIA_MIMETYPE_AUDIO_AMR_NB)
	            || !strcasecmp(mime, MEDIA_MIMETYPE_AUDIO_AMR_WB)) {
	        ret = new AMRExtractor(source);
	    } else if (!strcasecmp(mime, MEDIA_MIMETYPE_AUDIO_FLAC)) {
	        ret = new FLACExtractor(source);
	    } else if (!strcasecmp(mime, MEDIA_MIMETYPE_CONTAINER_WAV)) {
	        ret = new WAVExtractor(source);
	    } else if (!strcasecmp(mime, MEDIA_MIMETYPE_CONTAINER_OGG)) {
	        ret = new OggExtractor(source);
	    } else if (!strcasecmp(mime, MEDIA_MIMETYPE_CONTAINER_MATROSKA)) {
	        ret = new MatroskaExtractor(source);
	    } else if (!strcasecmp(mime, MEDIA_MIMETYPE_CONTAINER_MPEG2TS)) {
	        ret = new MPEG2TSExtractor(source);
	    } else if (!strcasecmp(mime, MEDIA_MIMETYPE_CONTAINER_WVM)) {
	        // Return now.  WVExtractor should not have the DrmFlag set in the block below.
	        return new WVMExtractor(source);
	    } else if (!strcasecmp(mime, MEDIA_MIMETYPE_AUDIO_AAC_ADTS)) {
	        ret = new AACExtractor(source, meta);
	    } else if (!strcasecmp(mime, MEDIA_MIMETYPE_CONTAINER_MPEG2PS)) {
	        ret = new MPEG2PSExtractor(source);
	    }
	...
}
```

{% img /安卓音视频播放-AwesomePlayer/4.png %}

# 解码器

然后AwesomePlayer::initVideoDecoder、AwesomePlayer::initAudioDecoder里面就是调用OMXCodec去做解码,OMXCodec其实是OpenMax的一层封装。OpenMax就是具体的解码器实现了:

```

status_t AwesomePlayer::initVideoDecoder(uint32_t flags) {
	...
	mVideoSource = OMXCodec::Create(
            mClient.interface(), mVideoTrack->getFormat(),
            false, // createEncoder
            mVideoTrack,
            NULL, flags, USE_SURFACE_ALLOC ? mNativeWindow : NULL);
   ...
}


status_t AwesomePlayer::initAudioDecoder() {
	...
	mOmxSource = OMXCodec::Create(
                mClient.interface(), mAudioTrack->getFormat(),
                false, // createEncoder
                mAudioTrack);
    ...
}
```

# 播放流程

应用在java层调用MediaPlayer.start,最终会通过IPC去到MediaPlayerService里调用到StagefrightPlayer::start方法,我们直接从这里开始往下挖:


```
//从这里开始是StagefrightPlayer.cpp里的代码
status_t StagefrightPlayer::start() {
    return mPlayer->play();
}

//从这里开始是AwesomePlayer.cpp里的代码
status_t AwesomePlayer::play() {
	...
    return play_l();
}

status_t AwesomePlayer::play_l() {
    ...
    createAudioPlayer_l();
    ...
    postVideoEvent_l();
    ...
    return OK;
}

void AwesomePlayer::postVideoEvent_l(int64_t delayUs) {
    ...
    mQueue.postEventWithDelay(mVideoEvent, delayUs < 0 ? 10000 : delayUs);
}
```

在AwesomePlayer::play\_l方法里面调用AwesomePlayer::createAudioPlayer\_l创建了一个AudioPlayer,然后调用AwesomePlayer::postVideoEvent\_l往mQueue里丢了一个事件。

还记得这个mVideoEvent吗?它对应的是AwesomePlayer::onVideoEvent方法,也就是说把这个Event丢到mQueue里面之后AwesomePlayer::onVideoEvent就会在子线程中被调用

```
mVideoEvent = new AwesomeEvent(this, &AwesomePlayer::onVideoEvent);
```

让我们继续看看AwesomePlayer::onVideoEvent方法里面干了什么:

```
void AwesomePlayer::onVideoEvent() {
	...
	status_t err = mVideoSource->read(&mVideoBuffer, &options);
	...
	if ((mNativeWindow != NULL)
            && (mVideoRendererIsPreview || mVideoRenderer == NULL)) {
        mVideoRendererIsPreview = false;

        initRenderer_l();
    }
    ...
    if (mAudioPlayer != NULL && !(mFlags & (AUDIO_RUNNING | SEEK_PREVIEW))) {
        startAudioPlayer_l();
    }
    ...
    if (mVideoRenderer != NULL) {
        ...
        mVideoRenderer->render(mVideoBuffer);
        ...
        }
    ...
	postVideoEvent_l();
}
```

这个方法最重要的就是创建一个VideoRender,从mVideoSource读取解码好的视频帧去渲染,渲染完之后再调AwesomePlayer::postVideoEvent\_l再往队列丢入一个VideoEvent。于是画面就不断的刷新了。

可以看到,这个方法内部也启动了音频播放器去播放音频。而且其实它还做了一些音视频同步的工作,但是考虑到逻辑比较啰嗦,我这里就省略了。


# VideoRender

最后让我们来看看VideoRendere是怎么来的

```
void AwesomePlayer::initRenderer_l() {
	...
	if (USE_SURFACE_ALLOC
	        && !strncmp(component, "OMX.", 4)
	        && strncmp(component, "OMX.google.", 11)
	        && strcmp(component, "OMX.Nvidia.mpeg2v.decode")) {
	    mVideoRenderer =
	        new AwesomeNativeWindowRenderer(mNativeWindow, rotationDegrees);
	} else {
	    mVideoRenderer = new AwesomeLocalRenderer(mNativeWindow, meta);
	}
}
```

可以看到,是根据解码器类型用mNativeWindow创建了不同的AwesomeNativeWindowRenderer或者AwesomeLocalRenderer。这个mNativeWindow就是画面最终需要渲染到的地方

我们看看mNativeWindow是怎么来的:

```
// AwesomePlayer.cpp
status_t AwesomePlayer::setNativeWindow_l(const sp<ANativeWindow> &native) {
    mNativeWindow = native;
    ...
}

status_t AwesomePlayer::setSurfaceTexture(const sp<IGraphicBufferProducer> &bufferProducer) {
   ...
   err = setNativeWindow_l(new Surface(bufferProducer));
   ...
}

//StagefrightPlayer.cpp
status_t StagefrightPlayer::setVideoSurfaceTexture(
        const sp<IGraphicBufferProducer> &bufferProducer) {
    ALOGV("setVideoSurfaceTexture");

    return mPlayer->setSurfaceTexture(bufferProducer);
}

//MediaPlayerService.cpp
status_t MediaPlayerService::Client::setVideoSurfaceTexture(
	...
	sp<MediaPlayerBase> p = getPlayer();
	...
	status_t err = p->setVideoSurfaceTexture(bufferProducer);
	...
}

//MediaPlayer.cpp
status_t MediaPlayer::setVideoSurfaceTexture(
        const sp<IGraphicBufferProducer>& bufferProducer)
{
    ...
    return mPlayer->setVideoSurfaceTexture(bufferProducer);
}


//android_media_MediaPlayer.cpp
static void setVideoSurface(JNIEnv *env, jobject thiz, jobject jsurface, jboolean mediaPlayerMustBeAlive)
{
    sp<MediaPlayer> mp = getMediaPlayer(env, thiz);
    ...
    sp<Surface> surface(android_view_Surface_getSurface(env, jsurface));
    ...
    new_st = surface->getIGraphicBufferProducer();
    ...
    mp->setVideoSurfaceTexture(new_st);
}

static void android_media_MediaPlayer_setVideoSurface(JNIEnv *env, jobject thiz, jobject jsurface)
{
    setVideoSurface(env, thiz, jsurface, true /* mediaPlayerMustBeAlive */);
}

//android.media.MediaPlayer.java
public class MediaPlayer extends PlayerBase
                         implements SubtitleController.Listener
                                  , VolumeAutomation
                                  , AudioRouting
{
	...
	private native void _setVideoSurface(Surface surface);
	...
	public void setDisplay(SurfaceHolder sh) {
        mSurfaceHolder = sh;
        Surface surface;
        if (sh != null) {
            surface = sh.getSurface();
        } else {
            surface = null;
        }
        _setVideoSurface(surface);
        updateSurfaceScreenOn();
    }
    ...
}
```

可以看到,VideoRendere最终是根据MediaPlayer.setDisplay这个方法设置的SurfaceHolder创建的到的。这就解释了画面是怎么渲染到指定的SurfaceView上的。

# 完整架构图


整个渲染的架构如下:

{% img /安卓音视频播放-AwesomePlayer/5.png %}


