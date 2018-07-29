title: 写给安卓程序员的C/C++编译入门
date: 2018-07-29 19:01:45
tags:
    - 技术相关
    - Android
    - C/C++
---

最近部门新入职了几个小鲜肉,打算给他们分享下一些C/C++编译的基础知识,于是整理了一些资料写了这篇博客.由于已经有差不多一年没有写c++了,可能会有一些不太正确的地方,希望哪位同学看到能够帮忙指出,免得误人子弟.

首先需要声明的是,我用的是Ubuntu系统,也是基于Linux去讲的,当然大家如果是用的Mac系统,其实可以无缝切换,用几乎完全一样的命令去跑.但是如果是Windows的同学,可能就不太适用了.

不过其实我还是鼓励大家用Linux系统或者Mac系统去编译C/C++程序.因为大多数流行库都是在linux下面写的,使用Linux或者Mac交叉编译出安卓的可用程序都比较方便.

# 为什么要学C/C++编译

很多的安卓程序员可能都会用Android Studio写一些简单的C/C++代码,然后通过jni去调用,但是对C/C++是如何编译的其实并没有什么概念.有人可能会问,为什么安卓程序员会需要了解C/C++是如何编译的呢?我一直都认为,要成为一个真正的高级安卓应用开发工程师,安卓源码和C/C++是两座绕不过的大山.安卓源码自然不必多说,而C/C++流行了几十年,存在着许多优秀的开源项目,我们在处理一些特定的需求的时候,可能会需要使用到它们.如脚本语言Lua,计算机视觉库OpenCV,音视频编解码库ffmpeg,谷歌的gRPC,国产游戏引擎Cocos2dx...有些库提供了完整的安卓接口,有些提供了部分安卓接口,有些则没有.在做一些高级功能时,我们常常需要使用源码,通过裁剪和交叉编译,才能编译出可以在安卓上使用的so库.总之,安卓做深做精总避不开C/C++交叉编译.

# C/C++编译器

类似java编译器javac可以将java代码编译成class文件,C/C++也有gcc、g++、clang等多种编译器可以用于编译C/C++代码.这里我们用gcc来举例.

gcc原名为GNU C 语言编译器(GNU C Compiler),因为它原本只能处理C语言.但GCC很快地扩展,变得可处理C++。后来又扩展能够支持更多编程语言,如Fortran、Pascal、Objective-C、Java、Ada、Go以及各类处理器架构上的汇编语言等,所以改名GNU编译器套件(GNU Compiler Collection).

我这篇文章的例子都是Ubuntu上编译的.使用Ubuntu系统的同学可以使用下面命令安装gcc:

> sudo apt-get install gcc

如果是CentOS使用yum去安装:

> yum install gcc

Mac系统的话可以用HomeBrew来安装,HomeBrew的安装方法我就不说了,大家可以自己搜索:

> brew install gcc

而使用Windows的同学,需要自己搜索下MinGw是如何安装的,MinGw 是 Minimal GNU on Windows 的缩写.

使用gcc其实只需要一个命令就能将一个c文件编译成可运行程序了:

> gcc test.c -o test

通过上面这条命令可以将test.c编译成可运行程序test.但是其实C/C++的编译是经过了好几个步骤的,我这边先给大家大概的讲一讲.

# C/C++的编译流程

C/C++的编译可以分为下面几个步骤:

{% img /写给安卓程序员的cc-编译入门/1.png %}

## 预处理

相信学过C/C++的同学都知道"宏"这个东西,它在编译的时候会被展开替换成实际的代码,这个展开的步骤就是在预处理的时候进行的.当然,预处理并不仅仅只是做宏的展开,它还做了类似头文件插入、删除注释等操作.

预处理之后的产品依然还是C/C++代码,它在代码的逻辑上和输入的C/C++源代码是完全一样的.

我们来举一个简单的例子,写一个test.h文件和一个test.c文件:

```
//test.h
#ifndef TEST_H            
#define TEST_H

#define A 1     
#define B 2        

/**
 * add 方法的声明
 */               
int add(int a, int b);

#endif
```

```
//test.c
#include "test.h"

/**
 * add 方法定义
 */

int add(int a, int b) {
    return a + b;
}

int main(int argc,char* argv[]) {
    add(A, B);
    return 0;                 
}
```

然后可以通过下面这个gcc命令预处理test.c文件,并且把预处理结果写到test.i:

> gcc -E test.c -o test.i

然后就能看到预处理之后的test.c到底长什么样子了:

```
# 1 "test.c"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 1 "<command-line>" 2
# 1 "test.c"
# 1 "test.h" 1
# 11 "test.h"
int add(int a, int b);
# 2 "test.c" 2

int add(int a, int b){
 return a + b;
}

int main(int argc,char* argv[]) {
 add(1, 2);
 return 0;
}
```

可以看到这里它把test.h的内容(add方法的声明)插入到了test.c的代码中,然后将A、B两个宏展开成了1和2,将注释去掉了,还在头部加上了一些信息.

但是光看代码逻辑,和之前我们写的代码是完全一样的.

## 汇编

可能大家都听过汇编语言这个东西,但是年轻一点的同学不一定真正见过.简单来说汇编语言是将机器语言符号化了的语言,是机器不能直接识别的低级语言.我们可以通过下面的命令,将预处理后的代码编译成汇编语言:

> gcc -S test.i -o test.s

然后就能看到生成的test.s文件了,里面就是我们写的c语言代码翻译而成的汇编代码:

```
        .file   "test.c"
        .text
        .globl  add
        .type   add, @function
add:
.LFB0:
        .cfi_startproc
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        movl    %edi, -4(%rbp)
        movl    %esi, -8(%rbp)
        movl    -4(%rbp), %edx
        movl    -8(%rbp), %eax
        addl    %edx, %eax
        popq    %rbp
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
.LFE0:
        .size   add, .-add
        .globl  main
        .type   main, @function
main:
.LFB1:
        .cfi_startproc
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        subq    $16, %rsp
        movl    %edi, -4(%rbp)
        movq    %rsi, -16(%rbp)
        movl    $2, %esi
        movl    $1, %edi
        call    add
        movl    $0, %eax
        leave
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
.LFE1:
        .size   main, .-main
        .ident  "GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.10) 5.4.0 20160609"
        .section        .note.GNU-stack,"",@progbits
```

## 汇编

汇编这一步是将汇编代码编译成机器语言:

> gcc -c test.s -o test.o

生成的test.o文件里面就是机器代码了,我们可以通过nm命令来列出test.o里面的符号:

> nm test.o

得到的结果如下:

```
0000000000000000 T add
0000000000000014 T main
```

## 链接

由于我们的例子代码比较简单只有一个test.h和test.h,所以只生成了一个.o文件,其实一般的程序都是由多个模块组合成的.链接这一步就是将多个模块的代码组合成一个可执行程序.我们可以用gcc命令将多个.o文件或者静态库、动态库链接成一个可执行文件:

> gcc test.o -o test

得到的就是可执行文件test了,可以直接用下面命令运行

> ./test

当然是没有任何输出的,因为我们就没有做任何的打印

## 编译so库

当然,在安卓中我们一般不会直接使用C/C++编译出来的可运行文件.用的更多的应该是so库.那要如何编译so库呢?

首先我们需要将test.c中的main函数去掉,因为so库中是不会带有main函数的:

```
#include "test.h"

/**
 * add 方法定义
 */
int add(int a, int b){
        return a + b;
}
```

然后可以使用下面命令将test.c编译成test.so:

> gcc -shared test.c -o test.so

其实也就是多了个-shared参数,指定编译的结果为动态链接库.

这里是直接将.c文件编译成so,当然也能像之前的例子一样先编译出.o文件再通过链接生成so文件.

当然一般编译动态链接库,我们还会带上-fPIC参数.

fPIC (Position-Independent Code)告诉编译器产生与位置无关代码,即产生的代码中没有绝对地址,全部使用相对地址.故而代码可以被加载器加载到内存的任意位置,都可以正确的执行.不加fPIC编译出来的so,是要再加载时根据加载到的位置再次重定位的.因为它里面的代码并不是位置无关代码.如果被多个应用程序共同使用,那么它们必须每个程序维护一份.so的代码副本了.因为.so被每个程序加载的位置都不同,显然这些重定位后的代码也不同,当然不能共享.

# 交叉编译

通过上面的例子,我们知道了一个C/C++程序是怎么从源代码一步步编译成可运行程序或者so库的.但是我们编译出来的程序或者so库只能在相同系统的电脑上使用.

例如我使用的电脑是Linux系统的,那它编译出来的程序也就只能在Linux上运行,不能在安卓或者Windows上运行.

当然正常情况下不会有人专门去到android系统下编译出程序来给安卓去用.一般我们都是在PC上编译出安卓可用的程序,在给到安卓去跑的.这种是在一个平台上生成另一个平台上的可执行代码的编译方式就叫做交叉编译.

交叉编译有是三个比较重要的概念要先说明一下:

- build : 当前你使用的计算机
- host : 你的目的是编译出来的程序可以在host上运行
- target : 普通程序没有这个概念。对于想编译出编译器的人来说此属性决定了新编译器编译出的程序可以运行在哪

如果我们想要交叉编译出安卓可运行的程序或者库的话就不能直接使用gcc去编译了.而需要使用Android NDK提供了的一套交叉编译工具链.

我们首先要下载Android NDK,然后配置好环境变量NDK_ROOT指向NDK的根目录.

然后可以通过下面命令安装交叉编译工具链:

> $NDK_ROOT/build/tools/make-standalone-toolchain.sh \
	--platform=android-19 \
	--install-dir=$HOME/Android/standalone-toolchains/android-toolchain-arm \
	--toolchain=arm-linux-androideabi-4.9 \
	--stl=gnustl

然后我们就能在$HOME/Android/目录下看到安装好的工具链了.进到$HOME/Android/standalone-toolchains/android-toolchain-arm/bin/目录下我们可以看到有arm-linux-androideabi-gcc这个程序.

它就是gcc的安卓交叉编译版本.我们将之前使用gcc去编译的例子全部换成使用它去编译就能编译出运行在安卓上的程序了:

如下面命令生成的so库就能在安卓上通过jni调用了:

>  $HOME/Android/standalone-toolchains/android-toolchain-arm/bin/arm-linux-androideabi-gcc -shared -fPIC test.c -o test.so

我们会将定义下面几个环境变量,将$HOME/Android/standalone-toolchains/放到PATH变量中,这样就可以直接使用arm-linux-androideabi-gcc命令,而不需要输入它的全路径去使用了:

```
export TOOLCHAIN_HOME=$HOME/Android/standalone-toolchains/android-toolchain-arm
export TOOLCHAIN_SYSROOT=$TOOLCHAIN_HOME/sysroot
export PATH=$PATH:$TOOLCHAIN_HOME/bin
```

设定好之后可以直接用下面命令去编译:

```
arm-linux-androideabi-gcc -shared -fPIC test.c -o test.so
```

## 不同CPU架构的编译方式

当然安卓也有很多不同的CPU架构,不同CPU架构的程序也是不一定兼容的,相信大家之前在使用Android Studio去编译so的时候也有看到编译出来的库有很多个版本像armeabi、armeabi-v7a、mips、x86等.

那这些不同CPU架构的程序又要如何编译了.

我们可以在$NDK_ROOT/toolchains目录下看到者几个目录:

```
arm-linux-androideabi-4.9
aarch64-linux-android-4.9
mipsel-linux-android-4.9
mips64el-linux-android-4.9
x86-4.9
x86_64-4.9
```

这就是不同CPU架构的交叉编译工具链了.还记得我们安装工具链的命令吗?

> $NDK_ROOT/build/tools/make-standalone-toolchain.sh \
	--platform=android-19 \
	--install-dir=$HOME/Android/standalone-toolchains/android-toolchain-arm \
	--toolchain=arm-linux-androideabi-4.9 \
	--stl=gnustl

toolchain参数就能指定使用哪个工具链,然后就能使用该工具链去编译该架构版本的程序了.

但是,我们看到这下面并没有armeabi-v7a的工具链,那armeabi-v7a的程序要如何编译呢?

其实armeabi-v7a的程序也是用arm-linux-androideabi-4.9去编译的,只不过在编译的时候可以带上-march=armv7-a:

> arm-linux-androideabi-gcc -march=armv7-a -shared -fPIC test.c -o test.so

## 官方文档

我这边其实只是简要的介绍了下NDK的基本用法而已,更多的用法大家可以到[官方文档](https://developer.android.com/ndk/guides/standalone_toolchain?hl=zh-cn)上查找.

# Makefile

理解完C/C++编译的原理之后,还有个十分重要的东西还要了解,这个东西就是Makefile.

我们前面的例子都是直接用gcc或着各个交叉编译的版本的gcc去编译C/C++代码的.在代码量不多的时候这么做还是可行的,但是如果软件一旦复杂一些,代码量一多,那么编译的命令就会十分的复杂,而且还需要考虑到多个模块之间的依赖关系.

Makefile就是一个帮助我们解决这些问题的工具.它的基本原理十分简单,先让我们看看它最最基本的用法:

```
目标文件 : 依赖文件
	命令1
  命令2
  命令3
  ...
```

还是举我们的例子代码,首先创建一个文件,名字叫Makefile,然后写上:

```
test.so : test.c test.h                                                          
    arm-linux-androideabi-gcc -march=armv7-a -shared -fPIC test.c -o test.so
```

然后就可以用make命令去编译了.make命令会找到当前目录下的Makefile,然后比较目标文件文件和依赖文件的修改时间,如果依赖文件的修改时间比较晚,或者干脆就还没有目标文件.就会执行命令.

如我们的例子,如果还没有test.so,或者test.c、test.h的修改时间比test.so要晚,那么就会执行arm-linux-androideabi-gcc -march=armv7-a -shared -fPIC test.c -o test.so,然后生成test.so文件.

而如果是目标文件比较新,就不会执行,它会告诉你目标文件已经是最新的了:

```
make: 'test.so' is up to date.
```

## 没有依赖的目标文件

然后可能有同学还有见过make clean,make install,make uninstall...这些命令,它们又是怎么一回事呢?

这里以make clean举例,我们在Makefile中加入目标文件clean:

```
test.so : test.c test.h
    arm-linux-androideabi-gcc -march=armv7-a -shared -fPIC test.c -o test.so

clean :
    rm test.so
```

现在除了test.so这个目标文件之后,还多了个目标文件clean,它下面的命令是tm test.so.而且特殊的是clean这个目标文件,它没有任何的依赖文件.

然后我们就能使用make clean命令了,因为clean文件不存在,所以就会执行下面的rm test.so.所以就会将test.so删除了.

刚刚我们说的时候clean存在的时候会执行命令,那如果我们自己创建了个文件名字叫做clean又会发生什么事情?

```
make: 'clean' is up to date.
```

由于没有依赖文件,所以不用比较时间,它会直接告诉你clean文件已经是最新的了,而不会执行命令.

那要如果规避这个问题呢?例如当前目录下的确需要有个clean文件,但是我又需要make clean这个功能.方法很简单,只需要加上".PHONY : clean"就可以了:

```
test.so : test.c test.h
    arm-linux-androideabi-gcc -march=armv7-a -shared -fPIC test.c -o test.so

clean :
    rm test.so

.PHONY : clean
```

## Makefile自动生成工具

Makefile,这里我也只是简单代过,其实它还有许多强大的功能,感兴趣的同学可以自行搜索.

但是如果我们的项目都是手动去写makefile的话也会十分的麻烦,那有没有办法可以根据我们的代码,自动生成makefile呢?

答案肯定是有的.比如现在安卓使用的CMake还有经典的AutoMake工具.

相信大家在用JNI的时候肯定都有配过CMakeLists.txt这个文件,CMake就是通过读取这个文件的配置去生成代码的.

而一些比较早期的库如ffmpeg,就是用automake去生成Makefile的,我之前写过四篇博客专门将如何使用AutoMake,如果感兴趣可以去看看.

- [automake学习笔记 - helloworld](http://blog.islinjw.cn/2017/03/17/automake%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-helloworld/)
- [automake学习笔记 - 模块化编译](http://blog.islinjw.cn/2017/03/21/automake%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-%E6%A8%A1%E5%9D%97%E5%8C%96%E7%BC%96%E8%AF%91/)
- [automake学习笔记 - 安装与发布](http://blog.islinjw.cn/2017/03/26/automake%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-%E5%AE%89%E8%A3%85%E4%B8%8E%E5%8F%91%E5%B8%83/)
- [automake学习笔记 - 交叉编译](http://blog.islinjw.cn/2017/04/02/automake%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91/)

自动生成工具这块内容比较多我就不详细讲了,它们其实并不是很难,大家自行找资料学习就好.
