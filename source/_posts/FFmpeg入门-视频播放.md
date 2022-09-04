title: FFmpeg入门 - 视频播放
date: 2022-09-04 12:53:54
tags:
  - 技术相关
  - 音视频
---


音视频最好从能够直接看到东西,也更加贴近用户的播放开始学起.

# 音视频编解码基础

我们可以通过http、rtmp或者本地的视频文件去播放视频。这里的"视频"实际上指的是mp4、avi这种既有音频也有视频的文件格式。

这样的视频文件可能会有多条轨道例如视频轨道、音频轨道、字幕轨道等.
有些格式限制比较多,例如AVI视频轨道只能有一条,音频轨道也只能有一条.
而有些格式则比较灵活,例如OGG视频的视频、音频轨道都能有多条.

像音频、视频这种数据量很大的轨道,上面的数据实际上都是通过压缩的。
视频轨道上可能是H264、H256这样压缩过的图像数据,通过解码可以还原成YUV、RGB等格式的图像数据。
音频轨道上可能是MP3、AAC这样压缩过的的音频数据,通过解码可以还原成PCM的音频裸流。


{% plantuml %}

@startuml
@startjson
{
    "视频文件":{
        "视频轨道":{"H264、H265等格式的压缩音频":["YUV、RGB等格式的图像"]}, 
        "音频轨道":{"AAC、MP3等格式的压缩音频":["PCM格式的音频"]}, 
        "字幕轨道":""
    }
}
@endjson
@enduml

{% endplantuml %}

实际上使用ffmpeg去播放视频也就是根据文件的格式一步步还原出图像数据交给显示设备显示、还原出音频数据交给音频设备播放:

{% plantuml %}
partition 协议层 {
(*) --> 数据
note right
http,rtmp,file...
end note
}

partition 封装格式层 {
-->[解协议] 封装格式数据
note right
mkv,mp4,mpegts,avi...
end note
}

if 解封装并且分流 then

partition 编解码层 {
-->[音频] 音频码流数据
note right
aac,mp3,...
end note
}

partition 原始数据层 {
--> [音频解码] 音频采样数据
note right
pcm
end note
}
-->音频设备播放

else

partition 编解码层 {
-->[视频] 视频码流数据
note right
h264,h265,mpeg2...
end note
}

partition 原始数据层 {
--> [视频解码] 视频像素数据
note right
yuv420p,yuv422p,rgb888...
end note
}
--> 显示设备显示

endif
{% endplantuml %}

# ffmpeg简单入门

了解了视频的播放流程之后我们来做一个简单的播放器实际入门一下ffmpeg。由于这篇博客是入门教程,这个播放器功能会进行简化:

1. 使用ffmpeg 4.4.2版本 - 4.x的版本被使用的比较广泛,而且最新的5.x版本资料比较少
1. 只解码一个视频轨道的画面进行播放 - 不需要考虑音视频同步的问题
2. 使用SDL2在主线程解码 - 不需要考虑多线程同步问题
3. 使用源码+Makefile构建 - 在MAC和Ubuntu上验证过,Windows的同学需要自己创建下vs的工程了

使用ffmpeg去解码大概有下面的几个步骤和关键函数,大家可以和上面的流程图对应一下:

### 解析文件流(解协议和解封装)

1. avformat\_open\_input : 可以打开File、RTMP等协议的数据流,并且读取文件头解析出视频信息,如解析出各个轨道和时长等
2. avformat\_find\_stream\_info : 对于没有文件头的格式如MPEG或者H264裸流等,可以通过这个函数解析前几帧得到视频的信息

### 创建各个轨道的解码器(分流)

1. avcodec\_find\_decoder: 查找对应的解码器
2. avcodec\_alloc\_context3: 创建解码器上下文
3. avcodec\_parameters\_to\_context: 设置解码所需要的参数
4. avcodec\_open2: 打开解码器

### 使用对应的解码器解码各个轨道(解码)

1. av\_read\_frame: 从视频流读取视频数据包
2. avcodec\_send\_packet: 发送视频数据包给解码器解码
3. avcodec\_receive\_frame: 从解码器读取解码后的帧数据

为了几种精力在音视频部分，我拆分出了专门进行解码的VideoDecoder类和专门进行画面显示的SdlWindow类,大家主要关注VideoDecoder部分即可。

# 视频流解析

由于实际解码前的解析文件流和创建解码器代码比较固定化,我直接将代码贴出来,大家可能跟着注释看下每个步骤的含义:

```c++
bool VideoDecoder::Load(const string& url) {
    mUrl = url;

    // 打开文件流读取文件头解析出视频信息如轨道信息、时长等
    // mFormatContext初始化为NULL,如果打开成功,它会被设置成非NULL的值,在不需要的时候可以通过avcodec_free_context释放。
    // 这个方法实际可以打开多种来源的数据,url可以是本地路径、rtmp地址等
    // 在不需要的时候通过avformat_close_input关闭文件流
    if(avformat_open_input(&mFormatContext, url.c_str(), NULL, NULL) < 0) {
        cout << "open " << url << " failed" << endl;
        return false;
    }

    // 对于没有文件头的格式如MPEG或者H264裸流等,可以通过这个函数解析前几帧得到视频的信息
    if(avformat_find_stream_info(mFormatContext, NULL) < 0) {
        cout << "can't find stream info in " << url << endl;
        return false;
    }

    // 查找视频轨道,实际上我们也可以通过遍历AVFormatContext的streams得到,代码如下:
    // for(int i = 0 ; i < mFormatContext->nb_streams ; i++) {
    //     if(mFormatContext->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) {
    //         mVideoStreamIndex = i;
    //         break;
    //     }
    // }
    mVideoStreamIndex = av_find_best_stream(mFormatContext, AVMEDIA_TYPE_VIDEO, -1, -1, NULL, 0);
    if(mVideoStreamIndex < 0) {
        cout << "can't find video stream in " << url << endl;
        return false;
    }

    // 获取视频轨道的解码器相关参数
    AVCodecParameters* codecParam = mFormatContext->streams[mVideoStreamIndex]->codecpar;
    cout << "codec id = " << codecParam->codec_id << endl;
    
    // 通过codec_id获取到对应的解码器
    // codec_id是enum AVCodecID类型,我们可以通过它知道视频流的格式,如AV_CODEC_ID_H264(0x1B)、AV_CODEC_ID_H265(0xAD)等
    // 当然如果是音频轨道的话它的值可能是AV_CODEC_ID_MP3(0x15001)、AV_CODEC_ID_AAC(0x15002)等
    AVCodec* codec = avcodec_find_decoder(codecParam->codec_id);
    if(codec == NULL) {
        cout << "can't find codec" << endl;
        return false;
    }

    // 创建解码器上下文,解码器的一些环境就保存在这里
    // 在不需要的时候可以通过avcodec_free_context释放
    mCodecContext = avcodec_alloc_context3(codec);
    if (mCodecContext == NULL) {
        cout << "can't alloc codec context" << endl;
        return false;
    }


    // 设置解码器参数
    if(avcodec_parameters_to_context(mCodecContext, codecParam) < 0) {
        cout << "can't set codec params" << endl;
        return false;
    }

    // 打开解码器,从源码里面看到在avcodec_free_context释放解码器上下文的时候会close,
    // 所以我们可以不用自己调用avcodec_close去关闭
    if(avcodec_open2(mCodecContext, codec, NULL) < 0) {
        cout << "can't open codec" << endl;
        return false;
    }

    // 创建创建AVPacket接收数据包
    // 无论是压缩的音频流还是压缩的视频流,都是由一个个数据包组成的
    // 解码的过程实际就是从文件流中读取一个个数据包传给解码器去解码
    // 对于视频，它通常应包含一个压缩帧
    // 对于音频，它可能是一段压缩音频、包含多个压缩帧
    // 在不需要的时候可以通过av_packet_free释放
    mPacket = av_packet_alloc();
    if(NULL == mPacket) {
        cout << "can't alloc packet" << endl;
        return false;
    }

    // 创建AVFrame接收解码器解码出来的原始数据(视频的画面帧或者音频的PCM裸流)
    // 在不需要的时候可以通过av_frame_free释放
    mFrame = av_frame_alloc();
    if(NULL == mFrame) {
        cout << "can't alloc frame" << endl;
        return false;
    }

    // 可以从解码器上下文获取视频的尺寸
    // 这个尺寸实际上是从AVCodecParameters里面复制过去的,所以直接用codecParam->width、codecParam->height也可以
    mVideoWidth = mCodecContext->width;
    mVideoHegiht =  mCodecContext->height;

    // 可以从解码器上下文获取视频的像素格式
    // 这个像素格式实际上是从AVCodecParameters里面复制过去的,所以直接用codecParam->format也可以
    mPixelFormat = mCodecContext->pix_fmt;

    return true;
}
```

我们使用VideoDecoder::Load打开视频流并准备好解码器。之后就是解码的过程,解码完成之后再调用VideoDecoder::Release去释放资源:

```c++
void VideoDecoder::Release() {
    mUrl = "";
    mVideoStreamIndex = -1;
    mVideoWidth = -1;
    mVideoHegiht = -1;
    mDecodecStart = -1;
    mLastDecodecTime = -1;
    mPixelFormat = AV_PIX_FMT_NONE;

    if(NULL != mFormatContext) {
        avformat_close_input(&mFormatContext);
    }

    if (NULL != mCodecContext) {
        avcodec_free_context(&mCodecContext);
    }
    
    if(NULL != mPacket) {
        av_packet_free(&mPacket);
    }

    if(NULL != mFrame) {
        av_frame_free(&mFrame);
    }
}
```

# 视频解码

解码器创建完成之后就可以开始解码了:

```c++
AVFrame* VideoDecoder::NextFrame() {
    if(av_read_frame(mFormatContext, mPacket) < 0) {
        return NULL;
    }

    AVFrame* frame = NULL;
    if(mPacket->stream_index == mVideoStreamIndex
        && avcodec_send_packet(mCodecContext, mPacket) == 0
        && avcodec_receive_frame(mCodecContext, mFrame) == 0) {
        frame = mFrame;

        ... //1.解码速度问题
    }

    av_packet_unref(mPacket); // 2.内存泄漏问题

    if(frame == NULL) {
        return NextFrame(); // 3.AVPacket帧类型问题
    }

    return frame;
}
```

它的核心逻辑其实就是下面这三步:
1. 使用av\_read\_frame 从视频流读取视频数据包
2. 使用avcodec\_send\_packet 发送视频数据包给解码器解码
3. 使用avcodec\_receive\_frame 从解码器读取解码后的帧数据

除了关键的三个步骤之外还有些细节需要注意:

### 1.解码速度问题

由于解码的速度比较快,我们可以等到需要播放的时候再去解码下一帧。这样可以降低cpu的占用,也能减少绘制线程堆积画面队列造成内存占用过高.

由于这个demo没有单独的解码线程,在渲染线程进行解码,sdl渲染本身就耗时,所以就算不延迟也会发现画面是正常速度播放的.可以将绘制的代码注释掉,然后在该方法内加上打印,会发现一下子就解码完整个视频了。

### 2.内存泄漏问题

解码完成之后压缩数据包的数据就不需要了,需要使用av\_packet\_unref将AVPacket释放。

其实AVFrame在使用完成之后也需要使用av\_frame\_unref去释放AVFrame的像画面素数据,但是在avcodec\_receive\_frame内会调用av_frame_unref将上一帧的内存清除,而最后一帧的数据也会在Release的时候被av\_frame\_free清除,所以我们不需要手动调用av\_frame\_unref.

### 3.AVPacket帧类型问题

由于视频压缩帧存在i帧、b帧、p帧这些类型,并不是每种帧都可以直接解码出原始画面,b帧是双向差别帧，也就是说b帧记录的是本帧与前后帧的差别,还需要后面的帧才能解码.

如果这一帧AVPacket没有解码出数据来的话,就递归调用NextFrame解码下一帧,直到解出下一帧原生画面来

## PTS同步

AVFrame有个pts的成员变量,代表了画面在什么时候应该显示.由于视频的解码速度通常会很快,例如一个1分钟的视频可能一秒钟就解码完成了.所以我们需要计算出这一帧应该在什么时候播放,如果时间还没有到就添加延迟。

有些视频流不带pts数据,按30fps将每帧间隔统一成32ms:

```c++
if(AV_NOPTS_VALUE == mFrame->pts) {
    int64_t sleep = 32000 - (av_gettime() - mLastDecodecTime);
    if(mLastDecodecTime != -1 && sleep > 0) {
        av_usleep(sleep);
    }
    mLastDecodecTime = av_gettime();
} else {
    ...
}
```

如果视频流带pts数据,我们需要计算这个pts具体是视频的第几微秒.

pts的单位可以通过AVFormatContext找到对应的AVStream,然后再获取AVStream的time\_base得到:

```c++
AVRational timebase = mFormatContext->streams[mPacket->stream_index]->time_base;
```

AVRational是个分数,代表几分之几秒:

```c++
/**
 * Rational number (pair of numerator and denominator).
 */
typedef struct AVRational{
    int num; ///< Numerator
    int den; ///< Denominator
} AVRational;
```

我们用timebase.num * 1.0f / timebase.den计算出这个分数的值,然后乘以1000等到ms,再乘以1000得到us.后半部分的计算其实可以放到VideoDecoder::Load里面保存到成员变量,但是为了讲解方便就放在这里了:

```c++
int64_t pts = mFrame->pts * 1000 * 1000 * timebase.num * 1.0f / timebase.den;
```

这个pts都是以视频开头开始计算的,所以我们需要先保存第一帧的时间戳,然后再去计算当前播到第几微秒.完整代码如下:

```c++
if(AV_NOPTS_VALUE == mFrame->pts) {
    ...
} else {
    AVRational timebase = mFormatContext->streams[mPacket->stream_index]->time_base;
    int64_t pts = mFrame->pts * 1000 * 1000 * timebase.num * 1.0f / timebase.den;

    // 如果是第一帧就记录开始时间
    if(mFrame->pts == 0) {
        mDecodecStart = av_gettime() - pts;
    }

    // 当前时间减去开始时间,得到当前播放到了视频的第几微秒
    int64_t now = av_gettime() - mDecodecStart;

    // 如果这一帧的播放时间还没有到就等到播放时间到了再返回
    if(pts > now) {
        av_usleep(pts - now);
    }
}
```

# 其他

完整的[Demo](https://github.com/bluesky466/FFmpegDemo)已经放到Github上,图像渲染的部分在SdlWindow类中,它使用SDL2去做ui绘制,由于和音视频编解码没有关系就不展开讲了.视频解码部分在VideoDecoder类中.

编译的时候需要修改Makefile里面ffmpeg和sdl2的路径,然后make编译完成之后用下面命令即可播放视频:

> demo -p 视频路径播放视频

PS: 

某些函数会有数字后缀,如avcodec_alloc_context3、avcodec_open2等，实际上这个数字后缀是这个函数的第几个版本的意思,从源码的doc/APIchanges可以看出来:

```
2011-07-10 - 3602ad7 / 0b950fe - lavc 53.8.0
  Add avcodec_open2(), deprecate avcodec_open().
  NOTE: this was backported to 0.7

  Add avcodec_alloc_context3. Deprecate avcodec_alloc_context() and
  avcodec_alloc_context2().
```