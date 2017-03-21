title: automake学习笔记 - 模块化编译
date: 2017-03-21 13:36:36
tags:
	- 技术相关
	- 编译相关
---

一般来说一个工程会由许多不同的模块组成。源码放在一个地方,示例代码放到另一个地方，第三方库又放到其他地方。这种时候又应该怎么去使用automake呢？

这篇文章就讨论了一下如何使用automake去进行模块化编译

我们还是用easylog来做例子，下面是我们修改后的easylog工程的根目录下的文件:

> configure.ac  examples  Makefile.am  src

## src目录

src目录放的就是库的源代码,我们使用src中的源代码编译出一个库来给其他的程序使用easylog的功能  

src目录中有下面几个文件log\_interface.h，easy\_log.h，easy\_log.cpp，cout\_log\_interface.h，cout\_log\_interface.cpp，Makefile.am。实际上就是除了main.cpp，其他文件都放到了这里来。因为我们提供给别的是一个库而不是一个可执行程序，所以main.cpp可以不需要编译到目标文件中

.h和.cpp的内容和上一篇[文章](http://blog.islinjw.cn/2017/03/17/automake%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-helloworld/)的内容是一样的。这里的重点其实是Makefile.am。让我们先来看看它的内容:

```
lib_LTLIBRARIES = libeasylog.la                                                             
libeasylog_la_SOURCES = cout_log_interface.cpp \                                            
                     easy_log.cpp
```

其实它的内容很简单，就是指定了要编译的库的名字和库的源码。但是有一个问题，我们这里需要编译的是但为什么这里的目标文件却是libeasylog.la呢?

Libtool是一种属于GNU构建系统的GNU程序设计工具,它将静态库和动态库抽象成了一种统一的叫做libtool库的概念。libtool库使用la作为后缀。它可以用来构建静态库也能用来构建动态库，而最终编译出来的到底是哪一种，在最后执行configure命令的时候才能确定。同时它编译的时候产生的文件就不再是.o文件而是.lo文件。  

这里lib\_LTLIBRARIES的lib前缀表示的就是目标文件是一个动态库而不是可执行文件(bin前缀表示目标文件是可执行文件,noinst\_LTLIBRARIES表示目标文件是静态库)。而LTLIBRARIES的LT指的就是Libtool。还有一点是一般编译库文件的话我们会在文件名钱加上lib前缀，所以我们的目标文件是libeasylog.la。

而下面的libeasylog\_la\_SOURCES就是指定编译libeasylog.la使用的源代码



## examples目录

examples目录里面放了这个库的example代码。因为我们的库是要提供给其他人使用的，所以一般除了文档之外，还会有一些例子去帮助使用者了解应该如何去使用我们的库。这个目录中的example.cpp其实就是上一篇[文章](http://blog.islinjw.cn/2017/03/17/automake%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-helloworld/)中的main.cpp：

```
#include "easy_log.h"
#include "cout_log_interface.h"


int main()
{
    EasyLog log(std::make_shared<COutLogInterface>());
    log.Debug("test", "testlog");

    return 0;
}
```

这个目录下也有一个Makefile.am，它是用来配置example程序的编译选项的:

```
AM_CPPFLAGS = -I$(top_srcdir)/src

bin_PROGRAMS = example
example_SOURCES = example.cpp

example_LDADD = -L$(top_builddir)/src \
				-leasylog
```

AM\_CPPFLAGS的值在c/c++预处理的时候会当做参数传给预处理器例如我们将源码目录传给预处理器，这样预处理器才能找到easy\_log.h和cout\_log\_interface.h

这里的top\_srcdir变量会在configure是被定义，它的值是工程目录的位置(也就是configure所在目录的位置)，后面的top\_builddir也是类似的，不过它的值是编译目录的位置(也就是执行make命令是所在的目录)

这里编译出来的example就是我们的demo程序


## 根目录

根目录下也有个Makefile.am,这个文件的内容很简单:

```
SUBDIRS = src examples
```

就是将src和examples指定为子目录于是在make编译的时候,编译器就会进入到这两个目录中继续编译。它们在这里的先后顺序决定了编译的先后顺序。因为examples中的example程序是依赖于easylog库的,所以要然src先编译

如果不在这里指定子目录的话,在编译目录执行make命令就不会自动编译子目录中为源码,需要自己进到子目录中手动执行make命令。如果工程中的某些部分是可选编译的时候可以这么做。

最后就是configure.ac文件了:

```
#                                               -*- Autoconf -*-
# Process this file with autoconf to produce a configure script.

AC_PREREQ([2.69])
AC_INIT([easylog], [0.0.2], [466474482@qq.com])
AM_INIT_AUTOMAKE([-Wall -Werror foreign])
AC_CONFIG_SRCDIR([src/log_interface.h])
AC_CONFIG_HEADERS([config.h])

AM_PROG_AR
LT_INIT

# Checks for programs.
AC_PROG_CXX
AC_PROG_CC
AX_CXX_COMPILE_STDCXX_11

# Checks for libraries.

# Checks for header files.

# Checks for typedefs, structures, and compiler characteristics.

# Checks for library functions.

AC_CONFIG_FILES([Makefile
                 examples/Makefile
                 src/Makefile])

AC_OUTPUT
```

它和上一篇[文章](http://blog.islinjw.cn/2017/03/17/automake%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-helloworld/)只有一点点小的不同:

一是由于将log\_interface.h放到src中了，所以AC\_CONFIG\_SRCDIR需要改一下

```
AC_CONFIG_SRCDIR([src/log_interface.h])
```

二是examples和src中的Makefile.am也需要在configure.ac中指定:
```
AC_CONFIG_FILES([Makefile
                 examples/Makefile
                 src/Makefile])
```

AC\_CONFIG\_FILES指定了一些需要从Makefile.in中生成的Makefile。这里如果不指定的话configure就不会为其生成Makefile


三是多了AM\_PROG\_AR和LT\_INIT。如果不定义这两个宏的话,执行autoreconf --install命令得到了下面的错误日志:

```
src/Makefile.am:1: error: Libtool library used but 'LIBTOOL' is undefined
src/Makefile.am:1:   The usual way to define 'LIBTOOL' is to add 'LT_INIT'
src/Makefile.am:1:   to 'configure.ac' and run 'aclocal' and 'autoconf' again.
src/Makefile.am:1:   If 'LT_INIT' is in 'configure.ac', make sure
src/Makefile.am:1:   its definition is in aclocal's search path.
automake: warnings are treated as errors
/usr/share/automake-1.15/am/ltlibrary.am: warning: 'libeasylog.la': linking libtool libraries using a non-POSIX
/usr/share/automake-1.15/am/ltlibrary.am: archiver requires 'AM_PROG_AR' in 'configure.ac'
src/Makefile.am:1:   while processing Libtool library 'libeasylog.la'
autoreconf: automake failed with exit status: 1
```

注意AM\_PROG\_AR 要放在 LT\_INIT 之前,要不然 autoreconf --install 的时候会报warn


## 编译工程

1. 在工程根目录创建子build目录用于编译

2. 进入build目录

3. 执行../configure

4. 执行make

之后进入build/examples运行example就可以看到下面输出:

> [test] testlog

## 静态库和动态库的区别

我们之前在src/Makefile.am中生成的是动态库lib\_LTLIBRARIES,所以如果将build/src目录删掉,build/examples/example就会因为找不到库而报错:

> /home/linjw/workspace/automake-demo/build/examples/.libs/lt-example: error while loading shared libraries: libeasylog.so.0: cannot open shared object file: No such file or directory

但如果将Makefile.am改成生成静态库则不会报错，因为静态库将库的代码也编译到可执行程序之中了。

```
noinst_LTLIBRARIES = libeasylog.la                                               
libeasylog_la_SOURCES = cout_log_interface.cpp \                                    
                     easy_log.cpp
```

这里的noinst代表的其实是no install的意思

## Demo项目

可以在[这里](https://github.com/bluesky466/automake-demo/tree/v0.0.2)查看完整的项目代码
