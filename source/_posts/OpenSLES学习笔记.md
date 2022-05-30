title: OpenSL ES 学习笔记
date: 2018-09-01 20:24:42
tags:
    - 技术相关
    - Android
    - 音视频
---

一般来讲在安卓中常使用AudioRecord、MediaRecorder对音频进行采集,使用SoundPool、MediaPlayer、AudioTrack进行音频播放。

但是这些接口都是java层的,而NDK其实也提供了一个叫做OpenSL的C语言引擎用于声音的处理。

OpenSL入门难度比较大,而且网上也没有什么特别好的教程,我这里把自己了解到的一些知识记录下来,希望以后忘记的时候可以快速回忆起来,也希望对大家有用。

这篇笔记的很多内容都参考了OpenSL的官方文档OpenSL\_ES\_Specification\_1.0.1.pdf,它是全英文的,可以在NDK的安装目录下找到,大家可以大概浏览一下,具体路径为:

> $NDK_ROOT/docs/Additional\_library\_docs/opensles

为什么要学OpenSL呢?除了C/C++的性能优势(不过其实java的效率也不低)之外,最主要是因为最近入坑FFmpeg,如果使用java层的接口,还需要通过一层JNI,比较复杂,性能消耗也大。如果用OpenSL的话就能直接在C/C++里面把事情都处理了。

# 基本概念

## Object和Interface

在OpenSL里面,Object和Interface是两个很重要的概念,基本上所有的操作都是通过它们两个去执行的。

Object和Interface是包含关系,一个Object里面包含了多个Interface:

{% img /OpenSLES学习笔记/1.png %}

### Object

Object是一个资源的抽象集合,可以通过它获取各种资源。

例如我们可以通过Object的GetInterface方法获取Interface。

所有的Object在OpenSL里面我们拿到的都是一个SLObjectItf:

```
struct SLObjectItf_ {
	SLresult (*Realize) (SLObjectItf self,SLboolean async);

	SLresult (*Resume) (SLObjectItf self,SLboolean async);

	SLresult (*GetState) (SLObjectItf self,SLuint32 * pState);

	SLresult (*GetInterface) (SLObjectItf self, const SLInterfaceID iid, void * pInterface);

	SLresult (*RegisterCallback) (SLObjectItf self, slObjectCallback callback, void * pContext);

	void (*AbortAsyncOperation) (SLObjectItf self);

	void (*Destroy) (SLObjectItf self);

	SLresult (*SetPriority) (SLObjectItf self, SLint32 priority, SLboolean preemptable);

	SLresult (*GetPriority) (SLObjectItf self, SLint32 *pPriority, SLboolean *pPreemptable);

	SLresult (*SetLossOfControlInterfaces) (SLObjectItf self, SLint16 numInterfaces, SLInterfaceID * pInterfaceIDs, SLboolean enabled);
};

typedef const struct SLObjectItf_ * const * SLObjectItf;
```

在创建出来之后必须先调用Realize方法做初始化。在不需要使用的时候调用Destroy方法释放资源。

#### GetInterface

GetInterface可以说是OpenSL里使用频率最高的方法,通过它我们可以获取Object里面的Interface。

由于一个Object里面可能包含了多个Interface,所以GetInterface方法有个SLInterfaceID参数来指定到的需要获取Object里面的那个Interface。

例如下面代码我们通过EngineObject去获取SL_IID_ENGINE这个id的Interface,而这个id对应的Interface就是SLEngineItf:

```
//create EngineObject
SLObjectItf engineObject;
slCreateEngine(&engineObject, 0, NULL, 0, NULL, NULL);
(*engineObject)->Realize(engineObject, SL_BOOLEAN_FALSE);

//get SLEngineItf
SLEngineItf engineInterface;
(*engineObject)->GetInterface(engineObject, SL_IID_ENGINE, &engineInterface);
```


## Interface

Interface则是方法的集合,例如SLRecordItf里面包含了和录音相关的方法,SLPlayItf包含了和播放相关的方法。我们功能都是通过调用Interfaces的方法去实现的。

#### SLEngineItf

SLEngineItf是OpenSL里面最重要的一个Interface,我们可以通过它去创建各种Object,例如播放器、录音器、混音器的Object,然后在用这些Object去获取各种Interface去实现各种功能。


```
struct SLEngineItf_ {
	SLresult (*CreateAudioPlayer) (SLEngineItf self, SLObjectItf * pPlayer, SLDataSource *pAudioSrc, SLDataSink *pAudioSnk, SLuint32 numInterfaces, const SLInterfaceID * pInterfaceIds, const SLboolean * pInterfaceRequired);

	SLresult (*CreateAudioRecorder) (SLEngineItf self, SLObjectItf * pRecorder, SLDataSource *pAudioSrc, SLDataSink *pAudioSnk, SLuint32 numInterfaces, const SLInterfaceID * pInterfaceIds, const SLboolean * pInterfaceRequired);

	SLresult (*CreateOutputMix) (SLEngineItf self, SLObjectItf * pMix, SLuint32 numInterfaces, const SLInterfaceID * pInterfaceIds, const SLboolean * pInterfaceRequired);

	...
};
```

# 录音

OpenSL的录音功能是通过AudioRecorder来实现的,而AudioRecorder是通过SLEngineItf.CreateAudioRecorder方法创建的:

```
SLresult (*CreateAudioRecorder) (
        SLEngineItf self,
        SLObjectItf * pRecorder,
        SLDataSource * pAudioSrc,
        SLDataSink * pAudioSnk,
        SLuint32 numInterfaces,
        const SLInterfaceID * pInterfaceIds,
        const SLboolean * pInterfaceRequired
    );
```

各个参数的意义如下:

- SLEngineItf C语言不像c++,没有this指针,只能每次调用SLEngineItf的方法的时候手动传入
- SLObjectItf 用于保存创建出来的AudioRecorderObject
- SLDataSource 数据的来源
- SLDataSink 数据的去处
- numInterfaces 与下面的SLInterfaceID和SLboolean配合使用,用于标记SLInterfaceID数组和SLboolean的大小
- SLInterfaceID 这里需要传入一个数组,指定创建的AudioRecorderObject会包含哪些Interface
- SLboolean 这里也是一个数组,用来标记每个需要包含的Interface,如果AudioRecorderObject不支持,是不是需要直接创建AudioRecorderObject失败。

最后的三个参数用于指定AudioRecorderObject需要包含哪些Interface,如果不包含,是不是要直接创建失败。如果成功的话我们就能使用AudioRecorderObject的GetInterface方法获取到这些Interface了。

SLDataSource和SLDataSink可能比较难理解。我们可以看下OpenSL录音的原理:

{% img /OpenSLES学习笔记/2.png %}

简而言之, AudioRecorder会从SLDataSource指定的数据源获取数据,然后将数据保存到SLDataSink指定的接收器。

SLDataSource很明显就是录音设备(SL\_IODEVICE\_AUDIOINPUT):

```
SLDataLocator_IODevice device;
device.locatorType = SL_DATALOCATOR_IODEVICE;
device.deviceType = SL_IODEVICE_AUDIOINPUT;
device.deviceID = SL_DEFAULTDEVICEID_AUDIOINPUT;
device.device = NULL; //Must be NULL if deviceID parameter is to be used.

SLDataSource source;
source.pLocator = &device;
source.pFormat = NULL; //This parameter is ignored if pLocator is SLDataLocator_IODevice.
```

而SLDataSink就可以任由我们指定了,它官方支持下面的类型:

```
SLDataLocator_Address
SLDataLocator_IODevice
SLDataLocator_OutputMix
SLDataLocator_URI
SLDataLocator_BufferQueue
SLDataLocator_MIDIBufferQueue
```

Android又拓展了下面几种类型:

```
SLDataLocator_AndroidFD
SLDataLocator_AndroidBufferQueue
SLDataLocator_AndroidSimpleBufferQueue
```

我这边把它设置成SLDataLocator_AndroidSimpleBufferQueue,它比较通用, AudioRecorder把数据放到这个队列中,我们再可以从这个队列中拿出来使用:

```
SLDataLocator_AndroidSimpleBufferQueue queue;
queue.locatorType = SL_DATALOCATOR_ANDROIDSIMPLEBUFFERQUEUE;
queue.numBuffers = 2;

SLDataFormat_PCM format;
format.formatType = SL_DATAFORMAT_PCM;
format.numChannels = numChannels;
format.samplesPerSec = samplingRate;
format.bitsPerSample = SL_PCMSAMPLEFORMAT_FIXED_16;
format.containerSize = SL_PCMSAMPLEFORMAT_FIXED_16;
format.channelMask = getChannelMask(numChannels);
format.endianness = SL_BYTEORDER_LITTLEENDIAN;

SLDataSink sink;
sink.pLocator = &queue;
sink.pFormat = &format;
```

同时在创建的时候需要检测下SL\_DATALOCATOR\_ANDROIDSIMPLEBUFFERQUEUE是不是支持:

```
SLInterfaceID id[] = {SL_IID_ANDROIDSIMPLEBUFFERQUEUE};
SLboolean required[] = {SL_BOOLEAN_TRUE};

SLObjectItf recorderObject;
(engineInterface)->CreateAudioRecorder(
        engineInterface,
        &(recorderObject),
        &source,
        &sink,
        1,
        id,
        required
);
(*recorderObject)->Realize(recorderObject, SL_BOOLEAN_FALSE);
```

所以我们可以通过GetInterface获取SLAndroidSimpleBufferQueueItf,然后注册个队列满的监听回调:

```
SLAndroidSimpleBufferQueueItf queueInterface;
(*recorderObject)->GetInterface(
        recorderObject,
        SL_IID_ANDROIDSIMPLEBUFFERQUEUE,
        &(queueInterface)
);

(*queueInterface)->RegisterCallback(
        queueInterface,
        bufferQueueCallback,
        NULL
);
```

回调函数如下,我们可以在这个时候从队列里面读取下来的音频数据:

```
static void bufferQueueCallback(SLAndroidSimpleBufferQueueItf queue, void *pContext) {
	...
}
```

最后需要打开录音设备开始录音:

```
SLRecordItf recorderInterface;
(*recorderObject)->GetInterface(
        recorderObject,
        SL_IID_RECORD,
        &(recorderInterface)
);

(*recorderInterface)->SetRecordState(
        recorderInterface,
        SL_RECORDSTATE_RECORDING
);
```

这里需要注意的是我们必须在队列满的时候将数据取出来,如果不取,那队列里面就没有空间可以继续存储音频数据了:

```
(*queueInterface)->Enqueue(queueInterface, buffer, BUFFER_SIZE*sizeof(short));
```

# 播放

播放的代码和录音很类似。我们需要先创建AudioPlayer:

```
SLresult (*CreateAudioPlayer) (
    SLEngineItf self,
    SLObjectItf * pPlayer,
    SLDataSource *pAudioSrc,
    SLDataSink *pAudioSnk,
    SLuint32 numInterfaces,
    const SLInterfaceID * pInterfaceIds,
    const SLboolean * pInterfaceRequired
);
```

它的参数和CreateAudioRecorder一样,我就不再一个个去解释了,可以看看播放的过程:

{% img /OpenSLES学习笔记/3.png %}

SLDataSource我也用SLDataLocator_AndroidSimpleBufferQueue,这样我们可以往队列中不断写入音频数据,AudioRecorder会从队列中不断获取数据传递到混音器中:

```
SLDataLocator_AndroidSimpleBufferQueue queue;
queue.locatorType = SL_DATALOCATOR_ANDROIDSIMPLEBUFFERQUEUE;
queue.numBuffers = 2;

SLDataFormat_PCM format;
format.formatType = SL_DATAFORMAT_PCM;
format.numChannels = numChannels;
format.samplesPerSec = samplingRate;
format.bitsPerSample = SL_PCMSAMPLEFORMAT_FIXED_16;
format.containerSize = SL_PCMSAMPLEFORMAT_FIXED_16;
format.channelMask = getChannelMask(numChannels);
format.endianness = SL_BYTEORDER_LITTLEENDIAN;

SLDataSource source;
source.pLocator = &queue;
source.pFormat = &format;
```

而SLDataSink需要配置成混音器。混音器用于将多个音频混合并且输出到喇叭:

```
SLObjectItf outputMixObject;
(*engineInterface)->CreateOutputMix(
        engineInterface,
        &(outputMixObject),
        0,
        NULL,
        NULL
);
(*outputMixObject)->Realize(
        outputMixObject,
        SL_BOOLEAN_FALSE
);

SLDataLocator_OutputMix outputMix;
outputMix.locatorType = SL_DATALOCATOR_OUTPUTMIX;
outputMix.outputMix = outputMixObject;

SLDataSink sink;
sink.pLocator = &outputMix;
sink.pFormat = NULL; //This parameter is ignored if pLocator is SLDataLocator_IODevice or SLDataLocator_OutputMix.
```

同样的我们在创建AudioPlayer的时候会检查是不是支持SL\_IID_ANDROIDSIMPLEBUFFERQUEUE:

```
SLObjectItf playerObject;
SLInterfaceID id[] = {SL_IID_ANDROIDSIMPLEBUFFERQUEUE};
SLboolean required[] = {SL_BOOLEAN_TRUE};
(*engineInterface)->CreateAudioPlayer(
        engineInterface,
        &(playerObject),
        &source,
        &sink,
        1,
        id,
        required
);
(*playerObject)->Realize(playerObject, SL_BOOLEAN_FALSE);
```

最后我们需要注册队列空的监听和打开播放器开始播放:

```
SLAndroidSimpleBufferQueueItf queueInterface;
(*playerObject)->GetInterface(
        playerObject,
        SL_IID_ANDROIDSIMPLEBUFFERQUEUE,
        &(queueInterface)
);
(*queueInterface)->RegisterCallback(
        queueInterface,
        bufferQueueCallback,
        NULL
);

//////Begin Playing//////
SLPlayItf playInterface;
(*playerObject)->GetInterface(
        playerObject,
        SL_IID_PLAY,
        &(playInterface)
);
(*playInterface)->SetPlayState(
        playInterface,
        SL_PLAYSTATE_PLAYING
);
```

# Demo

这里有个简单的录音和播放的[demo](https://github.com/bluesky466/OpenSLDemo),按兴趣的同学可以参考一下。
