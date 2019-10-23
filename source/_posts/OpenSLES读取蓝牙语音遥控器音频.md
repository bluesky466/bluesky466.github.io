title: OpenSL ES 读取蓝牙语音遥控器音频
date: 2019-10-23 21:42:14
tags:
    - 技术相关
    - Android
    - C/C++
---

最近协助处理一个OpenSL的问题。问题背景和描述如下:

我司的安卓电视需要使用OpenSL从语音遥控器上采集音频,而语音遥控器有两种协议:2.4G和蓝牙。

在2.4G遥控上功能正常,但是使用蓝牙遥控器,没有任何报错,流程也正常跑通,但是读取到的音频数据全部是空数据。

其实OpenSL我之前也只是简单入门而已(笔记在[这里](http://blog.islinjw.cn/2018/09/01/OpenSLES%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/)),首先想到的是去修改它的设备类型

```
pHelper->device.locatorType = SL_DATALOCATOR_IODEVICE;
pHelper->device.deviceType = SL_IODEVICE_AUDIOINPUT;
pHelper->device.deviceID = SL_DEFAULTDEVICEID_AUDIOINPUT;
```

但是查看了下locatorType、deviceType、deviceID可以选择的选项,发现并没有适合的:

```
/** Data locator */
#define SL_DATALOCATOR_URI			((SLuint32) 0x00000001)
#define SL_DATALOCATOR_ADDRESS		((SLuint32) 0x00000002)
#define SL_DATALOCATOR_IODEVICE		((SLuint32) 0x00000003)
#define SL_DATALOCATOR_OUTPUTMIX		((SLuint32) 0x00000004)
#define SL_DATALOCATOR_RESERVED5		((SLuint32) 0x00000005)
#define SL_DATALOCATOR_BUFFERQUEUE	((SLuint32) 0x00000006)
#define SL_DATALOCATOR_MIDIBUFFERQUEUE	((SLuint32) 0x00000007)
#define SL_DATALOCATOR_RESERVED8		((SLuint32) 0x00000008)

/** IODevice-types */
#define SL_IODEVICE_AUDIOINPUT	((SLuint32) 0x00000001)
#define SL_IODEVICE_LEDARRAY	((SLuint32) 0x00000002)
#define SL_IODEVICE_VIBRA		((SLuint32) 0x00000003)
#define SL_IODEVICE_RESERVED4	((SLuint32) 0x00000004)
#define SL_IODEVICE_RESERVED5	((SLuint32) 0x00000005)

/** Device ids */
#define SL_DEFAULTDEVICEID_AUDIOINPUT 	((SLuint32) 0xFFFFFFFF)
#define SL_DEFAULTDEVICEID_AUDIOOUTPUT 	((SLuint32) 0xFFFFFFFE)
#define SL_DEFAULTDEVICEID_LED          ((SLuint32) 0xFFFFFFFD)
#define SL_DEFAULTDEVICEID_VIBRA        ((SLuint32) 0xFFFFFFFC)
#define SL_DEFAULTDEVICEID_RESERVED1    ((SLuint32) 0xFFFFFFFB)
```

其实以前在用java层的AudioRecorder的时候就遇到过获取不到蓝牙语音遥控器音频的问题,当时是通过将audioSource设置成MediaRecorder.AudioSource.CAMCORDER解决的.

所以猜测OpenSL里面大概也是这样处理,在OpenSLES\_AndroidConfiguration.h下找到类似的宏定义:

```
/**   uses the microphone audio source with the same orientation as the camera
 *     if available, the main device microphone otherwise */
#define SL_ANDROID_RECORDING_PRESET_CAMCORDER           ((SLuint32) 0x00000002)
```

这个宏用于指定音频设备的预设值。也就是说需要改变的不是音频设备的类型,依然使用SL\_DEFAULTDEVICEID\_AUDIOINPUT这个设备去读取音频,只不过将它的配置改成SL\_ANDROID\_RECORDING\_PRESET\_CAMCORDER让这个AUDIOINPUT设备读取的是摄像机方向的麦克风数据。

那要怎么修改这个预设值呢?

从[安卓开发者文档](https://developer.android.com/ndk/guides/audio/opensl/android-extensions)找到了播放器的设置方法:

```
// CreateAudioPlayer and specify SL_IID_ANDROIDCONFIGURATION
// in the required interface ID array. Do not realize player yet.
// ...
SLAndroidConfigurationItf playerConfig;
result = (*playerObject)->GetInterface(playerObject,
    SL_IID_ANDROIDCONFIGURATION, &playerConfig);
assert(SL_RESULT_SUCCESS == result);
SLint32 streamType = SL_ANDROID_STREAM_ALARM;
result = (*playerConfig)->SetConfiguration(playerConfig,
    SL_ANDROID_KEY_STREAM_TYPE, &streamType, sizeof(SLint32));
assert(SL_RESULT_SUCCESS == result);
// ...
// Now realize the player here.
```

其实录音的方法也是类似的,通过SLAndroidConfigurationItf的SetConfiguration方法去配置。不过文档其实默认读者都是了解OpenSL的使用方法的了,所以还有些必要的代码其实没有展示出来。

要从playerObject里面用GetInterface获取到SLAndroidConfigurationItf,必须在CreateAudioPlayer的时候就将SLAndroidConfigurationItf的id传入,这样playerObject才会有SLAndroidConfigurationItf这个接口。要不然GetInterface会获取失败。

这里我就直接将录音代码展示出来了,注意看我注释的那两行:

```
SLObjectItf recorderObject;

SLInterfaceID id[] = { 
        SL_IID_ANDROIDSIMPLEBUFFERQUEUE, 
        SL_IID_ANDROIDCONFIGURATION     // 需要指定AudioRecorder包含SLAndroidConfigurationItf
};
SLboolean required[] = {
        SL_BOOLEAN_TRUE, 
        SL_BOOLEAN_TRUE  // 指定SL_IID_ANDROIDCONFIGURATION是必须包含的
};

(*engineInterface)->CreateAudioRecorder(
        engineInterface,
        &(recorderObject),
        &(source),
        &(sink),
        2,
        id,
        required
);

SLAndroidConfigurationItf configItf;
(*recorderObject)->GetInterface(recorderObject, SL_IID_ANDROIDCONFIGURATION, (void*)&configItf);

SLuint32 presetValue = SL_ANDROID_RECORDING_PRESET_CAMCORDER;
(*configItf)->SetConfiguration(configItf, SL_ANDROID_KEY_RECORDING_PRESET, &presetValue, sizeof(SLuint32));

(*recorderObject)->Realize(recorderObject, SL_BOOLEAN_FALSE);
```

顺嘴说一句,如果我们不设置SL\_ANDROID\_RECORDING\_PRESET\_CAMCORDER,它默认是用SL\_ANDROID\_RECORDING\_PRESET\_GENERIC。


所有的配置如下:

```

/**  配置中“无”不能设置，它是用来表示当前设置不匹配任何配置 */
#define SL_ANDROID_RECORDING_PRESET_NONE                ((SLuint32) 0x00000000)

/**  平台上通用的录像配置 */
#define SL_ANDROID_RECORDING_PRESET_GENERIC             ((SLuint32) 0x00000001)

/**  如果有的话使用具有相同的方向与摄像机的麦克风的音频源,否则使用过主麦克风 */
#define SL_ANDROID_RECORDING_PRESET_CAMCORDER           ((SLuint32) 0x00000002)

/**  使用为语音识别优化过的主要麦克风 */
#define SL_ANDROID_RECORDING_PRESET_VOICE_RECOGNITION   ((SLuint32) 0x00000003)

/** 使用为音频通信优化过的主要麦克风 */
#define SL_ANDROID_RECORDING_PRESET_VOICE_COMMUNICATION ((SLuint32) 0x00000004)

/** 使用未经处理的主麦克风 */
#define SL_ANDROID_RECORDING_PRESET_UNPROCESSED         ((SLuint32) 0x00000005)
```

# 其它坑

然后除了上面的SL\_IID\_ANDROIDCONFIGURATION之外,其实我这边还遇到了几个坑,这里也记录一下

- __采样率需要和录音设备硬件一致__

由于代码里面一直用16000Hz的采样率,在2.4G语音遥控器上也工作的好好的,于是适配蓝牙语音遥控之后就没有修改它,然后发现读取直接阻塞住了,读取音频数据的回调一直没有被调用(如果不设置SL\_ANDROID\_RECORDING\_PRESET\_CAMCORDER的时候虽然也拿不到音频数据,但是回调是会调用的,只不过拿到的数据都是空数据)。之后改成了和硬件一致的44100Hz之后就正常了

- __SetConfiguration必须在Realize前面调用__

其实这块并不是我司代码的问题,我们的项目接入了亚马逊的Alexa sdk,它对OpenSL进行了一层封装,它在创建AudioRecorder之后顺手就Realize了,但是它原本也是有提供封装好的SetConfiguration的配置的,于是我们就直接在上面改了。

于是就一直没有效果。最后还是在分析log的时候发现调用SetConfiguration的时候底层输出了“Realize已经调用”的打印才找到问题所在。

这里忍不住吐槽一下,亚马逊写这块的人明显对OpenSL也不是很熟悉,除了这个问题之外我们还发现了不少其他的低级问题。看来亚马逊这种大公司的代码审查也做的不怎么样嘛......