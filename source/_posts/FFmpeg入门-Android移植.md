title: FFmpeg入门 - Android移植
date: 2022-10-25 20:23:18
tags:
  - 技术相关
  - 音视频
  - Android
---

系列文章:

1. [FFmpeg入门 - 视频播放](https://blog.islinjw.cn/2022/09/04/FFmpeg%E5%85%A5%E9%97%A8-%E8%A7%86%E9%A2%91%E6%92%AD%E6%94%BE/)
2. [FFmpeg入门 - rtmp推流](https://blog.islinjw.cn/2022/09/08/FFmpeg%E5%85%A5%E9%97%A8-rtmp%E6%8E%A8%E6%B5%81/)
3. [FFmpeg入门 - Android移植](https://blog.islinjw.cn/2022/10/25/FFmpeg%E5%85%A5%E9%97%A8-Android%E7%A7%BB%E6%A4%8D/)
4. [FFmpeg入门 - 格式转换](https://blog.islinjw.cn/2022/11/27/FFmpeg%E5%85%A5%E9%97%A8-%E6%A0%BC%E5%BC%8F%E8%BD%AC%E6%8D%A2/)

前两篇文章介绍了如何使用ffmpeg推流和拉流,这篇我们来看看怎样将之前的代码移植到安卓上。

# FFmpeg编译与集成

FFmpeg的安卓交叉编译网上有很多的资料,基本上都是些编译配置而已。可以直接将我的[脚本](https://github.com/bluesky466/FFmpegAndroidDemo/blob/master/ffmpeg-4.4.2/build_ffmpeg.sh)放到ffmpeg源码根目录,修改下NDK的路径和想要编译的ABI之后直接执行。然后就能在android目录里面得到编译好的so和.h

如果的确编译出现问题,也可以直接用我编出来的[库](https://github.com/bluesky466/FFmpegAndroidDemo/tree/master/app/jniLibs)。

将库放到AndroidStudio工程的jniLibs目录,将include目录放到app/src/main/cpp下,然后修改CMakeLists.txt添加ffmpeg头文件路径、库路径、链接配置等:

```
cmake_minimum_required(VERSION 3.18.1)

project("ffmpegdemo")

add_library(ffmpegdemo SHARED ffmpeg_demo.cpp video_sender.cpp opengl_display.cpp egl_helper.cpp video_decoder.cpp)

find_library(log-lib log)

# 头文件路径
include_directories(${CMAKE_SOURCE_DIR}/include)

# ffmpeg库依赖
add_library(avcodec SHARED IMPORTED)
set_target_properties(avcodec PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/../../../jniLibs/${ANDROID_ABI}/libavcodec.so)

add_library(avfilter SHARED IMPORTED)
set_target_properties(avfilter PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/../../../jniLibs/${ANDROID_ABI}/libavfilter.so)

add_library(avformat SHARED IMPORTED)
set_target_properties(avformat PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/../../../jniLibs/${ANDROID_ABI}/libavformat.so)

add_library(avutil SHARED IMPORTED)
set_target_properties(avutil PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/../../../jniLibs/${ANDROID_ABI}/libavutil.so)

add_library(swresample SHARED IMPORTED)
set_target_properties(swresample PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/../../../jniLibs/${ANDROID_ABI}/libswresample.so)

add_library(swscale SHARED IMPORTED)
set_target_properties(swscale PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/../../../jniLibs/${ANDROID_ABI}/libswscale.so)

target_link_libraries(
        ffmpegdemo

        # log
        ${log-lib}

        EGL
        GLESv2
        android

        # FFmpeg libs
        avcodec
        avfilter
        avformat
        avutil
        swresample
        swscale
)
```

这样一套下来其实ffmpeg的安卓环境就整好了,我们把之前的[video_sender.cpp](https://github.com/bluesky466/FFmpegDemo/blob/main/video_sender.cpp)和[video_sender.h](https://github.com/bluesky466/FFmpegDemo/blob/main/video_sender.h)拷贝过来添加个jni的接口验证下推流:

```
// java
File file = new File(getFilesDir(), "video.flv");

try {
    InputStream is = getAssets().open("video.flv");
    OutputStream os = new FileOutputStream(file);
    FileUtils.copy(is, os);
} catch (Exception e) {
    Log.d("FFmpegDemo", "err", e);
}

new Thread(new Runnable() {
    @Override
    public void run() {
        send(file.getAbsolutePath(), "rtmp://" + SERVER_IP + "/live/livestream");
    }
}).start();
```

```
//jni
extern "C" JNIEXPORT void JNICALL
Java_me_linjw_demo_ffmpeg_MainActivity_send(
        JNIEnv *env,
        jobject /* this */,
        jstring srcFile,
        jstring destUrl) {
    const char *src = env->GetStringUTFChars(srcFile, NULL);
    const char *dest = env->GetStringUTFChars(destUrl, NULL);
    LOGD("send: %s -> %s", src, dest);
    VideoSender::Send(src, dest);
}
```

然后就可以用安卓去推流,在pc上用之前的demo进行播放验证。

# OpenGLES播放FFmpeg

之前的[demo](https://blog.islinjw.cn/2022/09/04/FFmpeg%E5%85%A5%E9%97%A8-%E8%A7%86%E9%A2%91%E6%92%AD%E6%94%BE/)使用SDL2播放视频,但是安卓上更常规的做法是通过OpenGLES去播放。其实之前在做摄像教程的时候已经有介绍过OpenGLES的使用了:

[安卓特效相机(二) EGL基础](https://blog.islinjw.cn/2019/09/13/%E5%AE%89%E5%8D%93%E7%89%B9%E6%95%88%E7%9B%B8%E6%9C%BA-%E4%BA%8C-EGL%E5%9F%BA%E7%A1%80/)

[安卓特效相机(三) OpenGL ES 特效渲染](https://blog.islinjw.cn/2019/09/22/%E5%AE%89%E5%8D%93%E7%89%B9%E6%95%88%E7%9B%B8%E6%9C%BA-%E4%B8%89-OpenGL-ES-%E7%89%B9%E6%95%88%E6%B8%B2%E6%9F%93/)

这篇我们就只补充下之前没有提到的部分。


## YUV 

首先有个很重要的知识点在于我们的视频很多情况下解码出来都是[YUV格式](https://zh.wikipedia.org/zh-sg/YUV)的画面而不是安卓应用开发常见的RGB格式。

YUV是编译true-color颜色空间（color space）的种类，Y'UV, YUV, YCbCr，YPbPr等专有名词都可以称为YUV，彼此有重叠。“Y”表示明亮度（Luminance、Luma），“U”和“V”则是色度、浓度（Chrominance、Chroma）,也就是说通过UV可以选择到一种颜色:


{% img /FFmpeg入门Android移植/1.png %}

然后再加上这种颜色的亮度就能代表我们实际看到的颜色。

YUV的发明是由于彩色电视与黑白电视的过渡时期,黑白电视只有亮度的值(Y)到了彩色电视的时代为了兼容之前的黑白电视,于是在亮度值后面加上了UV值指定颜色,如果忽略了UV那么剩下的Y,就和黑白电视的信号保持一致。

这种情况下数据是以 **平面格式(planar formats)** 去保存的,类似YYYYUUUUVVVV,YUV三者分开存放。
另外也有和常见的RGB存放方式类似的 **紧缩格式(packed formats)** ,类似YUVYUVYUV,每个像素点的YUV数据连续存放。

由于人的肉眼对亮度敏感对颜色相对不敏感,所以我们可以相邻的几个像素共用用UV信息,减少数据带宽。

这里的共用UV信息并没有对多个像素点做UV数据的均值,而是简单的跳过一些像素点不去读取他们的UV数据。

### YUV444

每个像素都有自己的YUV数据,每个像素占用Y + U + V = 8 + 8 + 8 = 24 bits

{% img /FFmpeg入门Android移植/YUV444.png %}

444的含义是同一行相邻的4个像素,分别采样4个Y,4个U,4个V

### YUV422 

每两个像素共用一对UV分量,每像素平均占用Y + U + V = 8 + 4 + 4 = 16 bits

{% img /FFmpeg入门Android移植/YUV422.png %}

422的含义是同一行相邻的4个像素,分别采样4个Y,2个U,2个V

### YUV420

每四个像素共用一对UV分量,每像素平均占用Y + U + V = 8 + 2 + 2 = 12 bits

{% img /FFmpeg入门Android移植/YUV420.png %}

YUV420在YUV422的基础上再隔行扫描UV信息,一行只采集U,下一行只采集V

420的含义是同一行相邻的4个像素,分别采样4个Y,2个U,0个V,或者4个Y,0个U,2个V


## OpenGLES显示YUV图像

由于OpenGLES使用RGB色彩,所以我们需要在fragmentShader里面将YUV转成RGB,转换公式如下:

R = Y + 1.4075 * V;  
G = Y - 0.3455 * U - 0.7169*V;  
B = Y + 1.779 * U;

由于解码之后的数据使用平面格式(planar formats)保存,所以我们可以创建三张灰度图图片分别存储YUV的分量,另外由于OpenGLES里面色彩的值范围是0\~1.0,而UV分量的取值范围是-0.5\~0.5所以我们UV分量统一减去0.5做偏移.于是fragmentShader代码如下:

```
static const string FRAGMENT_SHADER = "#extension GL_OES_EGL_image_external : require\n"
                                      "precision highp float;\n"
                                      "varying vec2 vCoord;\n"
                                      "uniform sampler2D texY;\n"
                                      "uniform sampler2D texU;\n"
                                      "uniform sampler2D texV;\n"
                                      "varying vec4 vColor;\n"
                                      "void main() {\n"
                                      "    float y = texture2D(texY, vCoord).x;\n"
                                      "    float u = texture2D(texU, vCoord).x - 0.5;\n"
                                      "    float v = texture2D(texV, vCoord).x - 0.5;\n"
                                      "    float r = y + 1.4075 * v;\n"
                                      "    float g = y - 0.3455 * u - 0.7169 * v;\n"
                                      "    float b = y + 1.779 * u;\n"
                                      "    gl_FragColor = vec4(r, g, b, 1);\n"
                                      "}";
```

接着由于OpenGLES里面纹理坐标原点是左下角,而解码的画面原点是左上角,所以纹理坐标需要上下调换一下:

```
static const float VERTICES[] = {
        -1.0f, 1.0f,
        -1.0f, -1.0f,
        1.0f, -1.0f,
        1.0f, 1.0f
};

// 由于OpenGLES里面纹理坐标原点是左下角,而解码的画面原点是左上角,所以纹理坐标需要上下调换一下
static const float TEXTURE_COORDS[] = {
        0.0f, 0.0f,
        0.0f, 1.0f,
        1.0f, 1.0f,
        1.0f, 0.0f
};

static const short ORDERS[] = {
        0, 1, 2, // 左下角三角形

        2, 3, 0  // 右上角三角形
};
```

最后就只要将每帧解析出来的图像交给OpenGLES去渲染就好:

```
AVFrame *frame;
while ((frame = decoder.NextFrame()) != NULL) {
    eglHelper.MakeCurrent();
    display.Render(frame->data, frame->linesize);
    eglHelper.SwapBuffers();
}
```
## linesize

接着我们就需要根据这些YUV数据创建三个灰度图分别存储各个分量的数据。这里有个知识点,解码得到的YUV数据,高是对应分量的高,但是宽却不一定是对应分量的宽.

这是因为在做视频解码的时候会对宽进行对齐,让宽是16或者32的整数倍,具体是16还是32由cpu决定.例如我们的video.flv视频,原始画面尺寸是289\*160,如果按32去对齐的话,他的Y分量的宽则是320.

对齐之后的宽在ffmpeg里面称为linesize,而由于我们这个demo只支持YUV420的格式,它的Y分量的高度为原始图像的高度,UV分量的高度由于是隔行扫描,所以是原生图像高度的一半:

```
void OpenGlDisplay::Render(uint8_t *yuv420Data[3], int lineSize[3]) {
    // 解码得到的YUV数据,高是对应分量的高,但是宽却不一定是对应分量的宽
    // 这是因为在做视频解码的时候会对宽进行对齐,让宽是16或者32的整数倍,具体是16还是32由cpu决定
    // 例如我们的video.flv视频,原始画面尺寸是689x405,如果按32去对齐的话,他的Y分量的宽则是720
    // 对齐之后的宽在ffmpeg里面称为linesize
    // 而对于YUV420来说Y分量的高度为原始图像的高度,UV分量的高度由于是隔行扫描,所以是原生图像高度的一半
    setTexture(0, "texY", yuv420Data[0], lineSize[0], mVideoHeight);
    setTexture(1, "texU", yuv420Data[1], lineSize[1], mVideoHeight / 2);
    setTexture(2, "texV", yuv420Data[2], lineSize[2], mVideoHeight / 2);

    // 由于对齐之后创建的纹理宽度大于原始画面的宽度,所以如果直接显示,视频的右侧会出现异常
    // 所以我们将纹理坐标进行缩放,忽略掉右边对齐多出来的部分
    GLint scaleX = glGetAttribLocation(mProgram, "aCoordScaleX");
    glVertexAttrib1f(scaleX, mVideoWidth * 1.0f / lineSize[0]);

    glClear(GL_DEPTH_BUFFER_BIT | GL_COLOR_BUFFER_BIT);
    glDrawElements(GL_TRIANGLES, sizeof(ORDERS) / sizeof(short), GL_UNSIGNED_SHORT, ORDERS);
}
```

另外由于对齐之后创建的纹理宽度大于原始画面的宽度,所以如果直接显示,视频的右侧会出现异常:

{% img /FFmpeg入门Android移植/2.png %}

所以我们将纹理坐标进行缩放,忽略掉右边对齐多出来的部分:

```
// VERTICES_SHADER
vCoord = vec2(aCoord.x * aCoordScaleX, aCoord.y);
```

## 保持视频长宽比

虽然视频能正常播放了，但是可以看到整个视频是铺满屏幕的。所以我们需要对视频进行缩放让他保持长宽比然后屏幕居中:

```
void OpenGlDisplay::SetVideoSize(int videoWidth, int videoHeight) {
    mVideoWidth = videoWidth;
    mVideoHeight = videoHeight;

    // 如果不做处理(-1.0f, 1.0f),(-1.0f, -1.0f),(1.0f, -1.0f),(1.0f, 1.0f)这个矩形会铺满整个屏幕导致图像拉伸
    // 由于坐标的原点在屏幕中央,所以只需要判断是横屏还是竖屏然后对x轴或者y轴做缩放就能让图像屏幕居中,然后恢复原始视频的长宽比
    if (mWindowHeight > mWindowWidth) {
        // 如果是竖屏的话,图像的宽不需要缩放,图像的高缩小使其竖直居中
        GLint scaleX = glGetAttribLocation(mProgram, "aPosScaleX");
        glVertexAttrib1f(scaleX, 1.0f);

        // y坐标 * mWindowWidth / mWindowHeight 得到屏幕居中的正方形
        // 然后再 * videoHeight / videoWidth 就能恢复原始视频的长宽比
        float r = 1.0f * mWindowWidth / mWindowHeight * videoHeight / videoWidth;
        GLint scaleY = glGetAttribLocation(mProgram, "aPosScaleY");
        glVertexAttrib1f(scaleY, r);
    } else {
        // 如果是横屏的话,图像的高不需要缩放,图像的宽缩小使其水平居中
        GLint scaleY = glGetAttribLocation(mProgram, "aPosScaleY");
        glVertexAttrib1f(scaleY, 1.0f);

        // x坐标 * mWindowHeight / mWindowWidth 得到屏幕居中的正方形
        // 然后再 * videoWidth / videoHeight 就能恢复原始视频的长宽比
        float r = 1.0f * mWindowHeight / mWindowWidth * videoWidth / videoHeight;
        GLint scaleX = glGetAttribLocation(mProgram, "aPosScaleX");
        glVertexAttrib1f(scaleX, r);
    }
}
```

```
// VERTICES_SHADER
gl_Position = vec4(aPosition.x * aPosScaleX, aPosition.y * aPosScaleY, 0, 1);
```

{% img /FFmpeg入门Android移植/3.jpeg %}


# Demo工程

完整的代码已经上传到[Github](https://github.com/bluesky466/FFmpegAndroidDemo)