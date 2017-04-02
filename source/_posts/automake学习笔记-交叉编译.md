title: automake学习笔记 - 交叉编译
date: 2017-04-02 11:05:27
tags:
	- 技术相关
	- 编译相关
---

## 什么是交叉编译

很多时候我们因为各种原因需要在一个平台上编译其他平台的程序。如在linux或者windows上编译可以在安卓使用的so库、apk等。在linux上编译windows的dll或者exe等。

这种在某个系统平台下可以产生另一个系统平台的可执行文件的技术就叫做交叉编译。

## 使用automake进行交叉编译

automake就提供了交叉编译的功能，但是它的[官方文档](https://www.gnu.org/software/automake/manual/html_node/Cross_002dCompilation.html)十分的简单:

> 2.2.8 Cross-Compilation
>
> To cross-compile is to build on one platform a binary that will run on another platform. When speaking of cross-compilation, it is important to distinguish between the build platform on which the compilation is performed, and the host platform on which the resulting executable is expected to run. The following configure options are used to specify each of them:
>
> --build=build
The system on which the package is built.
>
> --host=host
The system where built programs and libraries will run.
>
> When the --host is used, configure will search for the cross-compiling suite for this platform. Cross-compilation tools commonly have their target architecture as prefix of their name. For instance my cross-compiler for MinGW32 has its binaries called i586-mingw32msvc-gcc, i586-mingw32msvc-ld, i586-mingw32msvc-as, etc.
>
> Here is how we could build amhello-1.0 for i586-mingw32msvc on a GNU/Linux PC.
>
> ~/amhello-1.0 % ./configure --build i686-pc-linux-gnu --host i586-mingw32msvc
checking for a BSD-compatible install... /usr/bin/install -c
checking whether build environment is sane... yes
checking for gawk... gawk
checking whether make sets $(MAKE)... yes
checking for i586-mingw32msvc-strip... i586-mingw32msvc-strip
checking for i586-mingw32msvc-gcc... i586-mingw32msvc-gcc
checking for C compiler default output file name... a.exe
checking whether the C compiler works... yes
checking whether we are cross compiling... yes
checking for suffix of executables... .exe
checking for suffix of object files... o
checking whether we are using the GNU C compiler... yes
checking whether i586-mingw32msvc-gcc accepts -g... yes
checking for i586-mingw32msvc-gcc option to accept ANSI C...
…
~/amhello-1.0 % make
…
~/amhello-1.0 % cd src; file hello.exe
hello.exe: MS Windows PE 32-bit Intel 80386 console executable not relocatable
The --host and --build options are usually all we need for cross-compiling. The only exception is if the package being built is itself a cross-compiler: we need a third option to specify its target architecture.
>
> --target=target
When building compiler tools: the system for which the tools will create output.
>
> For instance when installing GCC, the GNU Compiler Collection, we can use --target=target to specify that we want to build GCC as a cross-compiler for target. Mixing --build and --target, we can actually cross-compile a cross-compiler; such a three-way cross-compilation is known as a Canadian cross.
>
> See Specifying the System Type in The Autoconf Manual, for more information about these configure options.

简单的来说就是通过在执行configure的时候通过传入下面三个参数进行配置,然后和普通的编译一样使用make命令就能编译出指定平台的程序

- --build

编译工程的平台

- --host

编译出来的程序或者库需要运行的平台

- --target

当构建编译器时,指定该编译器编译的程序的运行平台

但是看例子，我们指定平台却并不是简单的指定windows、android这么简单。

> ~/amhello-1.0 % ./configure --build i686-pc-linux-gnu --host i586-mingw32msvc

这里的 i686-pc-linux-gnu和 i586-mingw32msvc-gcc其实指的是一系列的编译工具。交叉编译工具的命名其实是有一定的格式的。 例如，用来编译windows程序的MinGW32的交叉编译器的二进制文件叫做i586-mingw32msvc-gcc，i586-mingw32msvc-ld，i586-mingw32msvc-as等。

> MinGW是Minimalist GNU for Windows的意思，又称mingw32，是将GCC编译器和GNU Binutils移植到Win32平台下的产物，包括一系列头文件（Win32API）、库和可执行文件。-- [维基百科](https://zh.wikipedia.org/wiki/MinGW)

其实就是我们指定了编译工具的前缀，然后automake就会更加这个前缀，找到对应的编译器去编译我们的程序。

## 在linux上编译windows上的程序

有人可能会问，为什么需要在linux上编译这么蛋疼而不直接在windows上编译呢？

就按我遇到的情况来说吧。我们部门的自动构建服务器就是liunx的，我们的项目都需要通过它来编译、检查和发布，我们也习惯于子啊linux上编程，最重要的是我们的项目就是跨平台的，不管是windows、linux还是android上都需要可以运行，所以没有必要为每个平台搭建一套编译环境。直接在linux上编译所有平台的软件是最好的选择。

为了在linux上交叉编译windows的程序，我们先要搭建一下交叉编译的环境:

1. 安装交叉编译工具

```
sudo apt-get install mingw-w64
```

2. 更新配置，使用 posix thread

```
sudo update-alternatives --config i686-w64-mingw32-g++
sudo update-alternatives --config i686-w64-mingw32-gcc
sudo update-alternatives --config x86_64-w64-mingw32-g++
sudo update-alternatives --config x86_64-w64-mingw32-gcc
```

上面的选项中，选择 posix 版本。

> 可移植操作系统接口（英语：Portable Operating System Interface of UNIX，缩写为POSIX），是IEEE为要在各种UNIX操作系统上运行软件，而定义API的一系列互相关联的标准的总称，其正式称呼为IEEE Std 1003，而国际标准名称为ISO/IEC 9945。此标准源于一个大约开始于1985年的项目。POSIX这个名称是由理查德·斯托曼应IEEE的要求而提议的一个易于记忆的名称。它基本上是Portable Operating System Interface（可移植操作系统接口）的缩写，而X则表明其对Unix API的传承。
Linux基本上逐步实现了POSIX兼容，但并没有参加正式的POSIX认证。[1]
微软的Windows NT声称部分实现了POSIX标准。[2]
当前的POSIX主要分为四个部分[3]：Base Definitions、System Interfaces、Shell and Utilities和Rationale。 -- [维基百科](https://zh.wikipedia.org/wiki/POSIX)

按道理这个时候就可以在build目录执行下面的命令去编译了

```
../configure --prefix=`pwd` --host i686-w64-mingw32
```

但是执行了configure之后却会报下面的错误:

> libtool: warning: undefined symbols not allowed in i686-w64-mingw32 shared libraries; building static only

本来我们的工程是需要编译动态库的，但是如果报了这个错误，就会编出静态库来，最终install之后在bin目录下面只有一个 __example.exe__ ，而没有dll。解决方法是在src/Makefile.am中加上

> libeasylog_la_LDFLAGS = -no-undefined

这样编译安装之后就能在bin目录下看到 __example.exe__ 和 __libeasylog-0.dll__ 了

这个时候将这两个东西拷贝到windows平台上去，记得它们需要在同级目录这样 __example.exe__ 才能找到 __libeasylog-0.dll__。然后在控制台中运行 __example.exe__ 就会报下面的错误,其实就是还有几个dll没有找到：

> 无法启动此程序,因为计算机中丢失libstdc++-6.dll。尝试重新安装该程序以解决此问题。  
>
> 无法启动此程序,因为计算机中丢失libgcc_s_sjlj-1.dll。尝试重新安装该程序以解决此问题。

我们到下面的目录把缺的dll也拷贝到example的同级目录

> /usr/lib/gcc/i686-w64-mingw32/5.3-posix/

再次运行发现有报了下面的错误:

> 无法启动此程序,因为计算机中丢失libwinpthread-1.dll。尝试重新安装该程序以解决此问题。

这个dll可以到下面的目录拷贝，同样放到example的同级目录，之后再运行example:

> [test] testlog

看已经成功运行了。

## 在linux上编译安卓上的程序

搭建安卓的交叉编译环境就是生成 standalone toochain

首先下载NDK，解压，假设NDK的根目录为NDK_ROOT，然后执行

```
sudo $NDK_ROOT/build/tools/make-standalone-toolchain.sh \
     --platform=android-19 \
     --install-dir=$HOME/Android/standalone-toolchains/android-toolchain-arm \
     --toolchain=arm-linux-androideabi-4.9 \
     --stl=gnustl
```

最后配置环境变量

```
export NDK_ROOT=$HOME/Android/android-ndk-r13b
export TOOLCHAIN_HOME=$HOME/Android/standalone-toolchains/android-toolchain-arm
export TOOLCHAIN_SYSROOT=$TOOLCHAIN_HOME/sysroot
export PATH=$PATH:$TOOLCHAIN_HOME/bin
```

环境搭建好之后在build目录中执行

```
../configure --prefix=`pwd` --host arm-linux-androideabi
```

之后就能使用make install命令编译并安装了。完成后再bin目录可以见到 __example__ ,在lib目录可以看到 __libeasylog.so__。

虽然看起来和linux程序一样，但是直接运行example的话会报错:

> zsh: 可执行文件格式错误: ./example

因为它的运行环境是安卓，在本机(Ubuntu)上不能运行

如果你有一台root了的安卓机器的话，可以使用adb将example给push到/system/bin，将libeasylog.so给push到/system/lib。这样就能在adb shell中使用example命令得到下面的输出了:

> [test] testlog

当然，做应用的一般都不会直接编译出可执行程序来给安卓使用。更多的是编译出so来给apk通过jni调用c/c++的方法。但是编译的过程和这里是一样的，关于jni的使用我之后会另写一篇文章来讨论。

## Demo项目

可以在[这里](https://github.com/bluesky466/automake-demo/tree/v0.0.4)查看完整的项目代码
