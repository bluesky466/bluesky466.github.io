title: TextureView清除摄像头最后一帧画面的原理探究
date: 2023-06-08 08:41:19
tags:
    - 技术相关
    - Android
---

最近协助一个摄像头相关的项目遇到了一个有意思的问题，这里记录一下。

原问题大概是使用TextureView预览摄像头,关闭摄像头之后画面会残留最后一帧,需要把他清除。我一开始使用的方式是获取Surface的Canvas去将整个画布画上黑色来实现清除画面:

```java
Canvas canvas = mPreviewSurface.lockCanvas(null);
canvas.drawColor(Color.BLACK);
mPreviewSurface.unlockCanvasAndPost(canvas);
```

但是遇到了下面的问题:

1. CameraDevice.close之前lockCanvas会抛出IllegalArgumentException
2. 在CameraDevice.close之后lockCanvas虽然可以清除画面,但是再次打开调用CameraDevice.createCaptureSession会失败,回调onConfigureFailed

在网上搜索了下[stackoverflow](https://stackoverflow.com/questions/24902114/mediaplayer-cannot-render-to-textureview-after-image-render/24914966#24914966)上fadden大神是这么解释的:

```
You can't do this, due to a limitation of the Android app framework (as of Android 4.4 at least).

The SurfaceTexture that underlies the TextureView is a buffer consumer. The MediaPlayer is one example of a buffer producer, Canvas is another. Once you attach a producer, you have to detach it before you can attach a second producer.

The trouble is that there is no way to detach a software-based (Canvas) buffer producer. There could be, but isn't. So once you draw with Canvas, you're stuck. (There's a note to that effect here.)

You can detach a GLES producer. For example, in one of Grafika's video player classes you can find a clearSurface() method that clears the surface to black using GLES. Note the EGL context and window are created and explicitly released within the scope of the method. You could expand the method to show an image instead.
```

大概意思就是TextureView作为一个画面的消费者,可以绑定到不同的画面生产者(Canvas是其中一种,另外像MediaPlayer、Camera这些也可以作为画面生产者)。一旦连接上一个生产者之后就不能再次连接其他的生产者了,而Canvas这个生产者比较野蛮,并没有提供解除绑定的方法。所以一旦TextureView绑定到Canvas之后，MediaPlayer、Camera就不能再使用这个Surface区显示画面了。

然后他提供的解决方法是参考[Grafika](https://github.com/google/grafika/blob/b1df331e89cffeab621f02b102d4c2c25eb6088a/app/src/main/java/com/android/grafika/PlayMovieSurfaceActivity.java#L255)使用OpenGL去做清除。


# 消费者生产者模型

消费者生产者模型在安卓的图像系统里面还是比较重要的一个东西,从[官方文档](https://source.android.com/docs/core/graphics?hl=zh-cn)的介绍里面我们可以大概看出整个工作流程:

{% img /TextureView清除摄像头最后一帧画面的原理探究/bufferqueue.png %}

- Producer 如Camera、视频解码器、OpenGL ES、Canvas等调用dequeue从BufferQueue里面获取一个空白Buffer,然后使用Buffer做绘制,绘制完成之后调用queue把Buffer交还给BufferQueue。
- Consumer 如SurfaceFlinger调用acquire从BufferQueue里面获取一个绘制好的Buffer,然后进行画面的渲染,渲染完成之后调用release把Buffer交还给BufferQueue作为空白Buffer。


```java
Canvas canvas = mPreviewSurface.lockCanvas(null);
canvas.drawColor(Color.BLACK);
mPreviewSurface.unlockCanvasAndPost(canvas);
```

用上面的lockCanvas来举例。在代码中Producer具体为IGraphicBufferProducer接口,在Surface构造的时候传入,在connect的的时候去连接:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/libs/gui/Surface.cpp
Surface::Surface(const sp<IGraphicBufferProducer>& bufferProducer, bool controlledByApp,
                 const sp<IBinder>& surfaceControlHandle)
      : mGraphicBufferProducer(bufferProducer),
      ...

int Surface::connect(
        int api, const sp<IProducerListener>& listener, bool reportBufferRemoval) {
    ...
    int err = mGraphicBufferProducer->connect(listener, api, mProducerControlledByApp, &output);
    ...
}
```

然后Surface.lockCanvas调用到native层的nativeLockCanvas去用Surface::lock来dequeueBuffer获取Buffer提供给Canvas绘制:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/base/core/jni/android_view_Surface.cpp
static jlong nativeLockCanvas(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject canvasObj, jobject dirtyRectObj) {
    ANativeWindow_Buffer buffer;
    status_t err = surface->lock(&buffer, dirtyRectPtr);
    ...
    graphics::Canvas canvas(env, canvasObj);
    canvas.setBuffer(&buffer, static_cast<int32_t>(surface->getBuffersDataSpace()));
    ...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/libs/gui/Surface.cpp
status_t Surface::lock(
        ANativeWindow_Buffer* outBuffer, ARect* inOutDirtyBounds)
{
    ...
    status_t err = dequeueBuffer(&out, &fenceFd);
    ...
}

int Surface::dequeueBuffer(android_native_buffer_t** buffer, int* fenceFd) {
    ...
    status_t result = mGraphicBufferProducer->dequeueBuffer(&buf, &fence, dqInput.width,
                                                            dqInput.height, dqInput.format,
                                                            dqInput.usage, &mBufferAge,
                                                            dqInput.getTimestamps ?
                                                            &frameTimestamps : nullptr);
    ...
}
```
而Surface.unlockCanvasAndPost会调用native层的nativeUnlockCanvasAndPost去调用Surface::unlockAndPost去queueBuffer:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/base/core/jni/android_view_Surface.cpp
static void nativeUnlockCanvasAndPost(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject canvasObj) {
    ...
    // detach the canvas from the surface
    graphics::Canvas canvas(env, canvasObj);
    canvas.setBuffer(nullptr, ADATASPACE_UNKNOWN);

    // unlock surface
    status_t err = surface->unlockAndPost();
    ...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/libs/gui/Surface.cpp
status_t Surface::unlockAndPost()
{
    ...
    err = queueBuffer(mLockedBuffer.get(), fd);
    ...
}

int Surface::queueBuffer(android_native_buffer_t* buffer, int fenceFd) {
    ...
    status_t err = mGraphicBufferProducer->queueBuffer(i, input, &output);
    ...
}
```

这样不断循环lockCanvas、绘制Canvas、unlockCanvasAndPost就能往SurfaceFlinger这个Consumer不断提供画面去渲染了。

虽然大概的原因和解决方法都讲清楚了,但是我还是有三点疑问:

1. 需要在CameraDevice.close之后才能lockCanvas是不是意味着CameraDevice.close里面会做解绑
2. 调用unlockCanvasAndPost为什么没有解除Canvas这个内容生产者的绑定?
3. GLES可以解除绑定,那它又是怎么解除的呢?

# CameraDevice.close之后才能lockCanvas

网上搜索没有找到答案,那就只能自己分析源码了,首先我们从unlockCanvasAndPost之后再次createCaptureSession会失败的日志入手看看能不能找到什么有用的信息:

```
06-06 18:55:13.130 28137 25285 E BufferQueueProducer: [SurfaceTexture-0-28137-0](id:6de900000001,api:2,p:28137,c:28137) connect: already connected (cur=2 req=4)
06-06 18:55:13.130  1905  8873 E Camera3-OutputStream: configureConsumerQueueLocked: Unable to connect to native window for stream 0
06-06 18:55:13.130  1905  8873 E Camera3-Stream: finishConfiguration: Unable to configure stream 0 queue: Invalid argument (-22)
06-06 18:55:13.130  1905  8873 E Camera3-Device: Camera 0: configureStreamsLocked: Can't finish configuring output stream 0: Invalid argument (
-22)
06-06 18:55:13.130  1047  1365 E minksocket: MinkIPC_QRTR_Service: client with node 1 port 6838 went down
06-06 18:55:13.130  1905  8873 D CameraService: CameraPerf: setpriority success, tid is 8873, priority is 0
06-06 18:55:13.130  1905  8873 E CameraDeviceClient: endConfigure: Camera 0: Unsupported set of inputs/outputs provided
```

从日志里面可以看到在Camera3OutputStream::configureConsumerQueueLocked里面会去调用Surface::connect: 

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/av/services/camera/libcameraservice/device3/Camera3OutputStream.h
sp<Surface> mConsumer;

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/av/services/camera/libcameraservice/device3/Camera3OutputStream.cpp
status_t Camera3OutputStream::configureConsumerQueueLocked(bool allowPreviewRespace) {
    ...
    // Configure consumer-side ANativeWindow interface. The listener may be used
    // to notify buffer manager (if it is used) of the returned buffers.
    res = mConsumer->connect(NATIVE_WINDOW_API_CAMERA,
            /*reportBufferRemoval*/true,
            /*listener*/mBufferProducerListener);
    if (res != OK) {
        ALOGE("%s: Unable to connect to native window for stream %d",
                __FUNCTION__, mId);
        return res;
    }
    ...
}
```

而在Surface::connect里面会调用BufferQueueProducer::connect:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/libs/gui/Surface.cpp
int Surface::connect(int api) {
    static sp<IProducerListener> listener = new StubProducerListener();
    return connect(api, listener);
}

int Surface::connect(int api, const sp<IProducerListener>& listener) {
    return connect(api, listener, false);
}

int Surface::connect(
        int api, const sp<IProducerListener>& listener, bool reportBufferRemoval) {
    ...
    int err = mGraphicBufferProducer->connect(listener, api, mProducerControlledByApp, &output);
    ...
}

```

在BufferQueueProducer::connect里面会判断如果mCore->mConnectedApi不为BufferQueueCore::NO\_CONNECTED\_API(即已经connect过了)就不能再connect:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/libs/gui/include/gui/BufferQueueProducer.h
sp<BufferQueueCore> mCore;

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/libs/gui/BufferQueueProducer.cpp
status_t BufferQueueProducer::connect(const sp<IProducerListener>& listener,
        int api, bool producerControlledByApp, QueueBufferOutput *output) {
    ...
    if (mCore->mConnectedApi != BufferQueueCore::NO_CONNECTED_API) {
        BQ_LOGE("connect: already connected (cur=%d req=%d)",
                mCore->mConnectedApi, api);
        return BAD_VALUE;
    }
    ...
    mCore->mConnectedApi = api;
    ...
}
```

所以我们看到的already connected日志就是从这里打印的。

```
06-06 18:55:13.130 28137 25285 E BufferQueueProducer: [SurfaceTexture-0-28137-0](id:6de900000001,api:2,p:28137,c:28137) connect: already connected (cur=2 req=4)
```
connect api的类型有下面几种,所以从日志上我们可以分析出,SurfaceTexture已经connect到NATIVE\_WINDOW\_API\_CPU了,不能再connect到NATIVE\_WINDOW\_API\_CAMERA:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/libs/nativewindow/include/system/window.h
/* parameter for NATIVE_WINDOW_[API_][DIS]CONNECT */
enum {
    /* Buffers will be queued by EGL via eglSwapBuffers after being filled using
     * OpenGL ES.
     */
    NATIVE_WINDOW_API_EGL = 1,

    /* Buffers will be queued after being filled using the CPU
     */
    NATIVE_WINDOW_API_CPU = 2,

    /* Buffers will be queued by Stagefright after being filled by a video
     * decoder.  The video decoder can either be a software or hardware decoder.
     */
    NATIVE_WINDOW_API_MEDIA = 3,

    /* Buffers will be queued by the the camera HAL.
     */
    NATIVE_WINDOW_API_CAMERA = 4,
};
```

而在CameraDevice.close里面会调用Camera3OutputStream::disconnectLocked最终会调用到BufferQueueProducer::disconnect将mCore->mConnectedApi赋值回BufferQueueCore::NO\_CONNECTED\_API:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/av/services/camera/libcameraservice/device3/Camera3OutputStream.cpp
status_t Camera3OutputStream::disconnectLocked() {
    ...
    ALOGV("%s: disconnecting stream %d from native window", __FUNCTION__, getId());

    res = native_window_api_disconnect(mConsumer.get(),
                                       NATIVE_WINDOW_API_CAMERA);
    ...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/libs/nativewindow/include/system/window.h
static inline int native_window_api_disconnect(
        struct ANativeWindow* window, int api)
{
    return window->perform(window, NATIVE_WINDOW_API_DISCONNECT, api);
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/libs/gui/Surface.cpp
int Surface::perform(int operation, va_list args)
{
    ...
    case NATIVE_WINDOW_API_DISCONNECT:
        res = dispatchDisconnect(args);
        break;
    ...
}

int Surface::dispatchDisconnect(va_list args) {
    int api = va_arg(args, int);
    return disconnect(api);
}

int Surface::disconnect(int api, IGraphicBufferProducer::DisconnectMode mode) {
    ...
    int err = mGraphicBufferProducer->disconnect(api, mode);
    ...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/libs/gui/BufferQueueProducer.cpp
status_t BufferQueueProducer::disconnect(int api, DisconnectMode mode) {
    ...
    mCore->mConnectedApi = BufferQueueCore::NO_CONNECTED_API;
    ...
}

```

所以在CameraDevice.close之后mCore->mConnectedApi被赋值成了BufferQueueCore::NO\_CONNECTED\_API,lockCanvas再去BufferQueueProducer::connect就不会失败。


# lockCanvas & unlockCanvasAndPost

Surface.lockCanvas最终会去到Surface::lock里调用Surface::connect(NATIVE\_WINDOW\_API\_CPU):

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/base/core/jni/android_view_Surface.cpp
static jlong nativeLockCanvas(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject canvasObj, jobject dirtyRectObj) {
    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));
    ...
    status_t err = surface->lock(&buffer, dirtyRectPtr);
    ...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/libs/gui/Surface.cpp
status_t Surface::lock(
        ANativeWindow_Buffer* outBuffer, ARect* inOutDirtyBounds)
{
    ...
    if (!mConnectedToCpu) {
        int err = Surface::connect(NATIVE_WINDOW_API_CPU);
        if (err) {
            return err;
        }
        // we're intending to do software rendering from this point
        setUsage(GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN);
    }
    ...
}
```

后面的流程就和Camera3OutputStream::configureConsumerQueueLocked里面调用Surface::connect类似了,最终会调用BufferQueueProducer::connect把mCore->mConnectedApi赋值成NATIVE\_WINDOW\_API\_CPU。但是稍有不同的是在Surface::connect里面会判断这个connect api,将mConnectedToCpu赋值为true:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/libs/gui/Surface.cpp
int Surface::connect(
        int api, const sp<IProducerListener>& listener, bool reportBufferRemoval) {
    int err = mGraphicBufferProducer->connect(listener, api, mProducerControlledByApp, &output);
    ...
    if (!err && api == NATIVE_WINDOW_API_CPU) {
        mConnectedToCpu = true;
        // Clear the dirty region in case we're switching from a non-CPU API
        mDirtyRegion.clear();
    }
    ...
}
```

所以之后unlockCanvasAndPost没有disconnect BufferQueueProducer也不会在再次调用Surface.lockCanvas的时候造成重复Surface::connect(NATIVE\_WINDOW\_API\_CPU)的问题:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/base/core/jni/android_view_Surface.cpp
static void nativeUnlockCanvasAndPost(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject canvasObj) {
    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));
    if (!isSurfaceValid(surface)) {
        return;
    }

    // detach the canvas from the surface
    graphics::Canvas canvas(env, canvasObj);
    canvas.setBuffer(nullptr, ADATASPACE_UNKNOWN);

    // unlock surface
    status_t err = surface->unlockAndPost();
    if (err < 0) {
        jniThrowException(env, IllegalArgumentException, NULL);
    }
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/libs/gui/Surface.cpp
status_t Surface::unlockAndPost()
{
    if (mLockedBuffer == nullptr) {
        ALOGE("Surface::unlockAndPost failed, no locked buffer");
        return INVALID_OPERATION;
    }

    int fd = -1;
    status_t err = mLockedBuffer->unlockAsync(&fd);
    ALOGE_IF(err, "failed unlocking buffer (%p)", mLockedBuffer->handle);

    err = queueBuffer(mLockedBuffer.get(), fd);
    ALOGE_IF(err, "queueBuffer (handle=%p) failed (%s)",
            mLockedBuffer->handle, strerror(-err));

    mPostedBuffer = mLockedBuffer;
    mLockedBuffer = nullptr;
    return err;
}
```

从上面的代码也可以看出来Surface.unlockCanvasAndPost只是将Canvas从Surface上分离,但是BufferQueueProducer没有disconnect,它的mCore->mConnectedApi还是NATIVE\_WINDOW\_API\_CPU。于是再次连接Camera的时候去connect NATIVE\_WINDOW\_API\_CAMERA就会失败。


NATIVE\_WINDOW\_API\_CPU的类型只有在Surface析构的时候才会去disconnect:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/libs/gui/Surface.cpp
Surface::~Surface() {
    if (mConnectedToCpu) {
        Surface::disconnect(NATIVE_WINDOW_API_CPU);
    }
}
```

# GLES disconnect

实际上GLES是靠EGL14.eglDestroySurface去调用BufferQueueProducer::disconnect的,如果没有调用,再次去连接摄像头也会失败:

```
06-06 20:13:59.940 29586 25849 E BufferQueueProducer: [SurfaceTexture-0-29586-0](id:739200000001,api:1,p:29586,c:29586) connect: already connected (cur=1 req=4)
```

这次就是NATIVE\_WINDOW\_API\_EGL已连接,请求NATIVE\_WINDOW\_API\_CAMERA连接失败了。


# 区分connect api


为什么需要区分connect api呢? 这是由于不同api的connect类型可能会有些不一样的处理逻辑,例如BufferQueueProducer::queueBuffer里就对NATIVE\_WINDOW\_API\_EGL类型做了判断:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/libs/gui/BufferQueueProducer.cpp
status_t BufferQueueProducer::queueBuffer(int slot,
        const QueueBufferInput &input, QueueBufferOutput *output) {
    ...
    // Wait without lock held
    if (connectedApi == NATIVE_WINDOW_API_EGL) {
        // Waiting here allows for two full buffers to be queued but not a
        // third. In the event that frames take varying time, this makes a
        // small trade-off in favor of latency rather than throughput.
        lastQueuedFence->waitForever("Throttling EGL Production");
    }
    ...
}
```