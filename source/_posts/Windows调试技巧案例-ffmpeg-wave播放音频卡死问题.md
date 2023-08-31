title: Windows调试技巧案例-ffmpeg&wave播放音频卡死问题
date: 2023-08-31 20:01:02
tags:
    - 技术相关
    - Windows
---


最近在windows上开发音视频,遇到了读流卡死的问题.定位过程中学到了一些Windows上的程序调试技巧,这里记录一下。

# WinDbg查看线程堆栈

最近的这个项目使用ffmpeg播放外部设备的音视频流,在播放的过程中强行拔掉外部设备和pc的硬件连接,会出现音频流卡死在`av_read_frame`的现象。卡死问题在安卓上可以用`kill -3`命令输出进程堆栈,在Windows上没有类似的命令而是需要使用[WinDbg](https://learn.microsoft.com/zh-cn/windows-hardware/drivers/debugger/)去调试。

可以直接使用它打开exe程序或者attach到已经打开的进程对程序进行调试:

{% img /Windows调试技巧案例-播放音频卡死问题/1.jpg %}


将编译时生成的pdb文件放到exe程序的同级目录为WinDbg提供符号表信息,在问题出现的时候就可以点击Break按钮然后在命令行输入框里面输入`~*k`命令打印出进程的所有线程堆栈,我们只要找到卡死的线程分析其堆栈看具体卡死在什么地方即可。

这里的`~`指线程信息,`*`指所有线程,`k`指打印堆栈,所以`~*k`指打印所有线程的堆栈。类似的可以用`~8k`打印序号为8的线程的堆栈,更多的命令可以查看[官方文档](https://learn.microsoft.com/zh-cn/windows-hardware/drivers/debugger/commands)

从我们这个案例举例,序号为8,Id为1278.1680的线程在audio_player.cpp的第228行调用了`av_read_frame`然后最终卡死在ffmpeg的dshow.c的第1862行:

{% img /Windows调试技巧案例-播放音频卡死问题/2.jpg %}

所以我们打开dshow.c的源码看1862行为什么会卡死:

{% img /Windows调试技巧案例-播放音频卡死问题/3.jpg %}

可以看到1862行是在`dshow_read_packet`的while循环,我一开始以为是卡在了等待mutex锁上,但是从ffmpeg的源码上看只有另外一个`callback`函数里面会去抢占这个锁:

{% img /Windows调试技巧案例-播放音频卡死问题/4.jpg %}

但是搜索所有的线程堆栈都没有调用到这个`callback`函数的。

我们需要先定位到它是怎么卡死的才能针对性的去分析如何解决问题,既然堆栈里面看不到,那么我们还有下面的手动去继续分析:

1. 在ffmpeg源码里面添加日志打印 - 我们使用的conan去依赖的ffmpeg,修改源码的方式又需要自己编译然后修改依赖配置比较麻烦
2. 使用WinDbg添加断点单步调试 - 使用起来不是很方便
3. 在调试机上部署开发环境,使用Visual Studio添加断点单步调试 - 配置环境的耗时比较大
4. 使用Visual Studio远程调试在开发机上远程单步调试开发机的程序 - 比较合适

# Visual Studio远程调试

综合考虑下来我选择使用Visual Studio远程调试功能去调试。

### 调试机

根据[官方文档](https://learn.microsoft.com/zh-cn/visualstudio/debugger/remote-debugging?view=vs-2022),我们可以从VS的安装目录拷贝`Program Files\Microsoft Visual Studio 17.0\Common7\IDE\Remote Debugger`到调试机上执行msvsmon.exe打开调试服务,而且为了方便我们可以直接配置成无身份认证:

{% img /Windows调试技巧案例-播放音频卡死问题/5.jpg %}

{% img /Windows调试技巧案例-播放音频卡死问题/6.jpg %}


### 开发机

开发机需要和调试机在同一个局域网,然后配置远程调试器:

{% img /Windows调试技巧案例-播放音频卡死问题/7.jpg %}

注意这里的远程命令、工作目录指的都是调试机上的文件路径,而且每次修改编译完成之后VS并不会自动帮我们把新生成的exe文件传输到调试机,所以要么每次重新编译之后手动拷贝exe到开发机,要么弄一个共享目录将编译的目标exe配置生成到共享目录。

完成配置之后就可以在vs上选择远程Windows调试器执行exe,然后像本地调试一样添加断点、查看堆栈、查看变量值等:

{% img /Windows调试技巧案例-播放音频卡死问题/8.jpg %}

### 问题定位

从单步调试的代码执行流程来看,是我之前对堆栈的分析有问题,卡在dshow.c的第1862行并不是卡在下一行等待mutex锁,而是卡在while的末尾等待进入下次while循环,即卡住1879行的WaitForMultipleObjects上。如果细心点的话也能发现,堆栈实际是卡在WaitForMultipleObjects而不是WaitForSingleObject:

{% img /Windows调试技巧案例-播放音频卡死问题/9.jpg %}

这个问题实际其实只涉及到`callback`和`dshow_read_packet`两个函数:

```c
static void
callback(void *priv_data, int index, uint8_t *buf, int buf_size, int64_t time, enum dshowDeviceType devtype)
{
    AVFormatContext *s = priv_data;
    struct dshow_ctx *ctx = s->priv_data;
    PacketListEntry **ppktl, *pktl_next;

//    dump_videohdr(s, vdhdr);

    WaitForSingleObject(ctx->mutex, INFINITE);

    if(shall_we_drop(s, index, devtype))
        goto fail;

    pktl_next = av_mallocz(sizeof(*pktl_next));
    if(!pktl_next)
        goto fail;

    if(av_new_packet(&pktl_next->pkt, buf_size) < 0) {
        av_free(pktl_next);
        goto fail;
    }

    pktl_next->pkt.stream_index = index;
    pktl_next->pkt.pts = time;
    memcpy(pktl_next->pkt.data, buf, buf_size);

    for(ppktl = &ctx->pktl ; *ppktl ; ppktl = &(*ppktl)->next);
    *ppktl = pktl_next;
    ctx->curbufsize[index] += buf_size;

    SetEvent(ctx->event[1]);
    ReleaseMutex(ctx->mutex);

    return;
fail:
    ReleaseMutex(ctx->mutex);
    return;
}
```

```c
static int dshow_read_packet(AVFormatContext *s, AVPacket *pkt)
{
    struct dshow_ctx *ctx = s->priv_data;
    PacketListEntry *pktl = NULL;

    while (!ctx->eof && !pktl) {
        WaitForSingleObject(ctx->mutex, INFINITE);
        pktl = ctx->pktl;
        if (pktl) {
            *pkt = pktl->pkt;
            ctx->pktl = ctx->pktl->next;
            av_free(pktl);
            ctx->curbufsize[pkt->stream_index] -= pkt->size;
        }
        ResetEvent(ctx->event[1]);
        ReleaseMutex(ctx->mutex);
        if (!pktl) {
            if (dshow_check_event_queue(ctx->media_event) < 0) {
                ctx->eof = 1;
            } else if (s->flags & AVFMT_FLAG_NONBLOCK) {
                return AVERROR(EAGAIN);
            } else {
                WaitForMultipleObjects(2, ctx->event, 0, INFINITE);
            }
        }
    }

    return ctx->eof ? AVERROR(EIO) : pkt->size;
}
```

`callback`会在工作线程读取音频数据,放到`ctx->pktl`,而`dshow_read_packet`里面的while循环回去判断`ctx->pktl`是否有数据,这里的mutex锁就是为了解决两个线程的同步问题。

如果判断还没有数据而且没有读取到eof,就会根据AVFormatContext::flags判断是否为阻塞读取,非阻塞直接返回again的错误码让调用方重试,如果是阻塞就等待数据填充的信号量。

这个信号量是在`callback`里面填充完成之后通过SetEvent发送的,而由于我们已经把设备拔掉了,所以永远不会有callback的回调,于是`dshow_read_packet`就卡死了。


### 解决方案一

解决的方式有两种,一是创建看门狗线程,在读取超时之后手动唤醒:

```c++
// 原始的priv_data数据结构
struct dshow_ctx {
    const AVClass *class;

    IGraphBuilder *graph;

    char *device_name[2];
    char *device_unique_name[2];

    int video_device_number;
    int audio_device_number;

    int   list_options;
    int   list_devices;
    int   audio_buffer_size;
    int   crossbar_video_input_pin_number;
    int   crossbar_audio_input_pin_number;
    char *video_pin_name;
    char *audio_pin_name;
    int   show_video_device_dialog;
    int   show_audio_device_dialog;
    int   show_video_crossbar_connection_dialog;
    int   show_audio_crossbar_connection_dialog;
    int   show_analog_tv_tuner_dialog;
    int   show_analog_tv_tuner_audio_dialog;
    char *audio_filter_load_file;
    char *audio_filter_save_file;
    char *video_filter_load_file;
    char *video_filter_save_file;
    int   use_video_device_timestamps;

    IBaseFilter *device_filter[2];
    IPin        *device_pin[2];
    DShowFilter *capture_filter[2];
    DShowPin    *capture_pin[2];

    HANDLE mutex;
    HANDLE event[2]; /* event[0] is set by DirectShow
                      * event[1] is set by callback() */
    PacketListEntry *pktl;

    int eof;

    int64_t curbufsize[2];
    unsigned int video_frame_num;

    IMediaControl *control;
    IMediaEvent *media_event;

    enum AVPixelFormat pixel_format;
    enum AVCodecID video_codec_id;
    char *framerate;

    int requested_width;
    int requested_height;
    AVRational requested_framerate;

    int sample_rate;
    int sample_size;
    int channels;
};

// 计算priv_data数据偏移,去除无用的符号依赖
struct PrivDataOffsetHelper {
    char* pointer_place_holder[20];
    int int_place_holder[14];

    HANDLE mutex;
    HANDLE event[2]; 
    char *pktl;
};

// 看门狗设置
watch_dog_timer_.expires_after(2s);
watch_dog_timer_.async_wait([this](const boost::system::error_code& code) {
    if (code == boost::asio::error::operation_aborted) {
        return;
    }
    LOGW("FORCE WAKEUP FOR WATCH DOG!!!");

    format_context_->flags |= AVFMT_FLAG_NONBLOCK;
    auto ctx = (PrivDataOffsetHelper*)format_context_->priv_data;
    ctx->pktl = nullptr;
    SetEvent(ctx->event[1]);
});

auto error = av_read_frame(format_context_, packet);
watch_dog_timer_.cancel();
```

这种方式最大的问题在于如果以后更新ffmpeg的时候priv\_data的数据结构改变了,内存偏移也变了的话就会出现野指针。它是一种无奈的做法,如果按我一开始想的是卡在mutex上,而外部设备拔出之后的确没有办法从锁的触发流程上规避的话就只能用这种硬编码方式主动调用ReleaseMutex。


### 解决方案二

这里实际上使用非阻塞的方式去读取去解决会更加合适一点:

```c++

// 打开AVFormatContext之后设置AVFMT_FLAG_NONBLOCK
auto error = avformat_open_input(&format_context_, device.second.data(), iformat, &options);
format_context_->flags |= AVFMT_FLAG_NONBLOCK;


// 使用非阻塞的方式不断读取
auto error = av_read_frame(format_context_, packet);
auto start = std::chrono::high_resolution_clock::now();
while (!exit_ && error == AVERROR(EAGAIN)) {
	// 如果读取的时间超过阈值就代表已经拔出设备
    auto now = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(now - start);
    if (duration.count() >= kReadFrameTimeout) {
        LOGW("read frame timeout");
        break;
    }

    // 暂停10ms防止空转耗费cpu
    std::this_thread::sleep_for(std::chrono::milliseconds(10));

    // 重新尝试读取
    error = av_read_frame(format_context_, packet);
}
```


# WinDbg分析线程死锁

本来以为问题已经解决了,但是在自检的时候又发现了另外的死锁现象:

```
   8  Id: 1d2c.3f14 Suspend: 1 Teb: 0000000a`5f55f000 Unfrozen
 # Child-SP          RetAddr               : Args to Child                                                           : Call Site
00 0000000a`5fefe0e8 00007ffc`d57a30ce     : 00000001`960a9301 000001e5`2feed570 000001e5`2feed570 000001e5`2feed570 : ntdll!NtWaitForSingleObject+0x14
01 0000000a`5fefe0f0 00007ffc`d1d21985     : 000001e5`2d26fe20 00000000`00000200 00000001`00000000 00000000`000018a4 : KERNELBASE!WaitForSingleObjectEx+0x8e
02 0000000a`5fefe190 00007ffc`9609fabe     : 000001e5`2d437a00 00000000`00000030 00007ffc`960b0e38 000001e5`2d26fe20 : wdmaud!wodMessage+0x235
03 0000000a`5fefe330 00007ffc`960a0ae9     : 000001e5`2d26fe20 000001e5`2d26fe20 000001e5`2d437a00 00000000`00000030 : winmmbase!waveMessage+0x9a
04 0000000a`5fefe370 00007ffc`d1f42ba1     : 000001e5`2d2d5240 00000000`00000030 000001e5`26218098 00000000`00000000 : winmmbase!waveOutWrite+0x79
05 0000000a`5fefe3a0 00007ffc`d1f43e80     : 00000000`00000000 00000000`00000000 00000000`00000030 00007ffc`960a1db0 : msacm32!mapWaveWriteBuffer+0x69
06 0000000a`5fefe3e0 00007ffc`9609fabe     : 000001e5`2d2b3e50 00000000`00000030 00007ffc`960b0e38 000001e5`26218098 : msacm32!wodMessage+0xd0
07 0000000a`5fefe430 00007ffc`960a0ae9     : 000001e5`2d26fe20 000001e5`26218098 000001e5`2d2b3e50 00000000`00000030 : winmmbase!waveMessage+0x9a
08 0000000a`5fefe470 00007ff6`1012c5f6     : 000001e5`26217fd0 0000000a`5fefe840 0000000a`5feffd08 00000000`0000022c : winmmbase!waveOutWrite+0x79
09 0000000a`5fefe4a0 00007ff6`1012c432     : 000001e5`26215db0 000001e5`2ff2b920 0000000a`00015888 0000000a`5fefea18 : XXXXXXX!AudioPlayer::Play+0xd6 [C:\Users\user\workspace\XXXXXXX\src\audio\audio_player.cpp @ 410] 
```

这里卡死在Windows原生的waveOutWrite里面,由于没有源码,所以也没有办法调试。搜索了下资料,发现WinDbg其实是可以用`!locks`命令打印被持有的线程锁的信息:

```
0:015> !locks

CritSec +2d2b3e28 at 000001e52d2b3e28
WaiterWoken        No
LockCount          1
RecursionCount     1
OwningThread       3f14
EntryCount         0
ContentionCount    1
*** Locked

Scanned 27 critical sections
```

这里的输出代表着有线程进入了临界区在等待3f14线程持有的2d2b3e28锁。

我们通过`~`命令输出线程信息,找到3f14的线程序号是8:

```

0:015> ~
   0  Id: 1d2c.47f4 Suspend: 1 Teb: 0000000a`5f54f000 Unfrozen
   1  Id: 1d2c.4cc Suspend: 1 Teb: 0000000a`5f5bb000 Unfrozen
   2  Id: 1d2c.2cec Suspend: 1 Teb: 0000000a`5f46e000 Unfrozen
   3  Id: 1d2c.150c Suspend: 1 Teb: 0000000a`5f470000 Unfrozen
   4  Id: 1d2c.2260 Suspend: 1 Teb: 0000000a`5f557000 Unfrozen
   5  Id: 1d2c.31c8 Suspend: 1 Teb: 0000000a`5f559000 Unfrozen
   6  Id: 1d2c.d58 Suspend: 1 Teb: 0000000a`5f55b000 Unfrozen
   7  Id: 1d2c.1010 Suspend: 1 Teb: 0000000a`5f55d000 Unfrozen
   8  Id: 1d2c.3f14 Suspend: 1 Teb: 0000000a`5f55f000 Unfrozen
   9  Id: 1d2c.33f0 Suspend: 1 Teb: 0000000a`5f561000 Unfrozen
  10  Id: 1d2c.21b4 Suspend: 1 Teb: 0000000a`5f563000 Unfrozen
  11  Id: 1d2c.2a68 Suspend: 1 Teb: 0000000a`5f565000 Unfrozen
  12  Id: 1d2c.227c Suspend: 1 Teb: 0000000a`5f567000 Unfrozen
  13  Id: 1d2c.3be0 Suspend: 1 Teb: 0000000a`5f569000 Unfrozen
  14  Id: 1d2c.2364 Suspend: 1 Teb: 0000000a`5f56b000 Unfrozen
. 15  Id: 1d2c.455c Suspend: 1 Teb: 0000000a`5f474000 Unfrozen
  16  Id: 1d2c.154c Suspend: 1 Teb: 0000000a`5f472000 Unfrozen
  21  Id: 1d2c.3f2c Suspend: 1 Teb: 0000000a`5f5a3000 Unfrozen
  22  Id: 1d2c.46b8 Suspend: 1 Teb: 0000000a`5f47e000 Unfrozen
  28  Id: 1d2c.12fc Suspend: 1 Teb: 0000000a`5f492000 Unfrozen
  30  Id: 1d2c.15cc Suspend: 1 Teb: 0000000a`5f496000 Unfrozen
```

然后使用`~8k`命令输出序号为8的线程堆栈,可以发现这个线程就是一开始我们看到的卡死的waveOutWrite线程:

```
0:015> ~8k
 # Child-SP          RetAddr               Call Site
00 0000000a`5fefe0e8 00007ffc`d57a30ce     ntdll!NtWaitForSingleObject+0x14
01 0000000a`5fefe0f0 00007ffc`d1d21985     KERNELBASE!WaitForSingleObjectEx+0x8e
02 0000000a`5fefe190 00007ffc`9609fabe     wdmaud!wodMessage+0x235
03 0000000a`5fefe330 00007ffc`960a0ae9     winmmbase!waveMessage+0x9a
04 0000000a`5fefe370 00007ffc`d1f42ba1     winmmbase!waveOutWrite+0x79
05 0000000a`5fefe3a0 00007ffc`d1f43e80     msacm32!mapWaveWriteBuffer+0x69
06 0000000a`5fefe3e0 00007ffc`9609fabe     msacm32!wodMessage+0xd0
07 0000000a`5fefe430 00007ffc`960a0ae9     winmmbase!waveMessage+0x9a
08 0000000a`5fefe470 00007ff6`1012c5f6     winmmbase!waveOutWrite+0x79
09 0000000a`5fefe4a0 00007ff6`1012c432     XXXXXXX!AudioPlayer::Play+0xd6 [C:\Users\user\workspace\XXXXXXX\src\audio\audio_player.cpp @ 410] 
```


这个线程卡持有了2d2b3e28锁之后卡在了WaitForSingleObject,导致其他线程无法获取2d2b3e28锁。一般死锁就是两个线程相互持有对方的锁导致的,所以我们需要找到哪个线程想要获取2d2b3e28锁。可以用`~*kv`命令打印打印线程堆栈并显示传递给每个函数的前三个参数,然后搜索2d2b3e28:

```
  21  Id: 1d2c.3f2c Suspend: 1 Teb: 0000000a`5f5a3000 Unfrozen
 # Child-SP          RetAddr               : Args to Child                                                           : Call Site
00 0000000a`60bff6a8 00007ffc`d7d338bd     : 000001e5`261e0000 000001e5`2ff56a80 000001e5`28c13cd0 00007ffc`d1d5e000 : ntdll!NtWaitForAlertByThreadId+0x14
01 0000000a`60bff6b0 00007ffc`d7d33772     : 00000000`00000000 00000000`00000000 0000000a`60bff798 000001e5`2d2b3e30 : ntdll!RtlpWaitOnAddressWithTimeout+0x81
02 0000000a`60bff6e0 00007ffc`d7d3358d     : 000001e5`看这里-----> 2d2b3e28 <-----看这里 00000000`00001722 00000000`00000000 00007ffc`d7cf5ba1 : ntdll!RtlpWaitOnAddress+0xae
03 0000000a`60bff750 00007ffc`d7cffcb4     : 0000b9bb`eef11ce8 000001e5`261e0000 00000000`fffffffa 00000000`00000000 : ntdll!RtlpWaitOnCriticalSection+0xfd
04 0000000a`60bff830 00007ffc`d7cffae2     : 0000000a`60bff8d0 000001e5`2ff56ac0 000001e5`2d2b3e50 00000000`00000000 : ntdll!RtlpEnterCriticalSectionContended+0x1c4
05 0000000a`60bff890 00007ffc`9609fa57     : 000001e5`2d2b3e00 00007ffc`960a1db0 000001e5`2feed580 00000000`00000000 : ntdll!RtlEnterCriticalSection+0x42
06 0000000a`60bff8c0 00007ffc`960a0a26     : 000001e5`28c0a7b0 00000000`000003bd 00000000`00000030 000001e5`2d2b3e50 : winmmbase!waveMessage+0x33
07 0000000a`60bff900 00007ff6`1012e74c     : 00000000`00000000 000001e5`2d2b3e50 00000000`00000000 00007ffc`d7cf5ba1 : winmmbase!waveOutUnprepareHeader+0x76
08 0000000a`60bff930 00007ffc`960955ff     : 000001e5`2d2b3e50 00007ffc`000003bd 000001e5`26215dd0 000001e5`28c0a7b0 : XXXXXXX!`anonymous namespace'::WaveOutProc+0x5c [C:\Users\user\workspace\XXXXXXX\src\audio\audio_player.cpp @ 109] 
09 0000000a`60bff980 00007ffc`d1f41043     : 000001e5`2d2d5240 00000000`000003bd 000001e5`28c0a7b0 000001e5`28c3b930 : winmmbase!DriverCallback+0x9f
0a 0000000a`60bff9c0 00007ffc`d1f4123c     : 000001e5`2d4532d0 000001e5`261e0000 00000000`00000002 00000000`00000000 : msacm32!mapWaveDriverCallback+0x3b
0b 0000000a`60bffa10 00007ffc`960955ff     : 00000000`00000000 00000000`000003bd 000001e5`2d437a00 00000000`00000000 : msacm32!mapWaveCallback+0x1dc
0c 0000000a`60bffa40 00007ffc`d1d23ce4     : 000001e5`28c3b930 00000000`00000002 00000000`00000000 00000000`00000808 : winmmbase!DriverCallback+0x9f
0d 0000000a`60bffa80 00007ffc`d1d235c6     : 000001e5`28c13cd0 00000000`0002b110 00007ffc`d1d5e000 00000000`00000000 : wdmaud!CWaveHandle::Work+0x324
0e 0000000a`60bffb00 00007ffc`d1d23410     : 000001e5`2d215a60 000001e5`2d215a60 00000000`00000000 00000000`00000000 : wdmaud!CWorker::_ThreadProc+0x186
0f 0000000a`60bffd80 00007ffc`d6b37614     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : wdmaud!CWorker::_StaticThreadProc+0x40
10 0000000a`60bffdb0 00007ffc`d7d226b1     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : KERNEL32!BaseThreadInitThunk+0x14
11 0000000a`60bffde0 00000000`00000000     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!RtlUserThreadStart+0x21
```

发现在audio\_player.cpp的第109行,调用了waveOutUnprepareHeader,在里面传入2d2b3e28给RtlpWaitOnAddress函数去等待这个锁。

这里实际上是waveOutOpen注册的callback函数,我在里面去释放播放数据的缓存:

```c++
void CALLBACK WaveOutProc(
   HWAVEOUT  hwo,
   UINT      msg,
   DWORD_PTR instance,
   DWORD_PTR param1,
   DWORD_PTR param2
) {
    if(WOM_DONE == msg) {
        auto hdr = (WAVEHDR*)param1;
        auto handle_ptr = (HWAVEOUT*)instance;
        av_free(hdr->lpData);
        waveOutUnprepareHeader(*handle_ptr, hdr, sizeof(WAVEHDR));
        delete hdr;
    }
}


auto ret = waveOutOpen(&wave_out_handle_, WAVE_MAPPER, &wfx, (DWORD_PTR) WaveOutProc, (DWORD_PTR) &wave_out_handle_, CALLBACK_FUNCTION);
```

从堆栈上看这个callback是在子线程中回调的,所以结合现象我们可以大概分析出

1. wave有一条工作线程和一个线程锁
2. waveOutWrite会获取线程锁,然后往工作线程丢入任务,然后等待任务完成的信号量
3. callback也是在工作线程中回调的,我们在里面调用waveOutUnprepareHeader,这里面也需要获取线程锁
4. 由于线程锁已经被waveOutWrite持有,所以callback会卡住,即工作线程会阻塞等待线程锁
5. 而waveOutWrite又需要等待工作线程去执行丢入的任务
6. 于是发生了死锁

从[官方文档](https://learn.microsoft.com/en-us/previous-versions/dd743869(v=vs.85))里面其实也可以看到在callback里面调用其他wave函数会造成死锁(Calling other wave functions will cause deadlock.),只能怪我自己没有仔细阅读文档了:

```
Applications should not call any system-defined functions from inside a callback function, except for EnterCriticalSection, LeaveCriticalSection, midiOutLongMsg, midiOutShortMsg, OutputDebugString, PostMessage, PostThreadMessage, SetEvent, timeGetSystemTime, timeGetTime, timeKillEvent, and timeSetEvent. Calling other wave functions will cause deadlock.
```