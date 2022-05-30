title: 安卓录制MP3(二) - LAME使用
date: 2022-05-30 20:15:49
tags:
  - 技术相关
  - Android
  - 音视频
---

[上篇文章](https://blog.islinjw.cn/2022/05/26/%E5%AE%89%E5%8D%93%E5%BD%95%E5%88%B6MP3-%E4%B8%80-%E6%95%B0%E5%AD%97%E9%9F%B3%E9%A2%91%E5%9F%BA%E7%A1%80/)介绍了数字音频的基础知识,这篇文章我们来看看代码应该怎么写:

# 录音PCM

第一步我们先用AudioRecord录制PCM音频:

```
private lateinit var buffer: ByteArray

fun start(audioSource: Int, sampleRate: Int, channelConfig: Int, audioFormat: Int): Boolean {
    ...
    val bufferSize = AudioRecord.getMinBufferSize(sampleRate, channelConfig, audioFormat)

    buffer = ByteArray(bufferSize)
    recorder = AudioRecord(audioSource, sampleRate, channelConfig, audioFormat, bufferSize)

    recorderThread = RecordThread()
    recorderThread?.start()
    return true
}

inner class RecordThread : Thread() {
    override fun run() {
        super.run()

        var readLen: Int
        recorder?.startRecording()
        while (recorder?.recordingState == AudioRecord.RECORDSTATE_RECORDING) {
            readLen = recorder?.read(buffer, 0, buffer.size) ?: break
            listener.onRecord(buffer, readLen)
        }
        recorder = null
    }
}
```

可以看到AudioRecord的使用很简单，就是创建一个AudioRecord对象然后在子线程中不断地read就好,audioSource指的是通过哪个设备录音,其他的几个参数在基础部分都有讲过,就不过多介绍了:

```
private const val AUDIO_SOURCE = MediaRecorder.AudioSource.MIC
private const val SAMPLE_RATE = 44100
private const val AUDIO_FORMAT = AudioFormat.ENCODING_PCM_16BIT
private const val CHANNEL_CONFIG = AudioFormat.CHANNEL_IN_MONO // 单通道
//private const val CHANNEL_CONFIG = AudioFormat.CHANNEL_IN_STEREO // 双通道
recorder.start(AUDIO_SOURCE, SAMPLE_RATE, CHANNEL_CONFIG, AUDIO_FORMAT)
```

当然,记得需要申请录音权限:

```
<uses-permission android:name="android.permission.RECORD_AUDIO" />
```

# 使用LAME进行MP3编码

得到PCM裸流之后我们就需要对他进行MP3编码,LAME就是一个开源的MP3音频压缩软件,也是公认有损MP3中压缩效果最好的编码器。

## 编译

使用LAME需要自行编译出静态库,首先我们从LAME的[官网](https://lame.sourceforge.io/download.php)下载最新版本的LAME源码。从源码文件目录结构中可以看出它使用AutoMake编译,之前写过几篇AutoMake的[学习笔记](https://blog.islinjw.cn/2017/03/17/automake%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-helloworld/),感兴趣的同学可以看看。

或者也可以直接修改Demo工程的[build-lame.sh](https://github.com/bluesky466/AndroidLameDemo/blob/master/lame/lame-3.100/build-lame.sh)。修改开头的NDK路径、安卓版本、abi这几个参数接口直接运行在output目录生成出.a文件和.h文件:

```
#!/bin/zsh

NDK_ROOT=/Users/linjw/Library/Android/sdk/ndk/22.1.7171670
ANDROID_API_VERSION=21
NDK_TOOLCHAIN_ABI_VERSION=4.9
ABIS=(armeabi armeabi-v7a arm64-v8a x86 x86_64)
...
```

## 导入AndroidStudio

得到静态库之后在CMake中导入静态库:

```
cmake_minimum_required(VERSION 3.18.1)


project("lame")

add_library(
        lame
        SHARED
        native-lib.cpp)

# 导入lame头文件
include_directories(${CMAKE_SOURCE_DIR}/include)

#导入lame静态库
add_library(mp3lame STATIC IMPORTED)
set_target_properties(
        mp3lame
        PROPERTIES IMPORTED_LOCATION
        ${CMAKE_SOURCE_DIR}/../../../jniLibs/${ANDROID_ABI}/libmp3lame.a)

find_library(
        log-lib
        log)

target_link_libraries(
        lame
        mp3lame # 链接lame静态库
        ${log-lib})
```

# lame基本使用

## 初始化

在了解了MP3的基础知识之后其实很容易上手lame。

第一步无非就是一些音频参数的初始化设置:
```
第一步: 创建lame_global_flags*
lame_global_flags *client = lame_init();

第二步: 配置参数
lame_set_in_samplerate(client, 44100);    // 输入采样率
lame_set_out_samplerate(client, 44100);   // 输出采样率44100
lame_set_num_channels(client, 2);         // 通道数2
lame_set_brate(client, 128);              // 比特率128kbps
lame_set_quality(client, 2);              // 编码质量2
lame_init_params(client);                 // 初始化参数
```

### 编码质量

上面的参数大部分都在前一篇介绍过了,只剩一个编码质量没讲过,他其实是用来选择压缩算法的。算法编号从0到9,0音质最好但是算法最复杂也最耗时,9音质最差但算法效率最高。一般会选择2。

比特率和压缩算法都能决定MP3的音质。压缩前的PCM裸流音质最好，大小最大。

由于MP3的大小等于比特率乘时长，压缩后的MP3比特率越大，那么它的大小就越大，就越接近压缩前的PCM，损失的信息也就越少，解压之后就越接近原本的PCM的曲线，音质自然就越好。

而选择压缩算法的意义在于，在相同比特率的情况下( MP3文件大小相同)，好的压缩算法解压出来的声音曲线能更接近原本的PCM曲线，但是相应的它的计算量就会更大。

可以说选择大的比特率是用存储空间换音质，而选择音质高的压缩算法则是用cpu资源换音质。


## MP3编码

配置好初始化参数之后就只要将PCM数据传递给LAME进行压缩即可,这里有几种PCM的传递方式:

**1. 左右通道分开输入**

双通道立体声,有时候左右通道的数据会放在不同的文件里面,可以使用下面的方法去编码MP3:

```
int CDECL lame_encode_buffer (
        lame_global_flags*  gfp,           /* global context handle         */
        const short int     buffer_l [],   /* PCM data for left channel     */
        const short int     buffer_r [],   /* PCM data for right channel    */
        const int           nsamples,      /* number of samples per channel */
        unsigned char*      mp3buf,        /* pointer to encoded MP3 stream */
        const int           mp3buf_size ); /* number of valid octets in this stream */
```

- gfp : lame\_init返回的lame_global_flags*
- buffer\_l : 左通道数据
- buffer\_r : 右通道数据
- nsamples : 每个通道的采样点数量。例如当AudioFormat为16bit的时候, 一个采样点大小为2byte, 则nsamples = buffer\_l有效数据长度(byte) / 2
- mp3buf : 编码后的MP3会存放到这里
- mp3buf\_size : mp3buf的大小,它可以通过采样率、比特率和前面的nsamples计算出来,不过LAME提供了一个最大buffer大小计算的简单算法(1.25*nsamples + 7200 byte)


他的返回值如下:

- \>=0 : 编码后的MP3数据的大小,即存储到mp3buf的MP3音频的byte数
- -1 : mp3buf的大小太小
- -2 : malloc()方法出现问题
- -3 : lame\_init\_params()没有调用
- -4 : 音频数据异常

参考代码:

```
int encodeSize = lame_encode_buffer_interleaved(client, pcm, numSamples, result, resultSize);
```

**2. 单通道输入**

有时候我们的数据只会有一个通道的数据,例如AudioRecord的ChannelConfig配置成AudioFormat.CHANNEL\_IN\_MONO。

这个时候我们只需要把right输入设置成NULL即可:

```
int encodeSize = lame_encode_buffer(client, left, null, numSamples, result, resultSize);
```

**3. 多通道混合输入**

当AudioRecord的ChannelConfig配置成AudioFormat.CHANNEL\_IN\_STEREO。左右声道的数据会交叉存储在PCM裸流中。或者其他更多通道混合输入的时候我们可以用lame_encode_buffer_interleaved方法进行编码:

```
int CDECL lame_encode_buffer_interleaved(
        lame_global_flags*  gfp,           /* global context  handlei */
        short int           pcm[],         /* PCM data for left and right channel, interleaved */
        int                 num_samples,   /* number of samples per channel,  _not_ number of samples in pcm [] */
        unsigned char*      mp3buf,        /* pointer to encoded MP3 stream */
        int                 mp3buf_size ); /* number of valid octets in this stream */ 
```

指的注意的是,这里的num\_samples指的是每个通道的采样点数,而不是所有通道的采样点数之和。

例如当AudioFormat为16bit、双通道输入的时候, 一个采样点大小为2byte, 则

nsamples = buffer\_l有效数据长度(byte) / 2(16bite为2byte) / 2(通道数为2)

```
int encodeSize = lame_encode_buffer_interleaved(client, pcm, numSamples, result, resultSize);
```

## 清理工作

当编码结束之后我们可以用lame\_close方法进行清理:

```
lame_close(client);
```

# 结合JNI进行使用

由于LAEM的使用需要保存lame\_global\_flags*,我们可以将它强转成long传到java进行保存,在需要的时候再讲传入的long强转回lame\_global\_flags*:

```
class LameMp3 {
    private var lameClientPtr: Long = createLameMp3Client()

    fun destroy(): Int {
        val ret = close(lameClientPtr)
        lameClientPtr = 0
        return ret
    }

    ...
    private external fun createLameMp3Client(): Long
    private external fun close(client: Long): Int
    ...
}

extern "C" JNIEXPORT jlong JNICALL
Java_me_linjw_lib_lame_LameMp3_createLameMp3Client(
        JNIEnv *env,
        jobject thiz) {
    lame_global_flags *client = lame_init();
    LOGD("createLameMp3Client: %ld", (long) client);
    return reinterpret_cast<jlong>(client);
}


extern "C" JNIEXPORT int JNICALL
Java_me_linjw_lib_lame_LameMp3_close(
        JNIEnv *env,
        jobject thiz,
        jlong clientPtr) {
    lame_global_flags *client = reinterpret_cast<lame_global_flags *>(clientPtr);
    LOGD("close(%ld)", client);
    return lame_close(client);
}
```

详细代码参考[LameMp3.kt](https://github.com/bluesky466/AndroidLameDemo/blob/master/lame/src/main/java/me/linjw/lib/lame/LameMp3.kt)和[native-lib.cpp](https://github.com/bluesky466/AndroidLameDemo/blob/master/lame/src/main/cpp/native-lib.cpp)

# DEMO代码

DEMO代码已经放到[github](https://github.com/bluesky466/AndroidLameDemo)上