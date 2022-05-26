title: 安卓录制MP3(一) - 数字音频基础
date: 2022-05-26 21:22:13
tags:
  - 技术相关
  - Android
---

在安卓上实现录音功能有两种方式使用AudioRecord录制PCM裸流,或者用MediaRecorder录制编码后的音频流。但是MediaRecorder的编码格式并不包括MP3格式:

```
/**
 * Defines the audio encoding. These constants are used with
 * {@link MediaRecorder#setAudioEncoder(int)}.
 */
public final class AudioEncoder {
    /* Do not change these values without updating their counterparts
     * in include/media/mediarecorder.h!
     */
    private AudioEncoder() {}
    public static final int DEFAULT = 0;
    /** AMR (Narrowband) audio codec */
    public static final int AMR_NB = 1;
    /** AMR (Wideband) audio codec */
    public static final int AMR_WB = 2;
    /** AAC Low Complexity (AAC-LC) audio codec */
    public static final int AAC = 3;
    /** High Efficiency AAC (HE-AAC) audio codec */
    public static final int HE_AAC = 4;
    /** Enhanced Low Delay AAC (AAC-ELD) audio codec */
    public static final int AAC_ELD = 5;
    /** Ogg Vorbis audio codec (Support is optional) */
    public static final int VORBIS = 6;
    /** Opus audio codec */
    public static final int OPUS = 7;
}
```

所以如果想要将录制的音频保存成MP3,就需要我们自行对PCM裸流进行编码。常用的是[LAME](https://lame.sourceforge.io/)库去编码,不过在学习怎样使用LAME之前我们先稍微了解下数字音频的一些基础知识。

# 数字音频基础

我们都知道声音是一种波:

{% img /安卓录制MP3一/1.png %}

它在时间上是连续的,如果我们想把他保存成2进制的音频数据,就只能隔一段时间对这个波形进行一次采样,读取那个时刻声波的具体数值:

{% img /安卓录制MP3一/2.png %}

## 采样率

采样的间隔越短,就越接近原始的音波,录制下来的音质也就越好。一般我们不直接用采样间隔,而是使用采样率来描述。采样率代表一秒钟采样的次数,单位为Hz。例如8000 Hz就代表1秒钟采样8000次。所以采样率越高采样间隔就越短。

常见的采样率有8000Hz、11025Hz、32000Hz 、44100Hz等。根据奈奎斯特定理（也称为采样定理, 根据奈奎斯特定理(也称为采样定理),当采样频率大于信号中最高频率的2倍时,采样之后的数字信号完整地保留了原始信号中的信息。由于人耳能听到的声音频率为20Hz~20000Hz,所以采样率一般会使用44100Hz

## AudioFormat

AudioFormat代表我们可以用8bit、16bit或者一个浮点数去保存每一次采样获取的数字:

```
/** Audio data format: PCM 16 bit per sample. Guaranteed to be supported by devices. */
public static final int ENCODING_PCM_16BIT = 2;

/** Audio data format: PCM 8 bit per sample. Not guaranteed to be supported by devices. */
public static final int ENCODING_PCM_8BIT = 3;

/** Audio data format: single-precision floating-point per sample */
public static final int ENCODING_PCM_FLOAT = 4;
```

## 通道数

由于左右耳朵听到的声音并不完全一样,所以常见的pcm流会有左右两个通道的数据,意味着我们可以从中拆分出两条音波曲线分别播放给左右耳朵:

```
/**
 * 单通道
 */
public static final int CHANNEL_IN_MONO = CHANNEL_IN_FRONT;

/**
 * 双通道
 */
public static final int CHANNEL_IN_STEREO = (CHANNEL_IN_LEFT | CHANNEL_IN_RIGHT);
```

## PCM大小计算

假设我们录制44100Hz、16bit、双通道的音频数据:

- 1秒钟的数据大小为 44100 * 16 * 2 = 1411200 bit = 176400 byte = 172 KB
- 1分钟的数据大小为 172 * 60 = 10320 KB = 10 MB

# MP3编码

从上面的计算我们可以得知,如果直接使用PCM裸流去保存,一首歌大约三到五分钟就有30~50MB。这样的数据量的确过于庞大了，所以我们对其进行压缩编码得到AAC、MP3、Ogg等格式的音频文件。

这篇文章就只介绍MP3格式，在MP3里面有个很重要的概念是bitrate(比特率),它代表的是1秒钟的音频MP3格式需要用多少bit去表示,单位为bps(Bit Per Second)或者kbps(1kbps = 1000bps)。

mp3的比特率默认是128kbps,也就是说这种比特率的MP3文件:

- 1秒钟的数据大小为 128000 bit = 16000 byte = 15 KB
- 1分钟的数据大小为 15 * 60 = 240 KB


## CBR

MP3中常用固定比特率CBR(Constant Bit-Rate)去进行压缩,这种类型的mp3每一帧的比特率都是固定的。

CBR由于每秒的bit数是固定的:

1. 音频时长可以通过文件大小计算 - 时长 = (文件大小 - 文件头大小) / bitrate
2. seek操作容易实现 - 文件位置 = 文件头大小 + 目标时间 * bitrate

但是一般来讲一个声音片段音调越高,代表它的频率越高,数据就越密集,压缩之后的大小也就越大。相反如果一个声音音调低,理论上我们压缩之后他的数据大小会越小。但是由于我们使用了固定的bitrate,那么在低音部分就会有部分的数据冗余。

## VBR

可变比特率VBR(Variable Bit-Rate)就很好的解决了这个这个数据冗余问题,这种类型每一帧的比特率是不固定的,在数据复杂的地方bitrate高,在数据简单的地方bitrate低。它的好处显而易见是可以节约更多的空间。但是也有下面缺点:

1. 音频时长需要额外的字段去保存,或者解析整个MP3文件去计算
2. seek操作复杂,需要保存关键时间点的文件位置,然后进行计算和插值

## ABR

MP3格式还存在一种平均比特率ABR(Average Bit-Rate)的编码，它大多数音频帧以固定bitrate编码,但会在个别的帧使用高bitrate编码。但是通常这些帧一般比较少,所以在文件大小上跟CBR相比没有太大的差异,因此这种类型并不常见。

## MP3帧长度

上面其实也有提到MP3的帧,在不同的规范中每一帧包含的采样点数量是不一样的:

|MPEG 1 | MPEG 2 (LSF) | MPEG 2.5 (LSF)|
|-|-|-|
|Layer I|384|384|384|
|Layer II|1152|1152|1152|
|Layer III|1152|576|576|

以常见的Layer II为例子,假设采样率为44100Hz,一秒钟有44100个采样点,一帧只包含1152个,那么: 

一帧的时长 = 1152 / 44100 = 0.02608s = 26.08ms

这也是MP3一帧26ms的由来