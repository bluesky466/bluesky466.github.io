title: Windows蓝屏dump文件分析入门
date: 2024-05-09 20:19:51
tags:
    - 技术相关
    - Windows
---

之前的一篇笔记[<Windows Dump文件分析>](https://blog.islinjw.cn/2024/02/11/Windows-Dump%E6%96%87%E4%BB%B6%E5%88%86%E6%9E%90/)介绍了应用dump的生成方式和调试手法,有稍微提一句蓝屏dump。但是实际在分析蓝屏的dump文件的时候和普通应用的dump文件分析还是有一些差异的。这篇笔记就记录下蓝屏dump的一些相关知识。

# 蓝屏dump文件的配置

首先我们可以在"右键选中我的电脑->属性->高级系统设置->高级->启动和故障恢复设置->系统失败"里面设置蓝屏时候生成dump文件的一些逻辑:

- 将事件写入系统日志 : 顾名思义
- 自动重新启动 : 蓝屏dump保存完成之后释放需要自动重启,有时候可能需要保持蓝屏的状态usb线连接外部笔记本电脑,在外部笔记本使用WinDbg的Attach to kernal功能调试系统可以去掉这个勾选
- 写入调试信息 : 用于选择生成蓝屏dump的内容,例如截图选择的“完成内存转储”我们也叫full dump,即将整个物理内存dump下来保存成dump文件(可以参考[官方文档](https://learn.microsoft.com/zh-cn/troubleshoot/windows-client/performance/configure-system-failure-and-recovery-options?source=recommendations)的详细介绍)
- 转存文件 : 生成dump文件的路径,例如截图的“%SystemRoot%\MEMORY.DMP”指的就是“C:\Windows\MEMORY.DMP”
- 覆盖任何现有文件 : 如果有新的dump生成会覆盖现有的dump文件
- 禁止在磁盘空间不足时自动删除内存转储 : 当磁盘空间不足的时候默认会删除dump文件,可以勾选禁止自动删除防止dump文件丢失

{% img /蓝屏dump/dump.png %}

蓝屏一般是驱动之类的底层代码出现异常导致的,在学习调试的时候可以使用[NotMyFault](https://learn.microsoft.com/zh-cn/troubleshoot/windows-client/performance/generate-a-kernel-or-complete-crash-dump#use-the-notmyfault-tool)去主动触发蓝屏获取dump文件。

# 内核模式

拿到之后可以同样用WinDbg打开它,打开full dump可以看到命令行提示为`kd`,意味着进入的是kernel debug内核模式:

{% img /蓝屏dump/kd.png %}

在这个模式下有些命令是不能使用的例如我们在调试应用crash的时候常用的[~](https://learn.microsoft.com/zh-cn/windows-hardware/drivers/debuggercmds/---thread-status-)就只能在用户模式下使用:

{% img /蓝屏dump/thread.png %}

如果在kd下输入就会报错:

```
5: kd> ~
       ^ Syntax error in '~'
```

不过依然可以直接`!analyze -v`分析:
```
DRIVER_IRQL_NOT_LESS_OR_EQUAL (d1)
...

myfault+0x1560:
fffff804`a0a41560 8b03            mov     eax,dword ptr [rbx] ds:00000000`00000000=????????
Resetting default scope

STACK_TEXT:  
ffff860c`933aec48 fffff802`3bc11aa9     : 00000000`0000000a ffff8904`92e54560 00000000`00000002 00000000`00000000 : nt!KeBugCheckEx
ffff860c`933aec50 fffff802`3bc0d563     : 00000000`00000000 00000000`00000000 00000000`00000f4d 00000000`00000000 : nt!KiBugCheckDispatch+0x69
ffff860c`933aed90 fffff804`a0a41560     : ffff9b0e`6e6470c0 fffff802`3ba4085f ffff9b0e`6e6470c0 ffff9b0e`61c0ee78 : nt!KiPageFault+0x463
ffff860c`933aef20 fffff804`a0a4191e     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : myfault+0x1560
ffff860c`933aef50 fffff804`a0a41a81     : 00000000`00000002 00000000`00000000 ffff9b0e`00000001 fffff802`3be462c1 : myfault+0x191e
ffff860c`933af090 fffff802`3ba35cf5     : 00000000`00000002 00000000`00000000 ffff860c`933af480 00000000`00000000 : myfault+0x1a81
ffff860c`933af0f0 fffff802`3be452ac     : 00000000`00000001 00000000`83360018 ffff9b0e`728045d0 fffff802`00000000 : nt!IofCallDriver+0x55
ffff860c`933af130 fffff802`3be44f03     : ffff9b0e`00000000 ffff860c`933af480 00000000`00010000 00000000`83360018 : nt!IopSynchronousServiceTail+0x34c
ffff860c`933af1d0 fffff802`3be441d6     : ffff9b0e`70743ab0 00000000`00000000 00000000`00000000 00000000`00000000 : nt!IopXxxControlFile+0xd13
ffff860c`933af320 fffff802`3bc11235     : ffff9b0e`6e6470c0 ffff860c`933af480 000000a3`a65eead8 ffff860c`933af3a8 : nt!NtDeviceIoControlFile+0x56
ffff860c`933af390 00007ffd`3b06d0c4     : 00007ffd`386e591b 00000000`00010000 00007ff7`83a24e88 000000a3`a65ef77c : nt!KiSystemServiceCopyEnd+0x25
000000a3`a65ef618 00007ffd`386e591b     : 00000000`00010000 00007ff7`83a24e88 000000a3`a65ef77c 000000a3`a65ef6a0 : ntdll!NtDeviceIoControlFile+0x14
000000a3`a65ef620 00007ffd`3a6d5921     : 00000000`83360018 00007ff7`83a96500 00000000`00000001 00007ff7`839dae99 : KERNELBASE!DeviceIoControl+0x6b
000000a3`a65ef690 00007ff7`839db437     : 0000028a`91751d30 0000028a`91751d85 000000a3`a65ef830 00000000`00000003 : KERNEL32!DeviceIoControlImplementation+0x81
000000a3`a65ef6e0 00007ff7`839dd162     : 0000028a`91751d30 000000a3`a65ef830 00000000`00000001 0000028a`91730000 : notmyfaultc64+0xb437
000000a3`a65ef730 00007ff7`839dd8e8     : 00000000`00000000 00007ff7`839dda71 0000028a`91751d30 00000000`00000000 : notmyfaultc64+0xd162
000000a3`a65ef900 00007ffd`3a6d7344     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : notmyfaultc64+0xd8e8
000000a3`a65ef940 00007ffd`3b0226b1     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : KERNEL32!BaseThreadInitThunk+0x14
000000a3`a65ef970 00000000`00000000     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!RtlUserThreadStart+0x21
```

可以看到蓝屏的代码是`DRIVER_IRQL_NOT_LESS_OR_EQUAL`,然后从后面的汇编可以看出来是读取0地址的内存到eax寄存器:

> fffff804`a0a41560 8b03            mov     eax,dword ptr [rbx] ds:00000000`00000000=????????

然后就是调用堆栈,可以看到的确是从notmyfault触发的。

# 分析其他进程

有时候驱动的异常是应用层调用接口的流程异常导致的,默认情况下内存上下文是触发蓝屏的进程,我们也可以转存full dump之后选择其他的进程进行分析。

在内核模式下我们可以用[!process 0 0](https://learn.microsoft.com/zh-cn/windows-hardware/drivers/debuggercmds/-process)这个命令列出所有的进程,如果你知道程序的exe名字也可以用在后面加上直接列举,例如`!process 0 0 Demo.exe`:

```
4: kd> !process 0 0 Demo.exe
PROCESS ffff9b0e6ef94080
    SessionId: 1  Cid: 0b48    Peb: 74b5500000  ParentCid: 247c
    DirBase: 23b8b1000  ObjectTable: ffff890492285600  HandleCount:  45.
    Image: Demo.exe
```

这个命令里面的第二个0是Flags用于指定需要展示进程的什么信息，0的话就是摘要。如果是`!process 0 0xf Demo.exe`就会列举出所有的Demo.exe的所有信息,例如全部的线程堆栈等,但是由于默认不会加载其他进程的pdb所以很多的符号看不到。


但可以从上面的信息看到进程的id是ffffe78e91d032c0,所以我们可以用[.process /p /r ffff9b0e6ef94080](https://learn.microsoft.com/zh-cn/windows-hardware/drivers/debuggercmds/-process--set-process-context-)指定用于进程上下文的进程并加载pdb。

然后用[lm](https://learn.microsoft.com/zh-cn/windows-hardware/drivers/debuggercmds/lm--list-loaded-modules-)命令就可以看到Demo.exe已经加载了,然后括号里的deferred代表后面有需要的时候就会去加载这个模块的pdb:

```
4: kd> lm
start             end                 module name
00007ff7`000b0000 00007ff7`000c0000   Demo       (deferred)   
...
```

之后再用`!process 0 0xf Demo.exe`就能看到具体的线程堆栈了：

```
4: kd> !process 0 0xf Demo.exe
PROCESS ffff9b0e6ef94080
    SessionId: 1  Cid: 0b48    Peb: 74b5500000  ParentCid: 247c
    DirBase: 23b8b1000  ObjectTable: ffff890492285600  HandleCount:  45.
    Image: Demo.exe
    VadRoot ffff9b0e720c7870 Vads 31 Clone 0 Private 143. Modified 0. Locked 0.
    DeviceMap ffff89047ff7a660
    Token                             ffff890492286060
    ElapsedTime                       00:00:19.086
    UserTime                          00:00:00.000
    KernelTime                        00:00:00.000
    QuotaPoolUsage[PagedPool]         25456
    QuotaPoolUsage[NonPagedPool]      4480
    Working Set Sizes (now,min,max)  (953, 50, 345) (3812KB, 200KB, 1380KB)
    PeakWorkingSetSize                916
    VirtualSize                       4145 Mb
    PeakVirtualSize                   4145 Mb
    PageFaultCount                    986
    MemoryPriority                    BACKGROUND
    BasePriority                      8
    CommitCharge                      164
    Job                               ffff9b0e6fb0e060

        THREAD ffff9b0e70a9f2c0  Cid 0b48.2bac  Teb: 00000074b5501000 Win32Thread: 0000000000000000 WAIT: (DelayExecution) UserMode Non-Alertable
            ffffffffffffffff  NotificationEvent
        Not impersonating
        DeviceMap                 ffff89047ff7a660
        Owning Process            ffff9b0e6ef94080       Image:         Demo.exe
        Attached Process          N/A            Image:         N/A
        Wait Start TickCount      194267         Ticks: 56 (0:00:00:00.875)
        Context Switch Count      107            IdealProcessor: 2             
        UserTime                  00:00:00.000
        KernelTime                00:00:00.000
        Win32 Start Address Demo!ILT+340(mainCRTStartup) (0x00007ff7000b1159)
        Stack Init ffff860c91fb7590 Current ffff860c91fb7070
        Base ffff860c91fb8000 Limit ffff860c91fb1000 Call 0000000000000000
        Priority 8 BasePriority 8 PriorityDecrement 0 IoPriority 2 PagePriority 5
        Child-SP          RetAddr               Call Site
        ffff860c`91fb70b0 fffff802`3ba41330     nt!KiSwapContext+0x76
        ffff860c`91fb71f0 fffff802`3ba4085f     nt!KiSwapThread+0x500
        ffff860c`91fb72a0 fffff802`3bacc132     nt!KiCommitThreadWait+0x14f
        ffff860c`91fb7340 fffff802`3be8be2f     nt!KeDelayExecutionThread+0x122
        ffff860c`91fb73d0 fffff802`3bc11235     nt!NtDelayExecution+0x5f
        ffff860c`91fb7400 00007ffd`3b06d664     nt!KiSystemServiceCopyEnd+0x25 (TrapFrame @ ffff860c`91fb7400)
        00000074`b56ff858 00007ffd`386fb62e     ntdll!NtDelayExecution+0x14
        00000074`b56ff860 00007ffd`08d1285c     KERNELBASE!SleepEx+0x9e
        00000074`b56ff900 00007ff7`000b16b3     MSVCP140!_Thrd_sleep+0x3c [d:\agent\_work\3\s\src\vctools\crt\crtw32\stdcpp\thr\cthread.cpp @ 70] 
        00000074`b56ff950 00007ff7`000b1619     Demo!std::this_thread::sleep_until<std::chrono::steady_clock,std::chrono::duration<__int64,std::ratio<1,1000000000> > >+0x83 [C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.36.32532\include\thread @ 199] 
        00000074`b56ff9a0 00007ff7`000b1736     Demo!std::this_thread::sleep_for<__int64,std::ratio<1,1> >+0x19 [C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.36.32532\include\thread @ 205] 
        00000074`b56ff9d0 00007ff7`000b175f     Demo!foo+0x36 [C:\Users\user\workspace\CppAutoRegisterDemo\main.cpp @ 8] 
        00000074`b56ffa00 00007ff7`000b1ab8     Demo!main+0xf [C:\Users\user\workspace\CppAutoRegisterDemo\main.cpp @ 12] 
        (Inline Function) --------`--------     Demo!invoke_main+0x22 (Inline Function @ 00007ff7`000b1ab8) [D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 78] 
        00000074`b56ffa30 00007ffd`3a6d7344     Demo!__scrt_common_main_seh+0x10c [D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288] 
        00000074`b56ffa70 00007ffd`3b0226b1     KERNEL32!BaseThreadInitThunk+0x14
        00000074`b56ffaa0 00000000`00000000     ntdll!RtlUserThreadStart+0x21
...
```

此时再用`lm`命令也可以看到pdb的确被加载了:

```
4: kd> lm
start             end                 module name
00007ff7`000b0000 00007ff7`000c0000   Demo     C (private pdb symbols)  d:\symbols\Demo.pdb
...
```

然后可以用`.thread ffff9b0e70a9f2c0`设置线程上下文就能在堆栈窗口看到对应的线程堆栈了。接着就能双击对应的栈帧打开代码窗口:

{% img /蓝屏dump/source.png %}

有时候还能在Locals窗口看到传入函数的参数的值，虽然这个dump没有抓到param的值。

# dump对虚拟内存大小的要求

在具体分析dump的时候发现有时候就算加载了正确的pdb文件也会有看到一些符号无法解析,我猜测是因为抓出来的dump只包含物理内存的信息,而这部分符号的地址在虚拟内存上。本来想关掉虚拟内存再抓取确认的。但是发现蓝屏dump的抓取对虚拟内存大小是有要求的，可以在[官方文档](https://learn.microsoft.com/zh-cn/troubleshoot/windows-client/performance/configure-system-failure-and-recovery-options?source=recommendations)上看到。例如full dump的要求如下:

|物理内存|虚拟内存要求|
|-|-|
|256 MB–1,373 MB|	物理内存大小的 1.5 倍|
|1,374 MB 或更大|	32 位系统：2 GB 加 16 MB<br/>64 位系统：物理内存的大小加上 128 MB|

所以有时候蓝屏dump抓不出来也可以看看是不是虚拟内存被关掉了。