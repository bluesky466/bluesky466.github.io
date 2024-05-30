title: NDK交叉编译入门
date: 2024-05-30 20:28:50
tags:
    - 技术相关
    - Android
    - C/C++
---

最近协助同事交叉编译openssl,整理了下知识点准备出个交叉编译的入门教程。其实多年前写过一篇[写给安卓程序员的C/C++编译入门](https://blog.islinjw.cn/2018/07/29/%E5%86%99%E7%BB%99%E5%AE%89%E5%8D%93%E7%A8%8B%E5%BA%8F%E5%91%98%E7%9A%84cc-%E7%BC%96%E8%AF%91%E5%85%A5%E9%97%A8/)。但并没有一个具体的开源项目实例可能对新人来讲不够直观,这里就用openssl举例。

事情的背景是我们有个老项目之前只有armeabi-v7a的so现在想添加arm64-v8a的so,负责的同事不太熟悉交叉编译所以找到我帮忙。

如果我们有全部的源码那只需要在build.gradle的abiFilters里面添加`arm64-v8a`就可以让ndk去编译了:

```grovvy
android {
	...
    defaultConfig {
    	...
    	ndk {
            abiFilters 'armeabi-v7a','arm64-v8a'
        }
    	...
    }
    ...
}
```

但是这样直接编译我们那个项目编译失败了:

```
ld: error: /Users/linjw/workspace/Demo/Demo/src/main/cpp/openssl/lib/libcrypto.a(bio_lib.o) is incompatible with
aarch64linux
```

然后我去看了下原来这个项目直接依赖了编译好的armeabi-v7a的openssl的libcrypto.a和libssl.a,所以在编译arm64-v8a版本的so的时候链接armeabi-v7a的.a文件就出问题了,所以我们还需要先编译出arm64-v8a的libcrypto.a和libssl.a才行。

# 库版本

那怎么知道之前编译的openssl是哪个版本呢？正常的c或者c++第三方库除了静态库、动态库之外还会提供头文件,大部分的第三方库都会在头文件里面声明版本,所以我们可以在include目录下`grep -rn version .`搜索下,可以看到这里的版本是1.0.2t:

```c++
#  define OPENSSL_VERSION_TEXT    "OpenSSL 1.0.2t  10 Sep 2019"
```

有些第三方库还会生成.pc（package configure 配置文件）文件,刚好这个项目把它也保存了进去,在`openssl.pc`看到它的确是1.0.2t:

```
prefix=/usr/local/ssl/android-23
exec_prefix=${prefix}
libdir=${exec_prefix}/lib
includedir=${prefix}/include

Name: OpenSSL
Description: Secure Sockets Layer and cryptography libraries and tools
Version: 1.0.2t
Requires: libssl libcrypto
```

于是我们可以在openssl的[旧版本列表](https://ftp.openssl.org/source/old/1.0.2/)里面找到这个版本的源码。

# 交叉编译

编译第三方库的源码第一步应该看看他的文档,一般都会有编译的方法,例如从OpenSSL 1.0.2t源码的README里面就能找到我们应该去INSTALL里面看编译方法:

```
...
 See the appropriate file:
        INSTALL         Linux, Unix, etc.
        INSTALL.DJGPP   DOS platform with DJGPP
        INSTALL.NW      Netware
        INSTALL.OS2     OS/2
        INSTALL.VMS     VMS
        INSTALL.W32     Windows (32bit)
        INSTALL.W64     Windows (64bit)
        INSTALL.WCE     Windows CE
...
```

在`INSTALL`文档里面可以看到大概是用`Configure`去配置编译环境然后make install就好,用`./Configure -h`查看下它的文档,然后可以大概知道我们应该用`./Configure android64-aarch64 --prefix=/Users/linjw/Downloads/opensslbuild`去配置。


然后`make install`就会发现报错了:

```
making all in crypto...
gcc -I. -I.. -I../include  -DOPENSSL_THREADS -D_REENTRANT -DDSO_DLFCN -DHAVE_DLFCN_H -mandroid -fPIC -I/include -B/lib -O3 -Wall -DSHA1_ASM -DSHA256_ASM -DSHA512_ASM   -c -o cryptlib.o cryptlib.c
clang: error: unknown argument: '-mandroid'
make[1]: *** [cryptlib.o] Error 1
make: *** [build_crypto] Error 1
```

这里的报错是clang不认识`-mandroid`这个参数。但看的仔细点的同学可以看到它用的是gcc去编译的源码,实际编译的是本机的版本而不是安卓的版本。

交叉编译其实就是找到对应的编译器去对源码进行编译。这部分有时间可以看看[写给安卓程序员的C/C++编译入门](https://blog.islinjw.cn/2018/07/29/%E5%86%99%E7%BB%99%E5%AE%89%E5%8D%93%E7%A8%8B%E5%BA%8F%E5%91%98%E7%9A%84cc-%E7%BC%96%E8%AF%91%E5%85%A5%E9%97%A8/)介绍的会比较详细,这里简单讲一下。


例如可以直接用gcc或者clang编译出本机可用的目标文件(可执行程序、.a静态库、.so动态库等),也可以用ndk里面的aarch64-linux-android21-clang去编译出android21 arm64-v8a的目标文件,用android16 armv7a-linux-androideabi16-clang编译出armeabi-v7a的目标文件。

新版本的ndk都是在sdk里面携带的,可以在android studio里面选择版本下载:

{% img /NDK交叉编译入门/1.png %}


例如我这边安装的20.0.5594570的ndk就在`/Users/linjw/Library/Android/sdk/ndk/20.0.5594570`,可以在它的子目录使用make-standalone-toolchain.sh生成android23的交叉编译工具链:

```
export NDK_ROOT=/Users/linjw/Library/Android/sdk/ndk/20.0.5594570
export TOOLCHAIN_HOME=$HOME/Android/standalone-toolchains/android-toolchain-arm
export TOOLCHAIN_SYSROOT=$TOOLCHAIN_HOME/sysroot
export PATH=$PATH:$TOOLCHAIN_HOME/bin

$NDK_ROOT/build/tools/make-standalone-toolchain.sh --platform=android-23 --install-dir=$HOME/Android/standalone-toolchains/android-toolchain-arm  --toolchain=arm-linux-androideabi-4.9 --stl=gnustl
```

然后我们在`$TOOLCHAIN_HOME/bin`目录下就可以看到各种ABI的编译器:

```
2to3                               arm-linux-androideabi-objdump      i686-linux-android-ld.gold         ndk-stack
aarch64-linux-android-addr2line    arm-linux-androideabi-ranlib       i686-linux-android-nm              ndk-stack.py
aarch64-linux-android-ar           arm-linux-androideabi-readelf      i686-linux-android-objcopy         ndk-which
aarch64-linux-android-as           arm-linux-androideabi-size         i686-linux-android-objdump         pydoc
aarch64-linux-android-c++filt      arm-linux-androideabi-strings      i686-linux-android-ranlib          python
aarch64-linux-android-dwp          arm-linux-androideabi-strip        i686-linux-android-readelf         python-config
aarch64-linux-android-elfedit      armv7a-linux-androideabi16-clang   i686-linux-android-size            python-config.sh
aarch64-linux-android-gprof        armv7a-linux-androideabi16-clang++ i686-linux-android-strings         python2
aarch64-linux-android-ld           armv7a-linux-androideabi17-clang   i686-linux-android-strip           python2-config
aarch64-linux-android-ld.bfd       armv7a-linux-androideabi17-clang++ i686-linux-android16-clang         python2.7
aarch64-linux-android-ld.gold      armv7a-linux-androideabi18-clang   i686-linux-android16-clang++       python2.7-config
aarch64-linux-android-nm           armv7a-linux-androideabi18-clang++ i686-linux-android17-clang         sancov
aarch64-linux-android-objcopy      armv7a-linux-androideabi19-clang   i686-linux-android17-clang++       sanstats
aarch64-linux-android-objdump      armv7a-linux-androideabi19-clang++ i686-linux-android18-clang         scan-build
aarch64-linux-android-ranlib       armv7a-linux-androideabi21-clang   i686-linux-android18-clang++       scan-view
aarch64-linux-android-readelf      armv7a-linux-androideabi21-clang++ i686-linux-android19-clang         smtpd.py
aarch64-linux-android-size         armv7a-linux-androideabi22-clang   i686-linux-android19-clang++       x86_64-linux-android-addr2line
aarch64-linux-android-strings      armv7a-linux-androideabi22-clang++ i686-linux-android21-clang         x86_64-linux-android-ar
aarch64-linux-android-strip        armv7a-linux-androideabi23-clang   i686-linux-android21-clang++       x86_64-linux-android-as
aarch64-linux-android21-clang      armv7a-linux-androideabi23-clang++ i686-linux-android22-clang         x86_64-linux-android-c++filt
aarch64-linux-android21-clang++    armv7a-linux-androideabi24-clang   i686-linux-android22-clang++       x86_64-linux-android-dwp
aarch64-linux-android22-clang      armv7a-linux-androideabi24-clang++ i686-linux-android23-clang         x86_64-linux-android-elfedit
aarch64-linux-android22-clang++    armv7a-linux-androideabi26-clang   i686-linux-android23-clang++       x86_64-linux-android-gprof
aarch64-linux-android23-clang      armv7a-linux-androideabi26-clang++ i686-linux-android24-clang         x86_64-linux-android-ld
aarch64-linux-android23-clang++    armv7a-linux-androideabi27-clang   i686-linux-android24-clang++       x86_64-linux-android-ld.bfd
aarch64-linux-android24-clang      armv7a-linux-androideabi27-clang++ i686-linux-android26-clang         x86_64-linux-android-ld.gold
aarch64-linux-android24-clang++    armv7a-linux-androideabi28-clang   i686-linux-android26-clang++       x86_64-linux-android-nm
aarch64-linux-android26-clang      armv7a-linux-androideabi28-clang++ i686-linux-android27-clang         x86_64-linux-android-objcopy
aarch64-linux-android26-clang++    armv7a-linux-androideabi29-clang   i686-linux-android27-clang++       x86_64-linux-android-objdump
aarch64-linux-android27-clang      armv7a-linux-androideabi29-clang++ i686-linux-android28-clang         x86_64-linux-android-ranlib
aarch64-linux-android27-clang++    bisect_driver.py                   i686-linux-android28-clang++       x86_64-linux-android-readelf
aarch64-linux-android28-clang      clang                              i686-linux-android29-clang         x86_64-linux-android-size
aarch64-linux-android28-clang++    clang++                            i686-linux-android29-clang++       x86_64-linux-android-strings
aarch64-linux-android29-clang      clang-check                        idle                               x86_64-linux-android-strip
aarch64-linux-android29-clang++    clang-format                       ld.lld                             x86_64-linux-android21-clang
arm-linux-androideabi-addr2line    clang-tidy                         llvm-ar                            x86_64-linux-android21-clang++
arm-linux-androideabi-ar           clang-tidy.real                    llvm-as                            x86_64-linux-android22-clang
arm-linux-androideabi-as           clang80                            llvm-config                        x86_64-linux-android22-clang++
arm-linux-androideabi-c++filt      clang80++                          llvm-cov                           x86_64-linux-android23-clang
arm-linux-androideabi-clang        gdb                                llvm-dis                           x86_64-linux-android23-clang++
arm-linux-androideabi-clang++      gdb-orig                           llvm-link                          x86_64-linux-android24-clang
arm-linux-androideabi-dwp          git-clang-format                   llvm-modextract                    x86_64-linux-android24-clang++
arm-linux-androideabi-elfedit      i686-linux-android-addr2line       llvm-nm                            x86_64-linux-android26-clang
arm-linux-androideabi-g++          i686-linux-android-ar              llvm-objcopy                       x86_64-linux-android26-clang++
arm-linux-androideabi-gcc          i686-linux-android-as              llvm-profdata                      x86_64-linux-android27-clang
arm-linux-androideabi-gprof        i686-linux-android-c++filt         llvm-readobj                       x86_64-linux-android27-clang++
arm-linux-androideabi-ld           i686-linux-android-dwp             llvm-strip                         x86_64-linux-android28-clang
arm-linux-androideabi-ld.bfd       i686-linux-android-elfedit         llvm-symbolizer                    x86_64-linux-android28-clang++
arm-linux-androideabi-ld.gold      i686-linux-android-gprof           make                               x86_64-linux-android29-clang
arm-linux-androideabi-nm           i686-linux-android-ld              ndk-gdb                            x86_64-linux-android29-clang++
arm-linux-androideabi-objcopy      i686-linux-android-ld.bfd          ndk-gdb.py                         yasm
```

不同cpu架构对应的工具链如下:

|ABI|工具链|
|-|-|
|armeabi-v7a|armv7a-linux-androideabi|
|arm64-v8a|aarch64-linux-android|
|x86|i686-linux-android|
|x86-64|x86_64-linux-android|

所以我们想要编译出arm64-v8a的库就需要指定`aarch64-linux-`的这堆工具去编译openssl的源码,但具体要怎么指定?

其实C/C++的编译有一些约定俗成的环境变量,我们可以用设置这些环境变量去指定编译工具:


```shell
export CC=aarch64-linux-android23-clang
export CXX=aarch64-linux-android23-clang++
export AR=aarch64-linux-android-ar
export AS=aarch64-linux-android-as
export RANLIB=aarch64-linux-android-ranlib
export LD=aarch64-linux-android-ld
export STRIP=aarch64-linux-android-strip
```

他们的作用如下:

- CC : c语言编译器
- CXX : C++编译器
- AR : 将.o文件打包成.a库
- AS : 将汇编代码转化为.o文件
- RANLIB : 为静态库文件生成索引,优化链接速度
- STRIP : 剥掉目标文件中一些符号信息和调试信息,使文件变小
- LD : 链接目标文件生成可执行文件

设置好了之后重新`./Configure android64-aarch64 --prefix=/Users/linjw/Downloads/opensslbuild`再`make install`会发现还是报错,但是可以看到编译器的确已经变成了`aarch64-linux-android23-clang`:

```
making all in crypto...
/usr/bin/perl ../util/mkbuildinf.pl "aarch64-linux-android23-clang -I. -I.. -I../include  -DOPENSSL_THREADS -D_REENTRANT -DDSO_DLFCN -DHAVE_DLFCN_H -mandroid -fPIC -I/include -B/lib -O3 -Wall -DSHA1_ASM -DSHA256_ASM -DSHA512_ASM" "android64-aarch64" >buildinf.h
aarch64-linux-android23-clang -I. -I.. -I../include  -DOPENSSL_THREADS -D_REENTRANT -DDSO_DLFCN -DHAVE_DLFCN_H -mandroid -fPIC -I/include -B/lib -O3 -Wall -DSHA1_ASM -DSHA256_ASM -DSHA512_ASM   -c -o cryptlib.o cryptlib.c
clang80: error: unknown argument: '-mandroid'
make[1]: *** [cryptlib.o] Error 1
make: *** [build_crypto] Error 1
```

旧的ndk版本使用gcc去编译需要这个参数,新的ndk版本转向clang之后的确就不认识这个参数,因为我们都有源码了,直接修改下Configure把`android64-aarch64`目标的`-mandroid`参数去掉重新执行Configure和编译就好。`make install`运行完之后就可以在`/Users/linjw/Downloads/opensslbuild`下看到编译好的产物了。


# openssl 静态库链接异常

虽然openssl arm64-v8a的库编译好了,但是实际放到项目里面去编译的时候出现了下面的链接异常:

```
ld: error: relocation R_AARCH64_PREL64 cannot be used against symbol OPENSSL_armcap_P; recompile with -fPIC
>>> defined in /Users/linjw/workspace/Demo/Demo/src/main/cpp/openssl/lib/libcrypto.a(armcap.o)
>>> referenced by sha1-armv8.o:(.text+0x1240) in archive /Users/linjw/workspace/Demo/Demo/src/main/cpp/openssl/lib/libcrypto.a
```

从编译提示和源码可以看到,在sha1-armv8.pl里面使用定义在armcap.c的OPENSSL\_armcap\_P的时候出现依赖异常:

```
// crypto/armcap.c
unsigned int OPENSSL_armcap_P = 0;

# crypto/sha/asm
ldr x16,.LOPENSSL_armcap_P
```

根据提示需要用`-fPIC`参数重新编译,但是我看无论是编译libcrypto.a还是libssl.a或者我们的项目都是有带的。

这种情况按道理应该有人遇到才对,搜索了下仓库的issues,发现其实之前就有人报过[问题](https://github.com/openssl/openssl/issues/10842),然后可以看到这个问题已经在1.1.1i上解决了:

> This should be fixed on both branch 1.1.1 (and incoming release 1.1.1i) and master now.

具体的修改在[这](https://github.com/openssl/openssl/compare/OpenSSL_1_1_1-stable%E2%80%A6Romain-Geissler-1A:OpenSSL_1_1_1-stable),我本来是想直接pick过来用的,但是因为这个版本代码架构和1.0.2t差异有点大,而且perl超出了我的知识边界也不确定会不会改出问题来,所以还是决定更新到1.1.1i的版本。在[列表](https://ftp.openssl.org/source/old/1.1.1/)找到对应的源码下载下来执行`./Configure android64-aarch64 --prefix=/Users/linjw/Downloads/opensslbuild`发现会报错:

```
Configuring OpenSSL version 1.1.1i (0x1010109fL) for android64-aarch64
Using os-specific seed configuration

Failure!  build file wasn't produced.
Please read INSTALL and associated NOTES files.  You may also have to look over
your available compiler tool chain or change your configuration.

$ANDROID_NDK_HOME is not defined at (eval 10) line 32.
```

看起来是编译方式改变了,从提示看需要配置ANDROID\_NDK\_HOME,我们直接在文档(NOTES.ANDROID)里搜索下它可以看到的确简化了安卓的编译流程:

```
...
    export ANDROID_NDK_HOME=/home/whoever/Android/android-sdk/ndk/20.0.5594570
    PATH=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin:$ANDROID_NDK_HOME/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin:$PATH
    ./Configure android-arm64 -D__ANDROID_API__=29
    make

 Older versions of the NDK have GCC under their common prebuilt tools directory, so the bin path
 will be slightly different. EG: to compile for ICS on ARM with NDK 10d:

    export ANDROID_NDK_HOME=/some/where/android-ndk-10d
    PATH=$ANDROID_NDK_HOME/toolchains/arm-linux-androideabi-4.8/prebuilt/linux-x86_64/bin:$PATH
    ./Configure android-arm -D__ANDROID_API__=14
    make
...
```

我的开发笔记本是MAC,编译配置如下:

```
export ANDROID_NDK_HOME=/Users/linjw/Library/Android/sdk/ndk/20.0.5594570
export PATH=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/darwin-x86_64/bin:$ANDROID_NDK_HOME/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/bin:$PATH
./Configure android-arm64 -D__ANDROID_API__=23 --prefix=/Users/linjw/Downloads/opensslbuild
make install
```

编译完成就能在`/Users/linjw/Downloads/opensslbuild`找到编译的产物了。

# 其他问题

### gradle config异常

项目clone下来之后编译它gradle出现了异常,添加`--stacktrace`参数编译之后看到下面的堆栈:

```
Caused by: java.lang.NullPointerException
        at com.google.common.base.Preconditions.checkNotNull(Preconditions.java:782)
        at com.android.build.gradle.internal.ndk.NdkHandler.getPlatformVersion(NdkHandler.java:158)
        at com.android.build.gradle.internal.ndk.NdkHandler.supports64Bits(NdkHandler.java:331)
        at com.android.build.gradle.internal.ndk.NdkHandler.getSupportedAbis(NdkHandler.java:397)

```

其实就是因为旧版本的gradle支持不了新版本的ndk的编译,所以就只需要更新下`gradle-wrapper.properties`里的版本就好,例如我更新到了6.7.1:

```
#Tue May 18 15:09:16 CST 2021
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-6.7.1-bin.zip
```

### 多ABI编译支持

编译出来了之后我们还需要修改项目的CMakeLists.txt配置让他能同时编译'armeabi-v7a','arm64-v8a'两种架构的so,可以用ANDROID\_ABI变量控制编译不同ABI的so的时候依赖不同的库:

```
set(OPENSSL_CRYPTO_LIBRARY "${CMAKE_SOURCE_DIR}/openssl/${ANDROID_ABI}/lib/libcrypto.a")
set(OPENSSL_INCLUDE_DIR "${CMAKE_SOURCE_DIR}/openssl/${ANDROID_ABI}/include")
set(OPENSSL_SSL_LIBRARY "${CMAKE_SOURCE_DIR}/openssl/${ANDROID_ABI}/lib/libcrypto.a;${CMAKE_SOURCE_DIR}/openssl/${ANDROID_ABI}/lib/libs
sl.a")
```

### stderr undefined问题

在导入arm64-v8a的openssl静态库编译项目之后其实在编译链接的时候其实除了前面讲的`OPENSSL_armcap_P`问题,还出现了stderr undefined问题:

```
ld: error: undefined symbol: stderr
>>> referenced by cryptlib.c
>>>               cryptlib.o:(OPENSSL_showfatal) in archive /Users/linjw/workspace/Demo/Demo/src/main/cpp/openssl/a
rm64-v8a/lib/libcrypto.a
```

由于.a是可以编译成功的,所以起码在编译静态库的时候是能找到stderr定义的,我们随便在cryptlib.c加个`int stderr = 1;`去编译让他出现重定义问题来确认stderr是在哪里定义的:

```
making all in crypto...
aarch64-linux-android23-clang -I. -I.. -I../include  -DOPENSSL_THREADS -D_REENTRANT -DDSO_DLFCN -DHAVE_DLFCN_H -fPIC -I/include -B/lib -O3 -Wall -DSHA1_ASM -DSHA256_ASM -DSHA512_ASM   -c -o cryptlib.o cryptlib.c
cryptlib.c:1039:5: error: redefinition of 'stderr' with a different type: 'int' vs 'FILE *' (aka 'struct __sFILE *')
int stderr = 1;
    ^
/Users/linjw/Android/standalone-toolchains/android-toolchain-arm/bin/../sysroot/usr/include/stdio.h:69:16: note: expanded from macro 'stderr'
#define stderr stderr
               ^
/Users/linjw/Android/standalone-toolchains/android-toolchain-arm/bin/../sysroot/usr/include/stdio.h:64:14: note: previous declaration is here
extern FILE* stderr __INTRODUCED_IN(23);
```

可以看到它实际是定义在ndk的stdio.h里面的:

```c++
#if __ANDROID_API__ >= __ANDROID_API_M__
extern FILE* stdin __INTRODUCED_IN(23);
extern FILE* stdout __INTRODUCED_IN(23);
extern FILE* stderr __INTRODUCED_IN(23);

/* C99 and earlier plus current C++ standards say these must be macros. */
#define stdin stdin
#define stdout stdout
#define stderr stderr
#else
/* Before M the actual symbols for stdin and friends had different names. */
extern FILE __sF[] __REMOVED_IN(23);

#define stdin (&__sF[0])
#define stdout (&__sF[1])
#define stderr (&__sF[2])
#endif
```

\_\_ANDROID\_API\_M\_\_的值为23,而我们编译openssl的时候定义的android版本刚好是23:

```
./android/api-level.h:83:#define __ANDROID_API_M__ 23
```

所以最终stderr的定义如下:

```c++
extern FILE* stderr __INTRODUCED_IN(23);
#define stderr stderr
```

而在链接libcrypto.a编译so的时候找不到stderr,意味着它并没有定义`extern FILE* stderr __INTRODUCED_IN(23);`,而是走了下面的:

```c++
#else
/* Before M the actual symbols for stdin and friends had different names. */
extern FILE __sF[] __REMOVED_IN(23);

#define stdin (&__sF[0])
#define stdout (&__sF[1])
#define stderr (&__sF[2])
#endif
```

因为预处理宏定义在编译的最前头,并不会修改.a里面的符号(见[C/C++的编译流](https://blog.islinjw.cn/2018/07/29/%E5%86%99%E7%BB%99%E5%AE%89%E5%8D%93%E7%A8%8B%E5%BA%8F%E5%91%98%E7%9A%84cc-%E7%BC%96%E8%AF%91%E5%85%A5%E9%97%A8/#C-C-%E7%9A%84%E7%BC%96%E8%AF%91%E6%B5%81%E7%A8%8B))所以并不会将.a里面的stderr修改成`(&__sF[2])`,所以就找不到stderr的符号定义了。

所以去看项目build.gradle里的minSdkVersion发现它是21:

```grovvy
android {
    ...
    defaultConfig {
        minSdkVersion 21
        ...
    }
    ...
}
```

于是在编译项目的时候`__ANDROID_API__`的值就是21小于\_\_ANDROID\_API\_M\_\_(23)。

所以修改也很简单,把minSdkVersion改成和编译openssl时的23或者用21去重新编译openssl就可以了。