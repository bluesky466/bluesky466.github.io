title: Android NDK Crash定位分析之寄存器
date: 2024-11-29 13:30:35
tags:
    - 技术相关
    - Android
    - C/C++
---


最近协助分析了一个audioserver crash的问题,堆栈如下:

```
11-27 11:39:45.974  1041 26144 26144 F DEBUG   : Cmdline: /system/bin/audioserver
11-27 11:39:45.974  1041 26144 26144 F DEBUG   : pid: 26089, tid: 26136, name: binder:26089_4  >>> /system/bin/audioserver <<<
11-27 11:39:45.974  1041 26144 26144 F DEBUG   : uid: 1041
11-27 11:39:45.974  1041 26144 26144 F DEBUG   : tagged_addr_ctrl: 0000000000000001 (PR_TAGGED_ADDR_ENABLE)
11-27 11:39:45.974  1041 26144 26144 F DEBUG   : signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0000000000000000
11-27 11:39:45.974  1041 26144 26144 F DEBUG   : Cause: null pointer dereference
11-27 11:39:45.974  1041 26144 26144 F DEBUG   :     x0  0000000000000000  x1  000000000000005c  x2  0000000000000000  x3  0000000000000000
11-27 11:39:45.974  1041 26144 26144 F DEBUG   :     x4  0000000000000000  x5  8080808080808080  x6  fefefefefefefeff  x7  7f7f7f7f7f7f7f7f
11-27 11:39:45.974  1041 26144 26144 F DEBUG   :     x8  a7a790ac07c2d30e  x9  a7a790ac07c2d30e  x10 0000000000000000  x11 0000000000000001
11-27 11:39:45.974  1041 26144 26144 F DEBUG   :     x12 0000006fde42d5d0  x13 0000000000000001  x14 0000000000000000  x15 000000729470a662
11-27 11:39:45.974  1041 26144 26144 F DEBUG   :     x16 00000072947a9d78  x17 00000072947301c0  x18 0000006fdd444000  x19 00000000000007cf
11-27 11:39:45.974  1041 26144 26144 F DEBUG   :     x20 0000006fde42dcd0  x21 0000000080000008  x22 0000000000000000  x23 b4000070cd42b7f0
11-27 11:39:45.974  1041 26144 26144 F DEBUG   :     x24 0000006fde42d9f0  x25 0000006fde42d980  x26 000000729a8dfee8  x27 000000729a8e1da0
11-27 11:39:45.974  1041 26144 26144 F DEBUG   :     x28 0000006fde42db10  x29 0000006fde42dc50
11-27 11:39:45.974  1041 26144 26144 F DEBUG   :     lr  000000729a8874a4  sp  0000006fde42d860  pc  000000729a887574  pst 0000000060001000
11-27 11:39:45.974  1041 26144 26144 F DEBUG   : backtrace:
11-27 11:39:45.974  1041 26144 26144 F DEBUG   :       #00 pc 0000000000035574  /system/lib64/libaudiopolicyenginedefault.so (android::audio_policy::Engine::getDeviceForInputSource(audio_source_t) const+5404) (BuildId: dedb284e4a36dc5488c54ec52bf97f75)
11-27 11:39:45.974  1041 26144 26144 F DEBUG   :       #01 pc 000000000002b4c4  /system/lib64/libaudiopolicyenginedefault.so (android::audio_policy::Engine::getInputDeviceForAttributes(audio_attributes_t const&, unsigned int, android::sp<android::AudioPolicyMix>*) const+672) (BuildId: dedb284e4a36dc5488c54ec52bf97f75)
...
```

从堆栈上能看出来是在getDeviceForInputSource方法里面出现了空指针导致奔溃。如果可以找到带符号表的so那么可以通过`android-addr2line -e 带符号表so路径 0000000000035574`命令直接定位到是c++哪一行源码出现的空指针。

例如这篇笔记最后的那个崩溃堆栈,用addr2line查看000000000009c758地址就能得到奔溃在native-api.cpp的57行(这部分详见我之前的[博客](https://blog.islinjw.cn/2020/06/05/Android-NDK-Crash%E5%AE%9A%E4%BD%8D%E5%88%86%E6%9E%90/)就不过多赘述了):

```
/Users/linjw/Library/Android/sdk/ndk/22.1.7171670/toolchains/aarch64-linux-android-4.9/prebuilt/darwin-x86_64/bin/aarch64-linux-android-addr2line -e ./app/.cxx/cmake/debug/arm64-v8a/lib/libdemo-native-api.so 000000000009c758
/Users/linjw/workspace/Demo/app/.cxx/cmake/debug/arm64-v8a/../../../../src/main/cpp/native-api.cpp:57
```

但是这里是发生在系统库里,我们这边不会保留系统的编译中间产物(各种带符号表的so),而且通过file命令也可以看到系统里的libaudiopolicyenginedefault.so显示`stripped`代表是已去除符号表的:

```
$ file /system/lib64/libaudiopolicyenginedefault.so
/system/lib64/libaudiopolicyenginedefault.so: ELF shared object, 64-bit LSB arm64, for Android 33, BuildID=2dcd2508ac02afde1acb25f0e6601435, stripped
```

如果没有去除符号表,会显示显示`not stripped`:

```
file libtest.so
libtest.so: ELF shared object, 64-bit LSB arm64, for Android 21, built by NDK r22b (7171670), BuildID=e678e5b301afc7fadfa7cd6b56e48c6223ed671e, not stripped
```

硬是要做的话可能需要切到这个软件的commit号本地编译,看看是否能编出对应的带符号so。但是这个编译环境和构建服务器的不一样不一定编出来的是和之前正式的软件一模一样的,另外编译也十分耗时。

所以只能先从日志的上下文还有源代码进行分析,从日志里面有看到下面的日志:

> 11-27 11:39:51.021  1041 26153 26195 W APM::AudioPolicyEngine: audiopolicydebug getDeviceForInputSource user selected off mic

而对应源代码里面有这样的代码,由于还会判断inputSource,只能说可能是这个原因,如果能看到inputSource的值确认会进入这个if判断的话才能百分百确认:

```c++
sp<DeviceDescriptor> Engine::getDeviceForInputSource(audio_source_t inputSource) const
{
	if (...) {
        ALOGW("audiopolicydebug %s user selected off mic", __func__);
        device = nullptr;
    }
    ...
    if(AUDIO_SOURCE_HOTWORD  == inputSource) {
        if(device->type() == AUDIO_DEVICE_IN_BUILTIN_MIC) {
            ...
        }
    }
    ...
}
```

从[源码](https://cs.android.com/android/platform/superproject/+/android13-dev:frameworks/proto_logging/stats/enums/media/audio/enums.proto;l=162)看到`AUDIO_SOURCE_HOTWORD`的值是1999:

```
AUDIO_SOURCE_HOTWORD = 1999;
```

而1999=0x7cf,从堆栈上看x19寄存器的值刚好就是`00000000000007cf`,所以基本能确定就是这个原因了,就算中间其他的代码没有问题走到这个if里面也会奔溃。

# 寄存器

我原本以为函数的入参都能在奔溃堆栈的寄存器信息里面看到,但是后面自己做了下实验发现还不一定能在寄存器里面看到参数的值。我们先从正常的情况来看如下面的代码(debug函数里面的这么多LOGD是为了增加行数防止内联):

```c++
void debug(int i, const char *str) {
    int x = 0xaaaa;
    int y = i + str[0] + x;
    LOGD("y=%d", y);
    LOGD("y=%d", y);
    LOGD("y=%d", y);
    LOGD("y=%d", y);
    LOGD("y=%d", y);
    LOGD("y=%d", y);
    LOGD("y=%d", y);
    LOGD("y=%d", y);
    LOGD("y=%d", y);
    LOGD("y=%d", y);
}


debug(0xabcd, nullptr);
```

运行之后就能看到空指针奔溃,从寄存器信息也能看到x0寄存器的值000000000000abcd就是我们的参数值:

```

11-28 18:57:48.054 17309 17309 F DEBUG   : tagged_addr_ctrl: 0000000000000001 (PR_TAGGED_ADDR_ENABLE)
11-28 18:57:48.054 17309 17309 F DEBUG   : signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0000000000000000
11-28 18:57:48.054 17309 17309 F DEBUG   : Cause: null pointer dereference
11-28 18:57:48.054 17309 17309 F DEBUG   :     x0  000000000000abcd  x1  0000000000000000  x2  0000000000000000  x3  0000000000000000
11-28 18:57:48.054 17309 17309 F DEBUG   :     x4  0000007487673000  x5  0000007fc0cfa740  x6  0000007fc0cfa718  x7  0000000000000000
11-28 18:57:48.054 17309 17309 F DEBUG   :     x8  984d9d4eddfe2a3b  x9  984d9d4eddfe2a3b  x10 0000000000000007  x11 0000007fc0cfa478
11-28 18:57:48.054 17309 17309 F DEBUG   :     x12 0000007fc0cfa360  x13 0000007fc0cfa330  x14 000000000000000a  x15 00000000ebad6a89
11-28 18:57:48.054 17309 17309 F DEBUG   :     x16 000000714f39a460  x17 000000714f1fa2ec  x18 00000074883f8000  x19 b400007291553190
11-28 18:57:48.054 17309 17309 F DEBUG   :     x20 0000000000000000  x21 0000000000000000  x22 0000007484938dce  x23 0000000000001071
11-28 18:57:48.054 17309 17309 F DEBUG   :     x24 00000071cdc00880  x25 0000007487673000  x26 00000000705d08a0  x27 0000007fc0cfb968
11-28 18:57:48.054 17309 17309 F DEBUG   :     x28 0000007fc0cfb860  x29 0000007fc0cfb790
11-28 18:57:48.054 17309 17309 F DEBUG   :     lr  000000714f1fa964  sp  0000007fc0cfb790  pc  000000714f1fa2fc  pst 0000000060001000
11-28 18:57:48.054 17309 17309 F DEBUG   : backtrace:
11-28 18:57:48.054 17309 17309 F DEBUG   :       #00 pc 00000000001ee2fc  /data/app/~~n278xi-BFwNh8wPmNeFUPA==/me.linjw.demo-bF94GJVWCMKCefwoNGhsrQ==/base.apk!libdemo-native-api.so (debug(int, char const*)+16) (BuildId: ca1c32722869b1a536437fd445250401017034de)
11-28 18:57:48.054 17309 17309 F DEBUG   :       #01 pc 00000000001ee960  /data/app/~~n278xi-BFwNh8wPmNeFUPA==/me.linjw.demo-bF94GJVWCMKCefwoNGhsrQ==/base.apk!libdemo-native-api.so (BuildId: ca1c32722869b1a536437fd445250401017034de)
11-28 18:57:48.054 17309 17309 F DEBUG   :       #02 pc 000000000021a354  /apex/com.android.art/lib64/libart.so (art_quick_generic_jni_trampoline+148) (BuildId: 7185f17e1e47100e6396535885066af5)
...
```

我们可以通过objdump对so进行反汇编,这个工具在ndk包里面,例如我的机器上可以通过下面命令将so反汇编输出到asm.txt:

```
/Users/linjw/Library/Android/sdk/ndk/22.1.7171670/toolchains/aarch64-linux-android-4.9/prebuilt/darwin-x86_64/bin/aarch64-linux-android-objdump -d libdemo-native-api.so > asm.txt
```

直接搜奔溃执行的汇编地址(1ee2fc)定位到对应代码:

```
00000000001ee2ec <_Z5debugiPKc@@Base>:
  1ee2ec:   a9bd7bfd    stp x29, x30, [sp,#-48]!
  1ee2f0:   f9000bf5    str x21, [sp,#16]
  1ee2f4:   a9024ff4    stp x20, x19, [sp,#32]
  1ee2f8:   910003fd    mov x29, sp
  1ee2fc:   39400028    ldrb    w8, [x1]
  1ee300:   52955549    mov w9, #0xaaaa                 // #43690
  1ee304:   f0fffc74    adrp    x20, 17d000 <_ZN3MD511HEX_NUMBERSE@@Base+0x2d30>
  1ee308:   d0fffc75    adrp    x21, 17c000 <_ZN3MD511HEX_NUMBERSE@@Base+0x1d30>
  1ee30c:   0b080008    add w8, w0, w8
  1ee310:   0b090113    add w19, w8, w9
  ...
  ...

...

00000000001ee888 <_ZN7_JNIEnv16CallObjectMethodEP8_jobjectP10_jmethodIDz@@Base>:
  ...

  1ee950:   529579a0    mov w0, #0xabcd                 // #43981
  1ee954:   aa1f03e1    mov x1, xzr
  1ee958:   aa0203f4    mov x20, x2
  1ee95c:   f81f83a8    stur    x8, [x29,#-8]
  1ee960:   9405a7f0    bl  358920 <_Z5debugiPKc@plt>
  ...
```

`1ee2fc`的汇编代码为`ldrb    w8, [x1]`将存储器地址为x1的字节数据读入寄存器w8,而x1的值在奔溃堆栈里面可以看到的确是`x1  0000000000000000`。

返回堆栈上一级的`1ee960`附近可以看到1ee950这里会将0xabcd设置到w0寄存器。

AArch64架构提供了31个通用寄存器,每个寄存器都可以用作64位X寄存器(X0～X30)或32位W寄存器(W0～W30),所以w0其实就是x0寄存器,在堆栈里面可以看到的确是`x0  000000000000abcd`

然后在debug函数里面可以看到后面会用w8去做加法:

```
  1ee30c:   0b080008    add w8, w0, w8
  1ee310:   0b090113    add w19, w8, w9
```

所以参数在前一级的函数里面设置到了寄存器里面,然后在debug函数里面就能直接使用了。

从arm的[官方文档](https://developer.arm.com/documentation/102374/0102/Procedure-Call-Standard?lang=en)可以看到各个寄存器的作用:

{% img /Android_NDK_Crash定位分析之寄存器/1.png %}

X0-X7是参数和结果寄存器,函数的参数和返回值由它们去传递,正常情况下可以在这里看到参数的值;

{% img /Android_NDK_Crash定位分析之寄存器/2.jpg %}


但是函数中间的临时变量也可以用它们去保存,如果甚至在参数使用完成之后可以修改掉保存参数的寄存器的值,而X19-X28则是Callee-saved寄存器,当函数退栈的时候需要恢复回去,官方文档里面是这么说的:

> For example, the function foo() can use registers X0 to X15 without needing to preserve their values. However, if foo() wants to use X19 to X28 it must save them to stack first, and then restore from the stack before returning.

所以一开始看到的audioserver奔溃寄存器信息里面参数和结果寄存器的寄存器值已经被覆盖掉了,但是刚好在前一级函数里面刚好将inputSource存到了X19寄存器里面(因为这种奇怪数字能刚好撞中的几率还是蛮低的)。

# 编译器优化

由于编译器会对代码做各种优化所以有时候甚至会将函数调用给优化掉,就更难从寄存器信息里面看到参数的值了。

```c++
void debug(int i, const char *str) {
    int x = 0xaaaa;
    int y = i + str[0] + x;
    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
}

debug(0xabcd, nullptr);
```

例如我将其他的LOGD都注释掉只剩下一个的话,奔溃堆栈就变成了下面的样子,libdemo-native-api.so里面的函数堆栈只剩下了一层了,所以函数被内联了,所以没有函数调用的过程:

```
11-28 20:08:45.525 27586 27586 F DEBUG   : tagged_addr_ctrl: 0000000000000001 (PR_TAGGED_ADDR_ENABLE)
11-28 20:08:45.525 27586 27586 F DEBUG   : signal 5 (SIGTRAP), code 1 (TRAP_BRKPT), fault addr 0x0000007151f31758
11-28 20:08:45.525 27586 27586 F DEBUG   :     x0  b400007291553190  x1  00000071be622180  x2  0000000000000000  x3  0000000000000000
11-28 20:08:45.525 27586 27586 F DEBUG   :     x4  0000007487673000  x5  0000007fc0cfa740  x6  0000007fc0cfa718  x7  0000000000000000
11-28 20:08:45.525 27586 27586 F DEBUG   :     x8  984d9d4eddfe2a3b  x9  984d9d4eddfe2a3b  x10 0000000000000007  x11 0000007fc0cfa478
11-28 20:08:45.525 27586 27586 F DEBUG   :     x12 0000007fc0cfa360  x13 0000007fc0cfa330  x14 000000000000000a  x15 00000000ebad6a89
11-28 20:08:45.525 27586 27586 F DEBUG   :     x16 0000007151f31758  x17 0000007fc0cfb850  x18 00000074883f8000  x19 b40000734154f380
11-28 20:08:45.525 27586 27586 F DEBUG   :     x20 0000000000000000  x21 0000000000000000  x22 0000007484938dce  x23 0000000000001071
11-28 20:08:45.525 27586 27586 F DEBUG   :     x24 00000071cdc00880  x25 0000007fc0cfb968  x26 00000000705d08a0  x27 0000007fc0cfb968
11-28 20:08:45.525 27586 27586 F DEBUG   :     x28 0000007fc0cfb860  x29 0000007fc0cfb860
11-28 20:08:45.525 27586 27586 F DEBUG   :     lr  00000071cdc1a358  sp  0000007fc0cfb850  pc  0000007151f31758  pst 0000000060001000
11-28 20:08:45.525 27586 27586 F DEBUG   : backtrace:
11-28 20:08:45.525 27586 27586 F DEBUG   :       #00 pc 000000000009c758  /data/app/~~cluxSX8rLOQhNQR0I2DMFA==/me.linjw.demo-ym0C4_78KOXiumCBzhKjpQ==/base.apk!libdemo-native-api.so (BuildId: 16aa24ceb636e7bcae7459d6e1403c8a2fa209ca)
11-28 20:08:45.525 27586 27586 F DEBUG   :       #01 pc 000000000021a354  /apex/com.android.art/lib64/libart.so (art_quick_generic_jni_trampoline+148) (BuildId: 7185f17e1e47100e6396535885066af5)
...
```

然后由于从编译器看来内联之后的代码都没有机会用到i这个参数,所以直接优化掉了,在反汇编的代码里面都搜索不到0xabcd的赋值,然后生成的代码这里直接一个[BRK](https://developer.arm.com/documentation/dui0801/l/A64-General-Instructions/BRK--A64-?lang=en)生成断点异常:

```

000000000009c6b8 <_ZN7_JNIEnv16CallObjectMethodEP8_jobjectP10_jmethodIDz@@Base>:
   ...
   9c758:   d4200020    brk #0x1
```

所以我们这里看到的堆栈也不是空指针异常,而是`TRAP_BRKPT`异常。

另外更加神奇的是如果我直接将所有的LOGD都注释掉,编译器判断这个代码没有任何的作用,则会直接都优化掉,然后我们运行的时候程序就会正常运行不会崩溃...

```c++
void debug(int i, const char *str) {
    int x = 0xaaaa;
    int y = i + str[0] + x;
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
//    LOGD("y=%d", y);
}

debug(0xabcd, nullptr);
```

当然上面所说的优化都属于编译器的行为不在c/c++的语言规范里面,不同的编译器编译出来的结果可能不一样。不得不感慨一句,c++真的太难了...