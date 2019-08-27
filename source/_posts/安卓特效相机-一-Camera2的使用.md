title: 安卓特效相机(一) Camera2的使用
date: 2019-08-27 23:32:52
tags:
  - 技术相关
  - Android
---

谷歌在安卓5.0的时候废弃了原来的Camera架构,推出了全新的Camera2架构。api相对之前的版本有很大的区别。

为了熟悉这个Camera2架构的使用,我写了个简单的[特效相机应用](https://github.com/bluesky466/ShaderCamera),它支持三种简单的特效:

{% img /安卓特效相机一/5.jpg %}

接下来的几篇文章我会一步步讲下整个程序是如何实现的。

# 整体架构

这篇文章主要讲Camera2的使用,包括预览和拍照。

Camera2的整体架构如下:

{% img /安卓特效相机一/1.png %}

一台安卓设备可能拥有多个摄像设备,比如一般手机都有前后摄像头,而每个摄像头即为一个CameraDevice。应用程序可以通过CameraManager获取到所有的摄像设备的信息,打开摄像设备然后创建一个CameraCaptureSession连接应用程序与摄像设备。之后应用程序就可以使用这个CameraCaptureSession向摄像设备发送CaptureRequest来指挥摄像头工作。

所以使用Camera2的流程大致为:

1. 从CameraManager选择摄像设备并打开
2. 创建与CameraDevice的CameraCaptureSession
3. 使用CameraCaptureSession向CameraDevice发送CaptureRequest

# 获取摄像设备信息

## 镜头朝向

通常应用程序想要使用摄像头,需要先遍历设备所有的摄像头,然后选出合适的摄像头去拍摄,例如我们想使用后置摄像头:

```
CameraManager manager = (CameraManager) context.getSystemService(Context.CAMERA_SERVICE);

try {
    for (String id : manager.getCameraIdList()) {
        CameraCharacteristics cc = manager.getCameraCharacteristics(id);
        if (cc.get(CameraCharacteristics.LENS_FACING) == facing) {
        	...
            break;
        }
    }
} catch (Exception e) {
    Log.e(TAG, "can not open camera", e);
}
```

通过CameraManager.getCameraIdList()方法可以列出所有摄像头的id,然后通过CameraManager.getCameraCharacteristics可以拿到对应摄像头的CameraCharacteristics(特征集合),通过这个CameraCharacteristics我们可以拿到摄像头的一些属性,例如上面的镜头朝向。

## 输出尺寸

应用展示摄像头画面的view大小千奇百怪,如果摄像头只能拍摄一种尺寸的画面,那屏幕上显示的时候就势必需要进行缩放了。如果view长宽比和拍摄的画面长宽比是一样的还好,只需要等比缩放就可以了。但是如果长宽比不一样那就势必要发生形变或者裁切像素了。

于是一般摄像头都会支持多种尺寸的输出画面,开发者可以种里面选取最合适的尺寸去显示。

```
Size[] previewSizes = cc.get(CameraCharacteristics.SCALER_STREAM_CONFIGURATION_MAP).getOutputSizes(SurfaceTexture.class);
Size previewSize = getMostSuitableSize(previewSizes, width, height);
preview.setDefaultBufferSize(previewSize.getWidth(), previewSize.getHeight());
...
manager.openCamera(id, mOpenCameraCallback, handler);
```

可以看到输出尺寸和想要用于显示的类相关,例如我们的demo使用SurfaceTexture去显示,就可以获取摄像头支持SurfaceTexture的所有尺寸。

然后指定输出尺寸并不是将想要的尺寸设置给摄像机,而是设置SurfaceTexture的Buffer大小,然后摄像头在将画面绘制到SurfaceTexture上的时候就会使用最接近的尺寸去绘制了。

Camera2支持将画面绘制到下面的几种目标:

- ImageReader
- MediaRecorder
- MediaCodec
- Allocation
- SurfaceHolder
- SurfaceTexture

getMostSuitableSize里面我们选择长宽比最接近width*height的尺寸:

```
 private Size getMostSuitableSize(
        Size[] sizes,
        float width,
        float height) {

    float targetRatio = height / width;
    Size result = null;
    for (Size size : sizes) {
        if (result == null || isMoreSuitable(result, size, targetRatio)) {
            result = size;
        }
    }
    return result;
}

private boolean isMoreSuitable(Size current, Size target, float targetRatio) {
    if (current == null) {
        return true;
    }
    float dRatioTarget = Math.abs(targetRatio - getRatio(target));
    float dRatioCurrent = Math.abs(targetRatio - getRatio(current));
    return dRatioTarget < dRatioCurrent
            || (dRatioTarget == dRatioCurrent && getArea(target) > getArea(current));
}

```

## 相机方向

细心的同学可能会看到这里的长宽比我用的是height/width,这是由于摄像机的方向和屏幕方向相差了90度,所以相机的长宽比应该是屏幕的宽长比。

这个摄像头方向的介绍可以看[官方文档](https://developer.android.com/reference/android/hardware/Camera.CameraInfo#orientation):


> The orientation of the camera image. The value is the angle that the camera image needs to be rotated clockwise so it shows correctly on the display in its natural orientation. It should be 0, 90, 180, or 270.
>
> For example, suppose a device has a naturally tall screen. The back-facing camera sensor is mounted in landscape. You are looking at the screen. If the top side of the camera sensor is aligned with the right edge of the screen in natural orientation, the value should be 90. If the top side of a front-facing camera sensor is aligned with the right of the screen, the value should be 270.

比方说后置摄像头的正方向就是竖着拿屏幕的时候的屏幕的右方:

{% img /安卓特效相机一/4.png %}

所以竖着拿手机的时候拍的照片其实是横的。于是我们在计算长宽比查找最时候的尺寸的时候就需要旋转90度,也就是用height/width。

# 打开摄像头

我们可以使用CameraManager.openCamera方法打开指定的摄像头:

```
private CameraDevice.StateCallback mOpenCameraCallback =
            new CameraDevice.StateCallback() {
                @Override
                public void onOpened(CameraDevice camera) {
                    openCameraSession(camera);
                }

                @Override
                public void onDisconnected(CameraDevice camera) {
                }

                @Override
                public void onError(CameraDevice camera, int error) {
                }
            };

...

manager.openCamera(id, mOpenCameraCallback, handler);
```

mOpenCameraCallback是打开结果的回调,而handler则决定了这个回调在哪个线程调用

# 连接摄像头

在打开摄像头的回调里我们可以拿到CameraDevice,然后但是我们并不能直接指挥摄像设备去干活,而是要通过CameraCaptureSession。

那怎么创建与CameraDevice的CameraCaptureSession呢？

可以通过CameraDevice.createCaptureSession

```
private CameraCaptureSession.StateCallback mCreateSessionCallback =
        new CameraCaptureSession.StateCallback() {
            @Override
            public void onConfigured(CameraCaptureSession session) {
                mCameraCaptureSession = session;
                requestPreview(session);
            }

            @Override
            public void onConfigureFailed(CameraCaptureSession session) {

            }
        };
...

private void openCameraSession(CameraDevice camera) {
        mCameraDevice = camera;
        try {
            List<Surface> outputs = Arrays.asList(mPreviewSurface);
            camera.createCaptureSession(outputs, mCreateSessionCallback, mHandler);
        } catch (CameraAccessException e) {
            Log.e(TAG, "createCaptureSession failed", e);
        }
    }
```

除了同样需要传入回调和指定回调执行线程的handler之外。

然后还需要传入一个列表告诉摄像设备它需要绘制到什么地方,Camera2支持同时往多个目标绘制画面。

但是并不是说我们这里指定mPreviewSurface,摄像头就会直接开始往里面绘制画面了,还需要发送request去请求绘制。

# 发送绘制请求

CaptureRequest的配置比较多,如果一个个去配的话比较繁琐,所以谷歌已经给我们创建好了几个常用的模板,我们可以根据自己的需求去选择一个来做修改

## 实时预览

我们用TextureView来实时预览摄像机画面:

```
mPreview.setSurfaceTextureListener(new TextureView.SurfaceTextureListener() {
    @SuppressLint("NewApi")
    @Override
    public void onSurfaceTextureAvailable(SurfaceTexture surface, int width, int height) {
        mSurfaceTexture = surface;
        mPreviewSurface = new Surface(surface);
        ...
        openCamera(mSurfaceTexture,
                CameraCharacteristics.LENS_FACING_BACK,
                mPreview.getWidth(),
                mPreview.getHeight());
        ...
    }

    @Override
    public void onSurfaceTextureSizeChanged(SurfaceTexture surface, int width, int height) {

    }

    @Override
    public boolean onSurfaceTextureDestroyed(SurfaceTexture surface) {
        return false;
    }

    @Override
    public void onSurfaceTextureUpdated(SurfaceTexture surface) {

    }
});
```

CaptureRequest这里依然需要指定将画面绘制到我们的预览Surface上:

```
CaptureRequest.Builder builder = mCameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW);
builder.addTarget(mPreviewSurface);
session.setRepeatingRequest(builder.build(), null, null);
```


值得注意的是每一次请求只会绘制一次,如果是预览界面的话需要不停绘制,我们可以使用CameraCaptureSession.setRepeatingRequest让他不断发送Request去不断的绘制,达到实时预览的功能。

这个方法的第一个参数是CaptureRequest,第二和第三个参数仍然是回调和handler,这里我们不需要监听回调,都设成null就好。

## 拍照

我们可以创建ImageReader来接收画面:

```
Size[] photoSizes = cc.get(CameraCharacteristics.SCALER_STREAM_CONFIGURATION_MAP)
                           .getOutputSizes(ImageReader.class);
mImageReader = getImageReader(getMostSuitableSize(photoSizes, width, height));

...

private ImageReader.OnImageAvailableListener mOnImageAvailableListener
            = new ImageReader.OnImageAvailableListener() {
        @Override
        public void onImageAvailable(ImageReader reader) {
            savePhoto(reader);
        }
    };

private ImageReader getImageReader(Size size) {
    ImageReader imageReader = ImageReader.newInstance(
            size.getWidth(),
            size.getHeight(),
            ImageFormat.JPEG,
            5);
    imageReader.setOnImageAvailableListener(mOnImageAvailableListener, mHandler);
    return imageReader;
}
```

在触摸屏幕的时候发送请求绘制画面到ImageReader上:

```
private void takePhoto() {
    try {
        CaptureRequest.Builder builder = mCameraDevice.createCaptureRequest(
                CameraDevice.TEMPLATE_STILL_CAPTURE);
        builder.addTarget(mPreviewSurface);
        builder.addTarget(mImageReader.getSurface());
        int rotation = getWindowManager().getDefaultDisplay().getRotation();
        builder.set(CaptureRequest.JPEG_ORIENTATION, mSensorOrientation);
        mCameraCaptureSession.capture(builder.build(), null, null);
    } catch (CameraAccessException e) {
        Log.d(TAG, "takePhoto failed", e);
    }
}
```

## 图片方向

我们这里设置了下CaptureRequest.JPEG_ORIENTATION,原因和上面说的摄像头设备的方向有关,如果不设置的话,预览的窗口里面是竖着拍的照片实际保存下来会变成横的。

这个mSensorOrientation也是从CameraCharacteristics里面获取的:

```
CameraCharacteristics cc = manager.getCameraCharacteristics(id);
...
mSensorOrientation = cc.get(CameraCharacteristics.SENSOR_ORIENTATION);
```

## 请求队列

细心的同学可能还会注意到,这个请求不仅将mImageReader.getSurface()添加到Target，同时也将mPreviewSurface添加到Target了,这是为什么呢?

不知道大家还记不记得之前我们说过,每一个CaptureRequest会执行一次绘制,实时预览靠的就是setRepeatingRequest不断重复的发送CaptureRequest。

其实CameraDevice对CaptureRequest的执行是串行的,当没有拍照的请求的时候,请求队列是这样的:

{% img /安卓特效相机一/2.png %}

而当有拍照的请求进去的时候，请求队列是这样的:

{% img /安卓特效相机一/3.png %}

预览请求中间插入了一个拍照请求,如果这个拍照请求里面没有将画面绘制到预览的View上面,预览画面就会少了一帧,相当于卡了一下。所以拍照的时候也要将mPreviewSurface放到Target中。

# 关闭摄像头

我们需要在退出应用的时候关闭摄像头,要不然可能会影响其他应用使用摄像头:

```
private CameraDevice mCameraDevice;。

...    

if (mImageReader != null) {
    mImageReader.close();
    mImageReader = null;
}

if (mCameraCaptureSession != null) {
    mCameraCaptureSession.close();
    mCameraCaptureSession = null;
}

if (mCameraDevice != null) {
    mCameraDevice.close();
    mCameraDevice = null;
}
```

本篇文章的完整代码可以在[github](https://github.com/bluesky466/CameraDemo)上获取
