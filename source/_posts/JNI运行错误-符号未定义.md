title: JNI运行错误-符号未定义
date: 2020-11-24 20:23:42
tags:
    - 技术相关
    - Android
    - C/C++
---

最近在弄ndk的时候遇到了个比较坑的问题，虽然最后发现原因挺低级的，但是的确花了我不少时间去查找，中间的分析手法可能不熟悉c/c++的同学会比较陌生，如果遇到的同样问题的话会无从下手。这里把整个分析的流程记录下来，希望有用。

背景项目分两个部分，自己编写的c库工程，和安卓工程，将它们分离的原因是这个c库的功能可能在其他的地方也能使用到。

由于项目只是初始阶段，为了验证流程，我先搭了个简单的demo框架，用c库工程编译出so之后导入到安卓工程。虽然整个代码比较简单，但是运行的时候直接就崩溃了，报找不到符号的异常。

# 问题还原

这里用个简单的demo还原下问题，JNI部分调用c库里面的getString函数返回字符串:

```c++
const char *getString(); // 这个函数的定义在c库工程编译出来的so库里面

extern "C" JNIEXPORT jstring JNICALL
Java_com_cvte_tv_ndkdemo_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
    return env->NewStringUTF(getString());
}
```

c库的代码也很简单，就返回字符串，我们会将它编译成libdemo.so:

```c++
const char* getString() {
    return "Hello world!\n";
}
```

cmake配置也很简单，我们的jni编译了一个libnative-lib.so依赖libdemo.so，java层通过这个libnative-lib.so去调用到libdemo.so里面的getString:

```cmake
cmake_minimum_required(VERSION 3.4.1)

add_library(native-lib SHARED native-lib.cpp)

add_library(demo SHARED IMPORTED)

set_target_properties(demo PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/jniLibs/${ANDROID_ABI}/libdemo.so)

target_link_libraries(native-lib  demo)
```

运行之后报的问题看起来也很简单:

```shell
java.lang.UnsatisfiedLinkError: dlopen failed: cannot locate symbol "_Z9getStringv" referenced by "/data/app/com.cvte.tv.ndkdemo-xD9KLsO5Wmh_YGDKRKL5lA==/lib/arm64/libnative-lib.so"...
```

这样奔溃其实挺常见的，因为编译的时候已经通过了，证明编译的时候是可以找到这个符号的，但是运行的时候没有找到，无非是so没有导入到apk里面，解压apk发现的确如此:

```shell
~/workspace/NDKDemo/app/build/outputs/apk/debug/app-debug/lib  tree
.
└── arm64-v8a
    └── libnative-lib.so

1 directory, 1 file
```

这种问题的原因在于jniLibs.srcDirs没有配置，我的so是放在app/src/main/cpp/jniLibs目录里面的，所以在build.gradle里面添加下面配置即可:

```groovy

android {
    ...
    sourceSets {
        main {
            jniLibs.srcDirs = ['src/main/cpp/jniLibs']
        }
    }
}
```

修改完之后满心欢喜的重新编译运行，立马啪啪打脸，依然找不到\_Z9getStringv

# 问题分析

## 疑点一: so仍未导入apk

难道是gradle配置没有起作用?解压apk之后发现libdemo.so是有导入的:

```shell
~/workspace/NDKDemo/app/build/outputs/apk/debug/app-debug/lib  tree .
.
└── arm64-v8a
    ├── libdemo.so
    └── libnative-lib.so

1 directory, 2 files
```

## 疑点二: so里面没有这个符号

难道是libdemo.so里面的确没有这个符号?我们可以用nm工具去查看so里面的符号。这个nm命令可以在ndk里面找到，最好找到对应cpu架构的目录下的工具。我编译的是arm64-v8a的so，可以用aarch64-linux-android下面的nm工具:

```shell
~/Library/Android/sdk/ndk/20.0.5594570  ./toolchains/aarch64-linux-android-4.9/prebuilt/darwin-x86_64/aarch64-linux-android/bin/nm  ~/workspace/NDKDemo/app/src/main/cpp/jniLibs/arm64-v8a/libdemo.so | grep getString
0000000000000538 T _Z9getStringv
```

输出显示没毛病，so里面的确是有_Z9getStringv这个符号的。

## 疑点三: 诡异的so依赖

其实之后我就在这里卡了很久，感觉哪里都对就结果不对。后面到处搜索也没有找到有人遇到类似的情况。后面是在用readelf分析发现它的依赖有些诡异:

```shell
~/Library/Android/sdk/ndk/20.0.5594570  ./toolchains/aarch64-linux-android-4.9/prebuilt/darwin-x86_64/aarch64-linux-android/bin/readelf -d ~/workspace/NDKDemo/app/build/outputs/apk/debug/app-debug/lib/arm64-v8a/libnative-lib.so

Dynamic section at offset 0xdd8 contains 26 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libnative-lib.so]
 0x0000000000000001 (NEEDED)             Shared library: [libm.so]
 0x0000000000000001 (NEEDED)             Shared library: [libdl.so]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so]
 0x000000000000000e (SONAME)             Library soname: [libnative-lib.so]
 ...
```

我们可以看到libnative-lib.so这个库它不但没有依赖libdemo.so，而且还依赖了它自己。

当时我就震惊了，还能有这种操作？

反复查看cmake配置的依赖配置，没有发现问题:

```cmake
cmake_minimum_required(VERSION 3.4.1)

add_library(native-lib SHARED native-lib.cpp)

add_library(demo SHARED IMPORTED)

set_target_properties(demo PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/jniLibs/${ANDROID_ABI}/libdemo.so)

target_link_libraries(native-lib  demo)
```

## 疑点四: 诡异的SONAME

我也卡了很久一直在cmake里面找原因，以为是编译libnative-lib.so的时候出了问题。后面实在没有头绪，无意中用readelf看了下libdemo.so:

```shell
~/Library/Android/sdk/ndk/20.0.5594570  ./toolchains/aarch64-linux-android-4.9/prebuilt/darwin-x86_64/aarch64-linux-android/bin/readelf -d ~/workspace/NDKDemo/app/build/outputs/apk/debug/app-debug/lib/arm64-v8a/libdemo.so

Dynamic section at offset 0xdf8 contains 25 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libm.so]
 0x0000000000000001 (NEEDED)             Shared library: [libdl.so]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so]
 0x000000000000000e (SONAME)             Library soname: [libnative-lib.so]
...
```

它的SONAME居然是libnative-lib.so，问题肯定就是出在这里了...

# so的几个名字

到了这一步，我们已经找到了问题的原因所在。但是要去解决它的话，我们还需要了解一些基础知识，这里也顺便普及下。so库的名字其实分三种realname、linkname和soname。

## realname

realname实际上就是so的文件名，一般格式为lib$(name).so.$(major).$(minor).$(revision)例如libcurl.so.4.5.0，我们可以在编译的时候用-o参数指定:

> gcc -shared -o $(realname) ...

## linkname

linkname是在链接时使用的,用-l参数指定例如下面的foo就是linkname。我们在这里不需要填so文件的名字，gcc会自动为linkname补上lib和.so，去链接lib$(name).so

> gcc main.c -L. -lfoo

## soname

soname顾名思义就是so的名字，它可以在编译的时候用−Wl,−soname,$(soname)指定，-Wl,表示后面的参数将传给link程序ld。如果不指定的话soname默认为realname:

```shell
gcc -shared -fPIC -Wl,-soname,libfoo.so.0 -o libfoo.so.0.0.0 foo.c
```

Soname会被记录在so的二进制数据中，我们可以用readelf命令查看:

```shell
readelf  -d libfoo.so.0.0.0

Dynamic section at offset 0xf18 contains 25 entries:
  标记        类型                         名称/值
 0x00000001 (NEEDED)                     共享库：[libc.so.6]
 0x0000000e (SONAME)                     Library soname: [libfoo.so.0]
 ...
```

那它有什么作用呢，我们可以做个试验:

```shell
$ gcc -shared -fPIC -Wl,-soname,libfoo.so.0 -o libfoo.so.0.0.0 foo.c
$ ln -s libfoo.so.0.0.0 libfoo.so
$ gcc main.c -L. -lfoo -o demo
$ ldd demo
        linux-vdso.so.1 (0xbece4000)
        /usr/lib/arm-linux-gnueabihf/libarmmem-${PLATFORM}.so => /usr/lib/arm-linux-gnueabihf/libarmmem-v7l.so (0xb6ef5000)
        libfoo.so.0 => not found
        libc.so.6 => /lib/arm-linux-gnueabihf/libc.so.6 (0xb6d8f000)
        /lib/ld-linux-armhf.so.3 (0xb6f0a000)

```

我们先编译了一个realname为libfoo.so.0.0.0，soname为libfoo.so.0的so库，然后创建一个软连接libfoo.so指向它，接着用foo这个linkname指定这个软链接去编译demo。

最后使用ldd查看demo的依赖，发现它依赖的是libfoo.so.0这个soname而不是编译的时候使用的libfoo.so。用readelf查看demo也能看到:

```shell
$ readelf -d demo

Dynamic section at offset 0xf10 contains 25 entries:
  标记        类型                         名称/值
 0x00000001 (NEEDED)                     共享库：[libfoo.so.0]
 0x00000001 (NEEDED)                     共享库：[libc.so.6]
...
```

也就是说在编译demo这个程序的时候，会通过linkname找到libfoo.so，它是个软链接实际指向libfoo.so.0.0.0，然后gcc会从libfoo.so.0.0.0里面读取soname写入demo的二进制信息。于是如果这个时候执行demo的话就会报找不到libfoo.so.0的问题:

```shell
$ ./demo
./demo: error while loading shared libraries: libfoo.so.0: cannot open shared object file: No such file or directory
```

# 问题原因

好了，现在回到我们的问题。最后我们分析到libdemo.so的soname居然是libnative-lib.so，那么原因很容易猜到就是−Wl,−soname指定错了。

查看编译记录的确是这个问题：由于新版本的ndk已经放弃gcc转向clang，我前段时间刚好换了电脑下载的是比较新的ndk，里面找不到熟悉的gcc了而我之前又没有用过clang。所以编译的指令是从android studio编译libnative-lib.so的日志里面拷贝修改的。它有很大一坨，又由于粗心，只改了-o 参数和.c文件，没有修改soname，然后问题就出现了。

然后这里还有一个坑，我一开始是直接报−Wl,−soname,libnative-lib.so这段给删掉了，因为使用gcc的时候如果没有指定，会自动把realname当做soname，但是clang不会。这个时候编译出来的so里面没有SONAME字段:

```shell
$ readelf -d libdemo.so

Dynamic section at offset 0xe08 contains 24 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libm.so]
 0x0000000000000001 (NEEDED)             Shared library: [libdl.so]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so]
 0x000000000000001a (FINI_ARRAY)         0x10df8
```

于是在运行的时候又会报找不到libdemo.so。也就是说在运行的时候查找依赖的原理是:从libnative-lib.so读到依赖libdemo.so，找到libdemo.so之后还会验证它的soname对不对，如果你只是realname为libdemo.so，soname不匹配也是不会去链接的。

最后将−Wl,−soname,libdemo.so加回上去问题解决。

事后回想了下，其实这种问题遇到的几率还是比较小的。因为如果c部分是我们自己写的，一般也就放到android stduio里面合成一个so。而如果需要导入外部的so一般也是用的第三方的，他们也很难出这种低级问题。就算像我这样的需求自己写个外部的so导入，干这活的一般也是个成熟的c/c++的程序员。也就我这种半桶水还啥都要自己干的苦逼会遇到。

