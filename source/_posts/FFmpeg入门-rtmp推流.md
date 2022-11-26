title: FFmpeg入门 - rtmp推流
date: 2022-09-08 21:42:44
tags:
  - 技术相关
  - 音视频
---

系列文章:


1. [FFmpeg入门 - 视频播放](https://blog.islinjw.cn/2022/09/04/FFmpeg%E5%85%A5%E9%97%A8-%E8%A7%86%E9%A2%91%E6%92%AD%E6%94%BE/)
2. [FFmpeg入门 - rtmp推流](https://blog.islinjw.cn/2022/09/08/FFmpeg%E5%85%A5%E9%97%A8-rtmp%E6%8E%A8%E6%B5%81/)
3. [FFmpeg入门 - Android移植](https://blog.islinjw.cn/2022/10/25/FFmpeg%E5%85%A5%E9%97%A8-Android%E7%A7%BB%E6%A4%8D/)
4. [FFmpeg入门 - 格式转换](https://blog.islinjw.cn/2022/11/27/FFmpeg%E5%85%A5%E9%97%A8-%E6%A0%BC%E5%BC%8F%E8%BD%AC%E6%8D%A2/)

[上一篇博客](https://blog.islinjw.cn/2022/09/04/FFmpeg%E5%85%A5%E9%97%A8-%E8%A7%86%E9%A2%91%E6%92%AD%E6%94%BE/)介绍了怎样用ffmpeg去播放视频.

里面用于打开视频流的avformat\_open\_input函数除了打开本地视频之外,实际上也能打开rtmp协议的远程视频,实现拉流:

```
./demo -p 本地视频路径

./demo -p rtmp://服务器ip/视频流路径
```

这篇文章我们来讲下怎样实现推流,然后和之前的demo代码配合就能完成推流、拉流的整个过程,实现直播。

# rtmp服务器

整个直播的功能分成下面三个模块:


{% plantuml %}

cloud {
  [服务器]
}

[推流端] -> [服务器]
[服务器] -> [拉流端]

{% endplantuml %}

从上图我们可以看到rtmp是需要服务器做转发的,我们选用开源的[srs](https://github.com/ossrs/srs).直接从github上把它的源码拉下来编译,然后直接启动即可:

```
git clone git@github.com:ossrs/srs.git
cd srs/trunk
./configure
make
./etc/init.d/srs start
```

如果是本地的电脑,这个时候就能在局域网内直接用它的内网ip去访问了.但如果是腾讯云、阿里云之类的云服务器还需要配置安全组开放下面几个端口的访问权限:

```
listen              1935;
max_connections     1000;
#srs_log_tank        file;
#srs_log_file        ./objs/srs.log;
daemon              on;
http_api {
    enabled         on;
    listen          1985;
}
http_server {
    enabled         on;
    listen          8080;
    dir             ./objs/nginx/html;
}
rtc_server {
    enabled on;
    listen 8000; # UDP port
    # @see https://ossrs.net/lts/zh-cn/docs/v4/doc/webrtc#config-candidate
    candidate $CANDIDATE;
}
...
```

当然如果这几个端口已经被占用的话可以修改配置文件conf/srs.conf去修改

服务器到这里就准备好了,浏览器访问下面网址对srs进行调试、配置:

> http://服务器ip:8080/players/rtc_publisher.html
> http://服务器ip:1985/console/ng_index.html

# 推流

## 准备输出流

我们选择推送本地的视频到rtmp服务器,所以第一步仍然是打开本地视频流:

```c++
bool VideoSender::Send(const string& srcUrl, const string& destUrl) {
    ...
    // 打开文件流读取文件头解析出视频信息如轨道信息、时长等
    // mFormatContext初始化为NULL,如果打开成功,它会被设置成非NULL的值
    // 这个方法实际可以打开多种来源的数据,url可以是本地路径、rtmp地址等
    // 在不需要的时候通过avformat_close_input关闭文件流
    if(avformat_open_input(&inputFormatContext, srcUrl.c_str(), NULL, NULL) < 0) {
        cout << "open " << srcUrl << " failed" << endl;
        break;
    }

    // 对于没有文件头的格式如MPEG或者H264裸流等,可以通过这个函数解析前几帧得到视频的信息
    if(avformat_find_stream_info(inputFormatContext, NULL) < 0) {
        cout << "can't find stream info in " << srcUrl << endl;
        break;
    }

    // 打印输入视频信息
    av_dump_format(inputFormatContext, 0, srcUrl.c_str(), 0);
    ...
}
```

本地视频打开之后,我们创建输出视频流上下文,然后为输出流创建轨道,最后打开输出视频流:

```c++
// 创建输出流上下文,outputFormatContext初始化为NULL,如果打开成功,它会被设置成非NULL的值,在不需要的时候使用avformat_free_context释放
// 输出流使用flv格式
if(avformat_alloc_output_context2(&outputFormatContext, NULL, "flv", destUrl.c_str()) < 0) {
    cout << "can't alloc output context for " << destUrl << endl;
    break;
}

// 拷贝编解码参数
if(!createOutputStreams(inputFormatContext, outputFormatContext)) {
    break;
}

// 打印输出视频信息
av_dump_format(outputFormatContext, 0, destUrl.c_str(), 1);

// 打开输出流,结束的时候使用avio_close关闭
if(avio_open(&outputFormatContext->pb, destUrl.c_str(), AVIO_FLAG_WRITE) < 0) {
    cout << "can't open avio " << destUrl << endl;
    break;
}
```

这里有个createOutputStreams用于根据本地视频文件的轨道信息,为输出流创建同样的轨道:

```c++
static bool createOutputStreams(AVFormatContext* inputFormatContext, AVFormatContext* outputFormatContext) {
    // 遍历输入流的所有轨道,拷贝编解码参数到输出流
    for(int i = 0 ; i < inputFormatContext->nb_streams ; i++) {
        // 为输出流创建轨道
        AVStream* stream = avformat_new_stream(outputFormatContext, NULL);
        if(NULL == stream) {
            cout << "can't create stream, index " << i << endl;
            return false;
        }

        // 编解码参数在AVCodecParameters中保存,从输入流拷贝到输出流
        if(avcodec_parameters_copy(stream->codecpar, inputFormatContext->streams[i]->codecpar) < 0) {
            cout << "can't copy codec paramters, stream index " << i << endl;
            return false;
        }

        // codec_tag代表了音视频数据采用的码流格式,不同的封装格式如flv、mp4等的支持情况是不一样的
        // 上面的avcodec_parameters_copy将输出流的codec_tag从输入拷贝过来变成了一样的
        // 由于我们输出流在avformat_alloc_output_context2的时候写死了flv格式
        // 如果输入流不是flv而是mp4等格式的话就可能会出现mp4里某种codec_tag在flv不支持导致推流失败的情况
        // 这里我们可以用av_codec_get_id从输出流的oformat的支持的codec_tag列表里面查找codec_id
        // 如果和codecpar的codec_id不一致的话代表不支持
        if(av_codec_get_id(outputFormatContext->oformat->codec_tag, stream->codecpar->codec_tag) != stream->codecpar->codec_id) {
            // 这里将codec_tag设置为0,FFmpeg会根据编码codec_id从封装格式的codec_tag列表中找到一个codec_tag
            stream->codecpar->codec_tag = 0;
        }
    }
    return true;
}
```

## codec\_id和codec\_tag

这里可以看到对于编码器有codec\_id和codec\_tag两个字段去描述,codec\_id代表的是数据的编码类型.而codec\_tag用于更详细的描述编解码的格式信息,它对应的是FourCC(Four-Character Codes)数据。

例如codec\_id都是AV\_CODEC\_ID\_RAWVIDEO的裸数据,但它可能是YUV的裸数据也可能是RGB的裸数据:

```c++
// libavformat/isom.c
{ AV_CODEC_ID_RAWVIDEO, MKTAG('r', 'a', 'w', ' ') }, /* uncompressed RGB */
{ AV_CODEC_ID_RAWVIDEO, MKTAG('y', 'u', 'v', '2') }, /* uncompressed YUV422 */
{ AV_CODEC_ID_RAWVIDEO, MKTAG('2', 'v', 'u', 'y') }, /* uncompressed 8-bit 4:2:2 */
{ AV_CODEC_ID_RAWVIDEO, MKTAG('y', 'u', 'v', 's') }, /* same as 2VUY but byte-swapped */
```

又例如codec\_id都是AV\_CODEC\_ID\_H264,但实际上也有许多细分类型:

```c++
// libavformat/isom.c
{ AV_CODEC_ID_H264, MKTAG('a', 'v', 'c', '1') }, /* AVC-1/H.264 */
{ AV_CODEC_ID_H264, MKTAG('a', 'v', 'c', '2') },
{ AV_CODEC_ID_H264, MKTAG('a', 'v', 'c', '3') },
{ AV_CODEC_ID_H264, MKTAG('a', 'v', 'c', '4') },
{ AV_CODEC_ID_H264, MKTAG('a', 'i', '5', 'p') }, /* AVC-Intra  50M 720p24/30/60 */
{ AV_CODEC_ID_H264, MKTAG('a', 'i', '5', 'q') }, /* AVC-Intra  50M 720p25/50 */
{ AV_CODEC_ID_H264, MKTAG('a', 'i', '5', '2') }, /* AVC-Intra  50M 1080p25/50 */
{ AV_CODEC_ID_H264, MKTAG('a', 'i', '5', '3') }, /* AVC-Intra  50M 1080p24/30/60 */
{ AV_CODEC_ID_H264, MKTAG('a', 'i', '5', '5') }, /* AVC-Intra  50M 1080i50 */
{ AV_CODEC_ID_H264, MKTAG('a', 'i', '5', '6') }, /* AVC-Intra  50M 1080i60 */
{ AV_CODEC_ID_H264, MKTAG('a', 'i', '1', 'p') }, /* AVC-Intra 100M 720p24/30/60 */
{ AV_CODEC_ID_H264, MKTAG('a', 'i', '1', 'q') }, /* AVC-Intra 100M 720p25/50 */
{ AV_CODEC_ID_H264, MKTAG('a', 'i', '1', '2') }, /* AVC-Intra 100M 1080p25/50 */
```

可以看出来codec\_tag是通过4个字母去表示的,我们来看看MKTAG的定义:

```c++
#define MKTAG(a,b,c,d) ((a) | ((b) << 8) | ((c) << 16) | ((unsigned)(d) << 24))
```

最终它得到的是一个整数,例如MKTAG('a', 'v', 'c', '1')得到的值是0x31637661

- 0x31 =1
- 0x63 = c
- 0x76 = v
- 0x61 = a

我们可以用av\_fourcc2str这个函数将最终的整数转换回字符串

回过头来看看这个判断:

```c++
if(av_codec_get_id(outputFormatContext->oformat->codec_tag, stream->codecpar->codec_tag) != stream->codecpar->codec_id)
```

大部分情况下如果codec\_tag在输出流不支持的情况下av\_codec\_get\_id拿到的是AV\_CODEC\_ID\_NONE,所以大部分情况可以等价于:

```c++
if(av_codec_get_id(outputFormatContext->oformat->codec_tag, stream->codecpar->codec_tag) != AV_CODEC_ID_NONE)
```

不过也存在都是MKTAG('l', 'p', 'c', 'm'),但codec\_id可能是AV\_CODEC\_ID\_PCM\_S16BE或者AV\_CODEC\_ID\_PCM\_S16LE的情况:

```
{ AV_CODEC_ID_PCM_S16BE,       MKTAG('l', 'p', 'c', 'm') },
{ AV_CODEC_ID_PCM_S16LE,       MKTAG('l', 'p', 'c', 'm') },
```

所以最好还是和原本的codec\_id做比较会靠谱点。

## 写入视频数据

接着就是视频数据的写入了,主要有三个步骤,写入文件头、读取本地视频包并写入输出视频流、写入文件结尾:

```c++
// 设置flvflags为no_duration_filesize用于解决下面的报错
// [flv @ 0x14f808e00] Failed to update header with correct duration.
// [flv @ 0x14f808e00] Failed to update header with correct filesize
AVDictionary * opts = NULL;
av_dict_set(&opts, "flvflags", "no_duration_filesize", 0);
if(avformat_write_header(outputFormatContext, opts ? &opts : NULL) < 0) {
    cout << "write header to " << destUrl << " failed" << endl;
    break;
}

// 创建创建AVPacket接收数据包
// 无论是压缩的音频流还是压缩的视频流,都是由一个个数据包组成的
// 解码的过程实际就是从文件流中读取一个个数据包传给解码器去解码
// 对于视频，它通常应包含一个压缩帧
// 对于音频，它可能是一段压缩音频、包含多个压缩帧
// 在不需要的时候可以通过av_packet_free释放
packet = av_packet_alloc();
if(NULL == packet) {
    cout << "can't alloc packet" << endl;
    break;
}

...

// 从文件流里面读取出数据包,这里的数据包是编解码层的压缩数据
while(av_read_frame(inputFormatContext, packet) >= 0) {
    // 我们以视频轨道为基准去同步时间
    // 如果时间还没有到就添加延迟,避免向服务器推流速度过快
    ...

    // 往输出流写入数据
    av_interleaved_write_frame(outputFormatContext, packet);

    // 写入成之后压缩数据包的数据就不需要了,将它释放
    av_packet_unref(packet);
}

// 写入视频尾部信息
av_write_trailer(outputFormatContext);
```

## 帧同步

由于av\_read\_frame这里读取出来的是未解码的压缩数据速度很快,如果不做控制一下子就发送完成了,会造成数据堆积在服务器上。这里我们忽略网络传输耗时,依然通过视频包的pts做一定的同步:

```c++
while(av_read_frame(inputFormatContext, packet) >= 0) {
    // 我们以视频轨道为基准去同步时间
    // 如果时间还没有到就添加延迟,避免向服务器推流速度过快
    if(videoStreamIndex == packet->stream_index) {
        if(AV_NOPTS_VALUE == packet->pts) {
            // 有些视频流不带pts数据,按30fps将间隔统一成32ms
            av_usleep(32000);
        } else {
            // 带pts数据的视频流,我们计算出每一帧应该在什么时候播放
            int64_t nowTime = av_gettime() - startTime;
            int64_t pts = packet->pts * 1000 * 1000 * timeBaseFloat;
            if(pts > nowTime) {
                av_usleep(pts - nowTime);
            }
        }
    }
    // 往输出流写入数据
    av_interleaved_write_frame(outputFormatContext, packet);

    // 写入成之后压缩数据包的数据就不需要了,将它释放
    av_packet_unref(packet);
}
```

## 资源释放

等视频流读写完成之后就是最后的资源释放收尾工作了:

```c++
if(NULL != packet) {
    av_packet_free(&packet);
}

if(NULL != outputFormatContext) {
    if(NULL != outputFormatContext->pb) {
        avio_close(outputFormatContext->pb);
    }
    avformat_free_context(outputFormatContext);
}

if(NULL != inputFormatContext) {
    avformat_close_input(&inputFormatContext);
}
```

# 其他

[源码](https://github.com/bluesky466/FFmpegDemo/blob/main/video_sender.cpp)和上篇博客的是同一个[仓库](https://github.com/bluesky466/FFmpegDemo),编译之后可以通过-s参数推流到服务器:

> ./demo -s video.flv rtmp://服务器ip/live/livestream

推流的同时就能使用-p参数去拉流进行实时播放:

> ./demo -p rtmp://服务器ip/live/livestream

这个demo只是简单的将本地视频文件推到服务器,实际上我们可以对他做些修改就能实现将摄像头的视频流推到服务器了。