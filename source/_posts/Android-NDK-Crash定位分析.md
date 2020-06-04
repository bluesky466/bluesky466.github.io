title: Android NDK Crash定位分析
date: 2020-06-05 00:18:45
tags:
    - 技术相关
    - Android
---

当拿到应用的crash日志,如果是在java层出现了异常,相信大家都知道通过堆栈信息查找到奔溃的代码,但是如果是在native层出现了问题,面对下面的一堆内存地址,有些小伙伴可能就会觉得无从下手了:

```shell
30597 30597 F DEBUG   : *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
30597 30597 F DEBUG   : Build fingerprint: 'Xiaomi/chiron/chiron:8.0.0/OPR1.170623.027/V10.3.1.0.ODECNXM:user/release-keys'                
30597 30597 F DEBUG   : Revision: '0'
30597 30597 F DEBUG   : ABI: 'arm64'
30597 30597 F DEBUG   : pid: 30535, tid: 30535, name: me.linjw.ndkdemo  >>> com.me.linjw.ndkdemo <<<
30597 30597 F DEBUG   : signal 6 (SIGABRT), code -6 (SI_TKILL), fault addr --------
30597 30597 F DEBUG   : Abort message: 'Invalid address 0x7ffd3cfac0 passed to free: value not allocated'
30597 30597 F DEBUG   :     x0   0000000000000000  x1   0000000000007747  x2   0000000000000006  x3   0000000000000008
30597 30597 F DEBUG   :     x4   8000000000808080  x5   8000000000808080  x6   8000000000808080  x7   0000000000000008
30597 30597 F DEBUG   :     x8   0000000000000083  x9   d6a0828f4d3c1493  x10  0000000000000000  x11  0000000000000001
30597 30597 F DEBUG   :     x12  ffffffffffffffff  x13  0000000000000001  x14  003275d83bd3efb5  x15  0000c345d3d41566
30597 30597 F DEBUG   :     x16  0000007b582112e8  x17  0000007b581b2d2c  x18  0000007ffd3ce5c8  x19  0000000000007747
30597 30597 F DEBUG   :     x20  0000000000007747  x21  0000007b5520d000  x22  0000000000000000  x23  0000007b5821c878
30597 30597 F DEBUG   :     x24  0000000000000004  x25  0000007b55214c98  x26  0000000000000000  x27  0000000000000001
30597 30597 F DEBUG   :     x28  0000000000000001  x29  0000007ffd3cf8c0  x30  0000007b58166e54                                      
30597 30597 F DEBUG   :     sp   0000007ffd3cf880  pc   0000007b581b2d34  pstate 0000000060000000
30597 30597 F DEBUG   :
30597 30597 F DEBUG   : backtrace:
30597 30597 F DEBUG   :     #00 pc 0000000000069d34  /system/lib64/libc.so (tgkill+8)
30597 30597 F DEBUG   :     #01 pc 000000000001de50  /system/lib64/libc.so (abort+88)
30597 30597 F DEBUG   :     #02 pc 0000000000025644  /system/lib64/libc.so (__libc_fatal+116)
30597 30597 F DEBUG   :     #03 pc 0000000000091204  /system/lib64/libc.so (ifree+812)
30597 30597 F DEBUG   :     #04 pc 0000000000091484  /system/lib64/libc.so (je_free+120)
30597 30597 F DEBUG   :     #05 pc 000000000000f60c  /data/app/com.me.linjw.ndkdemo-qgq0-FTl7SRsBBdmCeMAdg==/lib/arm64/libnative-lib.so (_Z9willCrashv+80)
30597 30597 F DEBUG   :     #06 pc 000000000000f728  /data/app/com.me.linjw.ndkdemo-qgq0-FTl7SRsBBdmCeMAdg==/lib/arm64/libnative-lib.so (Java_com_me_linjw_ndkdemo_MainActivity_callNative+20)
30597 30597 F DEBUG   :     #07 pc 000000000000909c  /data/app/com.me.linjw.ndkdemo-qgq0-FTl7SRsBBdmCeMAdg==/oat/arm64/base.odex (offset 0x9000)
```

莫慌,这篇博客就来讲讲怎么分析这份崩溃日志。

{% img /AndroidNDK定位分析/0.jpg %}


# 信号

首先第一个知识点就是信号(signal)机制,它其实是进程间通信的一种方式。在处理ndk crash日志的时候可以大概理解为错误码,它描述了错误的大概原因。例如上面的log,可以看到这个程序是因为SIGABRT这个信号奔溃的,它的码字是6:

```shell
06-04 19:05:38.910 30597 30597 F DEBUG   : signal 6 (SIGABRT), code -6 (SI_TKILL), fault addr --------
```

我们常见的信号有下面这些:

| **信号** | **码值** | **描述**                                   |
| -------- | -------- | ------------------------------------------ |
| SIGILL   | 4        | 非法指令，例如损坏的可执行文件或代码区损坏 |
| SIGABRT  | 6        | 通过C函数abort()发送；为assert()使用       |
| SIGBUS   | 7        | 不存在的物理地址，更多为硬件或系统引起     |
| SIGFPE   | 8        | 浮点数运算错误，如除0操作                  |
| SIGKILL  | 9        | 迅速完全终止进程；不能被捕获               |
| SIGSEGV  | 11       | 段地址错误，例如空指针、野指针、数组越界等 |

从表里面我们知道SIGABRT信号的触发原因是通过C函数abort()发送为assert()使用,也就是说它是个断言失败,从日志里面我们还能看到abort的信息:

```shell
Abort message: 'Invalid address 0x7ffd3cfac0 passed to free: value not allocated'
```

# 堆栈分析

但是光知道SIGABRT信号我们是很难定位到问题的。所以我们还需要分析下面的堆栈信息,找到对应的代码:

```shell
30597 30597 F DEBUG   : backtrace:
30597 30597 F DEBUG   :     #00 pc 0000000000069d34  /system/lib64/libc.so (tgkill+8)
30597 30597 F DEBUG   :     #01 pc 000000000001de50  /system/lib64/libc.so (abort+88)
30597 30597 F DEBUG   :     #02 pc 0000000000025644  /system/lib64/libc.so (__libc_fatal+116)
30597 30597 F DEBUG   :     #03 pc 0000000000091204  /system/lib64/libc.so (ifree+812)
30597 30597 F DEBUG   :     #04 pc 0000000000091484  /system/lib64/libc.so (je_free+120)
30597 30597 F DEBUG   :     #05 pc 000000000000f60c  /data/app/com.me.linjw.ndkdemo-qgq0-FTl7SRsBBdmCeMAdg==/lib/arm64/libnative-lib.so (_Z9willCrashv+80)
30597 30597 F DEBUG   :     #06 pc 000000000000f728  /data/app/com.me.linjw.ndkdemo-qgq0-FTl7SRsBBdmCeMAdg==/lib/arm64/libnative-lib.so (Java_com_me_linjw_ndkdemo_MainActivity_callNative+20)
30597 30597 F DEBUG   :     #07 pc 000000000000909c  /data/app/com.me.linjw.ndkdemo-qgq0-FTl7SRsBBdmCeMAdg==/oat/arm64/base.odex (offset 0x9000)
```

从这里我们可以分析到libnative-lib.so里面的Java\_com\_me\_linjw\_ndkdemo\_MainActivity\_callNative调用了willCrash函数,然后在willCrash函数里面触发了异常:

```shell
30597 30597 F DEBUG   :     #05 pc 000000000000f60c  /data/app/com.me.linjw.ndkdemo-qgq0-FTl7SRsBBdmCeMAdg==/lib/arm64/libnative-lib.so (_Z9willCrashv+80)
30597 30597 F DEBUG   :     #06 pc 000000000000f728  /data/app/com.me.linjw.ndkdemo-qgq0-FTl7SRsBBdmCeMAdg==/lib/arm64/libnative-lib.so (Java_com_me_linjw_ndkdemo_MainActivity_callNative+20)
```

# C++ 编译器的函数名修饰

细心的同学可能会有疑问,函数名明明是显示的\_Z9willCrashv,为啥我会说是willCrash？它和下面的Java\_com\_me\_linjw\_ndkdemo\_MainActivity\_callNative有什么区别？

我们可以先来看看源代码确认下我没有骗你:

{% img /AndroidNDK定位分析/1.jpeg %}

那为什么willCrash在编译之后so里面会变成\_Z9willCrashv?这主要是C++编译器的函数名修饰功能在作怪。由于c++是支持重载的,也就是只要参数不一样,函数的名字可以相同。

这个重载其实在编译期就能确定，所以编译器实现重载的原理是给函数加上修饰符，例如在函数后面拼接上参数类型简写，这里\_Z9willCrashv最后拼接的v就代表void,说明该函数没有参数。

也就是说虽然你在代码里面写的是同样的函数名，但是在编译之后，重载的函数其实就变成了不同名字的不同函数。

解释完了\_Z9willCrashv我们再来说说Java\_com\_me\_linjw\_ndkdemo\_MainActivity\_callNative，为什么它又没有被修饰呢？原因就在于函数上面的extern "C"，它告诉编译器将这个函数当做c语言的函数来处理。而c语言是没有重载这一说的，所以也就不会改变它原本的函数名。

# 指令偏移地址

然后方法名+号后面的数字是指的什么？方法行数吗?实际去代码里面看Java\_com\_me\_linjw\_ndkdemo\_MainActivity\_callNative只有一行代码,找不到20行，同样willCrash也没有80行:

{% img /AndroidNDK定位分析/1.jpeg %}

这里我们来解释下+号后面的值的意义。我们都知道c/c++代码都是需要编译成二进制文件之后才能运行,而实际上程序就是通过执行二进制文件中的一条条指令来运行的。上面日志中的#06 pc 000000000000f728指的就是出现问题的时候Java\_com\_me\_linjw\_ndkdemo\_MainActivity\_callNative执行到了0x000000000000f728这个地址的指令,而后面的+20指的是这个地址相对方法起始地址的偏移。

说起来可能比较难以理解,这里我们直接通过反汇编libnative-lib.so来帮助理解。ndk提供了objdump工具用于反汇编,由于不同cpu架构的反编译工具也是不一样的,大家可以根据需要找到对应的程序进行反汇编:

```shell
 LinJW@LinJWdeMacBook-Pro  ~/Library/Android/sdk/ndk  find . -name "*objdump"
./20.0.5594570/toolchains/x86-4.9/prebuilt/darwin-x86_64/bin/i686-linux-android-objdump
./20.0.5594570/toolchains/x86-4.9/prebuilt/darwin-x86_64/i686-linux-android/bin/objdump
./20.0.5594570/toolchains/llvm/prebuilt/darwin-x86_64/aarch64-linux-android/bin/objdump
./20.0.5594570/toolchains/llvm/prebuilt/darwin-x86_64/bin/x86_64-linux-android-objdump
./20.0.5594570/toolchains/llvm/prebuilt/darwin-x86_64/bin/aarch64-linux-android-objdump
./20.0.5594570/toolchains/llvm/prebuilt/darwin-x86_64/bin/i686-linux-android-objdump
./20.0.5594570/toolchains/llvm/prebuilt/darwin-x86_64/bin/arm-linux-androideabi-objdump
./20.0.5594570/toolchains/llvm/prebuilt/darwin-x86_64/arm-linux-androideabi/bin/objdump
./20.0.5594570/toolchains/llvm/prebuilt/darwin-x86_64/x86_64-linux-android/bin/objdump
./20.0.5594570/toolchains/llvm/prebuilt/darwin-x86_64/i686-linux-android/bin/objdump
./20.0.5594570/toolchains/x86_64-4.9/prebuilt/darwin-x86_64/bin/x86_64-linux-android-objdump
./20.0.5594570/toolchains/x86_64-4.9/prebuilt/darwin-x86_64/x86_64-linux-android/bin/objdump
./20.0.5594570/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/bin/arm-linux-androideabi-objdump
./20.0.5594570/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/arm-linux-androideabi/bin/objdump
./20.0.5594570/toolchains/aarch64-linux-android-4.9/prebuilt/darwin-x86_64/aarch64-linux-android/bin/objdump
./20.0.5594570/toolchains/aarch64-linux-android-4.9/prebuilt/darwin-x86_64/bin/aarch64-linux-android-objdump
```

我这边使用的是aarch64-linux-android-objdump,命令如下:

```
aarch64-linux-android-objdump -S ./libnative-lib.so
```

然后我们搜索Java\_com\_me\_linjw\_ndkdemo\_MainActivity\_callNative找到这个方法的定义:

```asm
000000000000f714 <Java_com_me_linjw_ndkdemo_MainActivity_callNative@@Base>:
    f714:   d10083ff    sub sp, sp, #0x20
    f718:   a9017bfd    stp x29, x30, [sp,#16]
    f71c:   910043fd    add x29, sp, #0x10
    f720:   f90007e0    str x0, [sp,#8]
    f724:   f90003e1    str x1, [sp]
    f728:   97ffff0a    bl  f350 <_Z9willCrashv@plt>
    f72c:   a9417bfd    ldp x29, x30, [sp,#16]
    f730:   910083ff    add sp, sp, #0x20
    f734:   d65f03c0    ret
    f738:   d100c3ff    sub sp, sp, #0x30
    f73c:   a9027bfd    stp x29, x30, [sp,#32]
    ...
```

然后我们上面看到的pc 000000000000f728其实指的就是f728这个地址的指令,也就是bl指令,这个指令用于调用子程序,于是我们可以容易猜出这行指令的作用是跳转到willCrash方法:

```asm
f728:   97ffff0a    bl  f350 <_Z9willCrashv@plt>
```

而Java\_com\_me\_linjw\_ndkdemo\_MainActivity\_callNative的起始地址为000000000000f714，于是可以计算出000000000000f728相对函数起始地址的偏移为0xf728-0xf714=0x14，而0x14在十进制里面就是20。

# addr2line

如果对这些汇编指令比较熟悉的话当然可以分析定位问题,但是一般的安卓程序员可能对这块比较陌生。所以我们可以用addr2line工具直接定位到源代码。

我们从下面log可以得到两个地址000000000000f728、000000000000f60c

```shell
30597 30597 F DEBUG   :     #05 pc 000000000000f60c  /data/app/com.me.linjw.ndkdemo-qgq0-FTl7SRsBBdmCeMAdg==/lib/arm64/libnative-lib.so (_Z9willCrashv+80)
30597 30597 F DEBUG   :     #06 pc 000000000000f728  /data/app/com.me.linjw.ndkdemo-qgq0-FTl7SRsBBdmCeMAdg==/lib/arm64/libnative-lib.so (Java_com_me_linjw_ndkdemo_MainActivity_callNative+20)
```



使用这个命令的前提是我们要有带符号的so库,因为一般情况下打包到apk里面的so都是不带符号的(可以大概理解成java层的混淆,去掉了符号信息),所以如果直接从apk里面解压出so,然后使用addr2line会得到下面结果，全是问号:

```shell
??:?
```

带符号的so一般会在编译的过程中生成,所以可以在app/build目录里面递归搜索下,而且不同cpu架构也需要用不同的addr2line,命令如下:

```shell
aarch64-linux-android-addr2line -e ./app/build/intermediates/cmake/debug/obj/arm64-v8a/libnative-lib.so 000000000000f728 000000000000f60c
```

得到结果:

```shell
/Users/LinJW/workspace/NdkDemo/app/src/main/cpp/native-lib.cpp:19
/Users/LinJW/workspace/NdkDemo/app/src/main/cpp/native-lib.cpp:13
```

我们来对比下源码就能找到崩溃的原因是delete了字符串常量的内存:

{% img /AndroidNDK定位分析/2.jpeg %}

# ndk-stack

作为认真看到这里的同学,我必须要奖励好学的你一个福利,那就是ndk-stack,他也在ndk里面:

```shell
NDK目录/prebuilt/darwin-x86_64/bin/ndk-stack
```

首先我们将含有native crash的log保存到crash_log.txt用-dump参数出入,然后将所有带符号的so放到某个目录下,用-sym参数传入:

```shell
ndk-stack -sym ./app/build/intermediates/cmake/debug/obj/arm64-v8a/ -dump ~/Downloads/crash_log.txt
```

然后它就会对native堆栈使用addr2line和目录下的so去转换,最终输出带符号的堆栈信息:

```shell
********* Crash dump: **********
Build fingerprint: 'Xiaomi/chiron/chiron:8.0.0/OPR1.170623.027/V10.3.1.0.ODECNXM:user/release-keys'
Abort message: 'Invalid address 0x7ffd3cfac0 passed to free: value not allocated'
#00 0x0000000000069d34 /system/lib64/libc.so (tgkill+8)
#01 0x000000000001de50 /system/lib64/libc.so (abort+88)
#02 0x0000000000025644 /system/lib64/libc.so (__libc_fatal+116)
#03 0x0000000000091204 /system/lib64/libc.so (ifree+812)
#04 0x0000000000091484 /system/lib64/libc.so (je_free+120)
#05 0x000000000000f60c /data/app/com.me.linjw.ndkdemo-qgq0-FTl7SRsBBdmCeMAdg==/lib/arm64/libnative-lib.so (_Z9willCrashv+80)
                                                                                                           willCrash()
                                                                                                           /Users/LinJW/workspace/NdkDemo/app/src/main/cpp/native-lib.cpp:13:5
#06 0x000000000000f728 /data/app/com.me.linjw.ndkdemo-qgq0-FTl7SRsBBdmCeMAdg==/lib/arm64/libnative-lib.so (Java_com_me_linjw_ndkdemo_MainActivity_callNative+2
0)
                                                                                                           Java_com_cvte_tv_ndkdemo_MainActivity_callNative
                                                                                                           /Users/LinJW/workspace/NdkDemo/app/src/main/cpp/native-lib.cpp:19:5
#07 0x000000000000909c /data/app/com.me.linjw.ndkdemo-qgq0-FTl7SRsBBdmCeMAdg==/oat/arm64/base.odex (offset 0x9000)
```

ndk-stack在开始解析 logcat 输出时将查找第一行星号,所以拷贝的时候记得这行不能缺少:

```shell
*** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
```

当然通常情况下我们直接将logcat出来的所有日志传给它就好，它会自动根据星号行识别出native堆栈:

```shell
adb logcat | ndk-stack路径 -sym 存放带符号so库目录的路径
```

