title: FFmpeg入门 - 格式转换
date: 2022-11-27 00:07:37
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

我们现在已经能在安卓上播放视频画面了,但是声音部分还是缺失的,这篇博客就来把视频的音频播放模块也加上。

为了音频和视频可以分别解码播放,我们需要对之前的代码做重构,将媒体流的读取和解码解耦:

{% plantuml %}
class MediaReader {
    + AVPacket *NextPacket(int streamIndex)
    + std::vector<AVStream *> GetStreams()
    + AVStream *GetStream(int index)
}

class StreamDecoder {
    + bool Init(MediaReader *reader, int streamIndex)
    + void Destroy()
    + AVFrame *NextFrame()
}

class VideoStreamDecoder extends StreamDecoder {
    + int GetVideoWidth()
    + int GetVideoHeight()
}

class AudioStreamDecoder extends StreamDecoder {
    + int GetSampleRate()
    + int GetChannelCount()
    + int GetBytePerSample()
}

StreamDecoder --> MediaReader
{% endplantuml %}

MediaReader从文件流中读取出AVPacket交由VideoStreamDecoder和AudioStreamDecoder做视频与音频的解码。我们在MediaReader里加上线程安全机制,使得视频和音频可以分别在各自的工作线程中进行解码。


# 音频分⽚(plane)与打包(packed)

解码出来的AVFrame,它的data字段放的是视频像素数据或者音频的PCM裸流数据,linesize字段放的是对齐后的画面行长度或者音频的分片长度:

```
   /**
    * For video, size in bytes of each picture line.
    * For audio, size in bytes of each plane.
    *
    * For audio, only linesize[0] may be set. For planar audio, each channel
    * plane must be the same size.
    *
    * For video the linesizes should be multiples of the CPUs alignment
    * preference, this is 16 or 32 for modern desktop CPUs.
    * Some code requires such alignment other code can be slower without
    * correct alignment, for yet other it makes no difference.
    *
    * @note The linesize may be larger than the size of usable data -- there
    * may be extra padding present for performance reasons.
    */
    int linesize[AV_NUM_DATA_POINTERS];
```

视频相关的在之前的[博客](https://blog.islinjw.cn/2022/10/25/FFmpeg%E5%85%A5%E9%97%A8-Android%E7%A7%BB%E6%A4%8D/#linesize)中有介绍,音频的话可以看到它只有linesize[0]会被设置,如果有多个分片,每个分片的size都是相等的。

要理解这里的分片size,先要理解音频数据的两种存储格式分⽚(plane)与打包(packed)。以常见的双声道音频为例子,

分⽚存储的数据左声道和右声道分开存储,左声道存储在data[0],右声道存储在data[1],他们的数据buffer的size都是linesize[0]。

打包存储的数据按照LRLRLR...的形式交替存储在data[0]中,这个数据buffer的size是linesize[0]。

AVSampleFormat枚举音频的格式,带P后缀的格式是分配存储的:

```
AV_SAMPLE_FMT_U8P,         ///< unsigned 8 bits, planar
AV_SAMPLE_FMT_S16P,        ///< signed 16 bits, planar
AV_SAMPLE_FMT_S32P,        ///< signed 32 bits, planar
AV_SAMPLE_FMT_FLTP,        ///< float, planar
AV_SAMPLE_FMT_DBLP,        ///< double, planar
```
不带P后缀的格式是打包存储的:

```
AV_SAMPLE_FMT_U8,          ///< unsigned 8 bits
AV_SAMPLE_FMT_S16,         ///< signed 16 bits
AV_SAMPLE_FMT_S32,         ///< signed 32 bits
AV_SAMPLE_FMT_FLT,         ///< float
AV_SAMPLE_FMT_DBL,         ///< double
```

# 音频数据的实际长度

这里有个坑点备注里面也写的很清楚了,linesize标明的大小可能会大于实际的音视频数据大小,因为可能会有额外的填充。

> * @note The linesize may be larger than the size of usable data -- there
> * may be extra padding present for performance reasons.

所以音频数据实际的长度需要用音频的参数计算出来:

```
int channelCount = audioStreamDecoder.GetChannelCount();
int bytePerSample = audioStreamDecoder.GetBytePerSample();
int size = frame->nb_samples * channelCount * bytePerSample;
```

# 音频格式转换

视频之前的demo中已经可以使用OpenGL播放,而音频可以交给OpenSL来播放,之前我写过一篇[《OpenSL ES 学习笔记》](https://blog.islinjw.cn/2018/09/01/OpenSLES%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/)详细的使用细节我就不展开介绍了,直接将[代码](https://github.com/bluesky466/OpenSLDemo/blob/master/app/src/main/cpp/opensl_helper.c)拷贝来使用。

但是由于OpenSLES只支持打包的几种音频格式:

```
#define SL_PCMSAMPLEFORMAT_FIXED_8	((SLuint16) 0x0008)
#define SL_PCMSAMPLEFORMAT_FIXED_16	((SLuint16) 0x0010)
#define SL_PCMSAMPLEFORMAT_FIXED_20 	((SLuint16) 0x0014)
#define SL_PCMSAMPLEFORMAT_FIXED_24	((SLuint16) 0x0018)
#define SL_PCMSAMPLEFORMAT_FIXED_28 	((SLuint16) 0x001C)
#define SL_PCMSAMPLEFORMAT_FIXED_32	((SLuint16) 0x0020)
```

这里我们指的AudioStreamDecoder的目标格式为AV\_SAMPLE\_FMT\_S16,如果原始音频格式不是它,则对音频做转码:

```c++
audioStreamDecoder.Init(reader, audioIndex, AVSampleFormat::AV_SAMPLE_FMT_S16);


bool AudioStreamDecoder::Init(MediaReader *reader, int streamIndex, AVSampleFormat sampleFormat) {
    ...

    bool result = StreamDecoder::Init(reader, streamIndex);

    if (sampleFormat == AVSampleFormat::AV_SAMPLE_FMT_NONE) {
        mSampleFormat = mCodecContext->sample_fmt;
    } else {
        mSampleFormat = sampleFormat;
    }

    if (mSampleFormat != mCodecContext->sample_fmt) {
        mSwrContext = swr_alloc_set_opts(
                NULL,
                mCodecContext->channel_layout,
                mSampleFormat,
                mCodecContext->sample_rate,
                mCodecContext->channel_layout,
                mCodecContext->sample_fmt,
                mCodecContext->sample_rate,
                0,
                NULL);
        swr_init(mSwrContext);

        // 虽然前面的swr_alloc_set_opts已经设置了这几个参数
        // 但是用于接收的AVFrame不设置这几个参数也会接收不到数据
        // 原因是后面的swr_convert_frame函数会通过av_frame_get_buffer创建数据的buff
        // 而av_frame_get_buffer需要AVFrame设置好这些参数去计算buff的大小
        mSwrFrame = av_frame_alloc();
        mSwrFrame->channel_layout = mCodecContext->channel_layout;
        mSwrFrame->sample_rate = mCodecContext->sample_rate;
        mSwrFrame->format = mSampleFormat;
    }
    return result;
}

AVFrame *AudioStreamDecoder::NextFrame() {
    AVFrame *frame = StreamDecoder::NextFrame();
    if (NULL == frame) {
        return NULL;
    }
    if (NULL == mSwrContext) {
        return frame;
    }

    swr_convert_frame(mSwrContext, mSwrFrame, frame);
    return mSwrFrame;
}
```

这里我们使用swr\_convert\_frame进行转码:

```c++
int swr_convert_frame(SwrContext *swr,     // 转码上下文
                      AVFrame *output,     // 转码后输出到这个AVFrame
                      const AVFrame *input // 原始输入AVFrame
);
```

这个方法要求输入输出的AVFrame都设置了channel_layout、 sample_rate、format参数,然后回调用av_frame_get_buffer为output创建数据buff:

```c++
/**
 * ...
 *
 * Input and output AVFrames must have channel_layout, sample_rate and format set.
 *
 * If the output AVFrame does not have the data pointers allocated the nb_samples
 * field will be set using av_frame_get_buffer()
 * is called to allocate the frame.
 * ...
 */
int swr_convert_frame(SwrContext *swr,
                      AVFrame *output, const AVFrame *input);
```

SwrContext为转码的上下文,通过swr\_alloc\_set\_opts和swr\_init创建,需要把转码前后的音频channel\_layout、 sample\_rate、format信息传入:

```c++
struct SwrContext *swr_alloc_set_opts(struct SwrContext *s,
                                      int64_t out_ch_layout, enum AVSampleFormat out_sample_fmt, int out_sample_rate,
                                      int64_t  in_ch_layout, enum AVSampleFormat  in_sample_fmt, int  in_sample_rate,
                                      int log_offset, void *log_ctx);

int swr_init(struct SwrContext *s);
```

# 视频格式转换

之前的demo里面我们判断了视频格式不为AV\_PIX\_FMT\_YUV420P则直接报错,这里我们仿照音频转换的例子,判断原始视频格式不为AV\_PIX\_FMT\_YUV420P则使用sws\_scale进行格式转换:

```c++
bool VideoStreamDecoder::Init(MediaReader *reader, int streamIndex, AVPixelFormat pixelFormat) {
    ...
    bool result = StreamDecoder::Init(reader, streamIndex);
    if (AVPixelFormat::AV_PIX_FMT_NONE == pixelFormat) {
        mPixelFormat = mCodecContext->pix_fmt;
    } else {
        mPixelFormat = pixelFormat;
    }

    if (mPixelFormat != mCodecContext->pix_fmt) {
        int width = mCodecContext->width;
        int height = mCodecContext->height;

        mSwrFrame = av_frame_alloc();

        // 方式一,使用av_frame_get_buffer创建数据存储空间,av_frame_free的时候会自动释放
        mSwrFrame->width = width;
        mSwrFrame->height = height;
        mSwrFrame->format = mPixelFormat;
        av_frame_get_buffer(mSwrFrame, 0);

        // 方式二,使用av_image_fill_arrays指定存储空间,需要我们手动调用av_malloc、av_free去创建、释放空间
//        unsigned char* buffer = (unsigned char *)av_malloc(
//                av_image_get_buffer_size(mPixelFormat, width, height, 16)
//        );
//        av_image_fill_arrays(mSwrFrame->data, mSwrFrame->linesize, buffer, mPixelFormat, width, height, 16);

        mSwsContext = sws_getContext(
                mCodecContext->width, mCodecContext->height, mCodecContext->pix_fmt,
                width, height, mPixelFormat, SWS_BICUBIC,
                NULL, NULL, NULL
        );
    }
    return result;
}


AVFrame *VideoStreamDecoder::NextFrame() {
    AVFrame *frame = StreamDecoder::NextFrame();
    if (NULL == frame) {
        return NULL;
    }
    if (NULL == mSwsContext) {
        return frame;
    }

    sws_scale(mSwsContext, frame->data,
              frame->linesize, 0, mCodecContext->height,
              mSwrFrame->data, mSwrFrame->linesize);
    return mSwrFrame;
}
```

sws\_scale看名字虽然是缩放,但它实际上也会对format进行转换,转换的参数由SwsContext提供:

```c++
struct SwsContext *sws_getContext(
    int srcW,                     // 源图像的宽
    int srcH,                     // 源图像的高
    enum AVPixelFormat srcFormat, // 源图像的格式
    int dstW,                     // 目标图像的宽
    int dstH,                     // 目标图像的高
    enum AVPixelFormat dstFormat, // 目标图像的格式
    int flags,                    // 暂时可忽略
    SwsFilter *srcFilter,         // 暂时可忽略
    SwsFilter *dstFilter,         // 暂时可忽略
    const double *param           // 暂时可忽略
);
```

sws\_scale支持区域转码,可以如我们的demo将整幅图像进行转码,也可以将图像切成多个区域分别转码,这样方便实用多线程加快转码效率:

```c++
int sws_scale(
    struct SwsContext *c,             // 转码上下文
    const uint8_t *const srcSlice[],  // 源画面区域像素数据,对应源AVFrame的data字段
    const int srcStride[],            // 源画面区域行宽数据,对应源AVFrame的linesize字段
    int srcSliceY,                    // 源画面区域起始Y坐标,用于计算应该放到目标图像的哪个位置
    int srcSliceH,                    // 源画面区域行数,用于计算应该放到目标图像的哪个位置
    uint8_t *const dst[],             // 转码后图像数据存储,对应目标AVFrame的data字段
    const int dstStride[]             // 转码后行宽数据存储,对应目标AVFrame的linesize字段
);
```
srcSlice和srcStride存储了源图像部分区域的图像数据,srcSliceY和srcSliceH告诉转码器这部分区域的坐标范围,用于计算偏移量将转码结果存放到dst和dstStride中。

例如下面的代码就将一幅完整的图像分成上下两部分分别进行转码:

```c++
int halfHeight = mCodecContext->height / 2;

// 转码上半部分图像
uint8_t *dataTop[AV_NUM_DATA_POINTERS] = {
        frame->data[0],
        frame->data[1],
        frame->data[2]
};
sws_scale(mSwsContext, dataTop,
            frame->linesize, 0,
            halfHeight,
            mSwrFrame->data, mSwrFrame->linesize);

// 转码下半部分图像
uint8_t *dataBottom[AV_NUM_DATA_POINTERS] = {
        frame->data[0] + (frame->linesize[0] * halfHeight),
        frame->data[1] + (frame->linesize[1] * halfHeight),
        frame->data[2] + (frame->linesize[2] * halfHeight),
};
sws_scale(mSwsContext, dataBottom,
            frame->linesize, halfHeight,
            mCodecContext->height - halfHeight,
            mSwrFrame->data, mSwrFrame->linesize);
```

# AVFrame内存管理机制

我们创建了一个新的AVFrame用于接收转码后的图像:

```c++
mSwrFrame = av_frame_alloc();

// 方式一,使用av_frame_get_buffer创建数据存储空间,av_frame_free的时候会自动释放
mSwrFrame->width = width;
mSwrFrame->height = height;
mSwrFrame->format = mPixelFormat;
av_frame_get_buffer(mSwrFrame, 0);

// 方式二,使用av_image_fill_arrays指定存储空间,需要我们手动调用av_malloc、av_free去创建、释放buffer的空间
// int bufferSize = av_image_get_buffer_size(mPixelFormat, width, height, 16);
// unsigned char* buffer = (unsigned char *)av_malloc(bufferSize);
// av_image_fill_arrays(mSwrFrame->data, mSwrFrame->linesize, buffer, mPixelFormat, width, height, 16);
```

av\_frame\_alloc创建出来的AVFrame只是一个壳,我们需要为它提供实际存储像素数据和行宽数据的内存空间,如上所示有两种方法:

1.通过av\_frame\_get\_buffer创建存储空间,data成员的空间实际上是由buf[0]->data提供的:

```c++
LOGD("mSwrFrame --> buf : 0x%X~0x%X, data[0]: 0x%X, data[1]: 0x%X, data[2]: 0x%X",
    mSwrFrame->buf[0]->data,
    mSwrFrame->buf[0]->data + mSwrFrame->buf[0]->size,
    mSwrFrame->data[0],
    mSwrFrame->data[1],
    mSwrFrame->data[2]
);
// mSwrFrame --> buf : 0x2E6E8AC0~0x2E753F40, data[0]: 0x2E6E8AC0, data[1]: 0x2E7302E0, data[2]: 0x2E742100
```

2. 通过av\_image\_fill\_arrays指定外部存储空间,data成员的空间就是我们指的的外部空间,而buf成员是NULL:

```c++
LOGD("mSwrFrame --> buffer : 0x%X~0x%X, buf : 0x%X, data[0]: 0x%X, data[1]: 0x%X, data[2]: 0x%X",
    buffer,
    buffer + bufferSize,
    mSwrFrame->buf[0],
    mSwrFrame->data[0],
    mSwrFrame->data[1],
    mSwrFrame->data[2]
);
// FFmpegDemo: mSwrFrame --> buffer : 0x2DAE4DC0~0x2DB4D5C0, buf : 0x0, data[0]: 0x2DAE4DC0, data[1]: 0x2DB2A780, data[2]: 0x2DB3BEA0
```

而av\_frame\_free内部会去释放AVFrame里buf的空间,对于data成员它只是简单的把指针赋值为0,所以通过av\_frame\_get\_buffer创建存储空间,而通过av\_image\_fill\_arrays指定外部存储空间需要我们手动调用av\_free去释放外部空间。

# align

细心的同学可能还看到了av\_image\_get\_buffer\_size和av\_image\_fill\_arrays都传了个16的align,这里对应的就是之前讲的linesize的字节对齐,会填充数据让linesize变成16、或者32的整数倍:

```c++
@param align         the value used in src for linesize alignment
```

这里如果为0会填充失败:

{% img /FFmpeg入门格式转换/1.png %}


而为1不做填充会出现和实际解码中的linesize不一致导致画面异常:

{% img /FFmpeg入门格式转换/2.png %}

av\_frame\_get\_buffer则比较人性化,它推荐你填0让它自己去判断应该按多少对齐:

```
 * @param align Required buffer size alignment. If equal to 0, alignment will be
 *              chosen automatically for the current CPU. It is highly
 *              recommended to pass 0 here unless you know what you are doing.
```

# 完整代码

完整的demo代码已经放到[Github](https://github.com/bluesky466/FFmpegAndroidDemo/tree/feature_conversion)上,感兴趣的同学可以下载来看看