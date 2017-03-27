title: automake学习笔记 - 安装与发布
date: 2017-03-26 15:06:56
tags:
	- 技术相关
	- 编译相关
---

辛辛苦苦写出来的代码当然是需要发布出来给自己或者别人去使用的。这篇文章就谈一谈发布相关的东西吧。

## 安装

软件在编译完后就需要进行安装。configure生成的Makefile支持install。使用make install 命令就可以将编译出来的软件安装到系统中。

如果没有做配置，默认会安装到/usr/local中,当然如果需要的话也可以使用configure的--prefix参数指定安装的路径,如在build中执行下面的命令就可以将build目录指定为安装目录:

> ../configure --prefix=\`pwd\`

之后再执行下面的安装命令,工程在编译完后就会安装到build目录下。安装完毕之后可以看到build里面多了bin和lib两个目录

> make install

bin目录下是编译出来的可执行文件example,而lib目录下就是编译出来的依赖库:

```
-rw-r--r-- 1 linjw linjw 263K 3月  22 13:44 libeasylog.a
-rwxr-xr-x 1 linjw linjw  985 3月  22 13:44 libeasylog.la
lrwxrwxrwx 1 linjw linjw   19 3月  22 13:44 libeasylog.so -> libeasylog.so.0.0.0
lrwxrwxrwx 1 linjw linjw   19 3月  22 13:44 libeasylog.so.0 -> libeasylog.so.0.0.0
-rwxr-xr-x 1 linjw linjw 143K 3月  22 13:44 libeasylog.so.0.0.0
```


编译出来的libeasylog.so.0.0.0就可以直接拿出去给其他人使用了

## 卸载

卸载的话很简单，只需要执行下面的命令就行了

> make uninstall

当然你也可以选择手动去将安装的文件一个个删除,但是这样既麻烦又容易漏删或者错删

## libtool 库版本号系统

我们可以看到编译出来的so库是带版本号的，默认0.0.0,当然我们也能直接忽略版本号(某些可动态加载的的插件模块可能不需要版本号):

```
lib_LTLIBRARIES = libeasylog.la

libeasylog_la_SOURCES = cout_log_interface.cpp \
					 easy_log.cpp 

libeasylog_la_LDFLAGS = -avoid-version
```

这样生成安装的so库就不会带版本号了:

```
-rw-r--r-- 1 linjw linjw 263K 3月  26 11:27 libeasylog.a
-rwxr-xr-x 1 linjw linjw  975 3月  26 11:27 libeasylog.la
-rwxr-xr-x 1 linjw linjw 143K 3月  26 11:27 libeasylog.so
```

当然，绝大部分的库都是需要带上版本号的。每个系统的库版本机制都不一样,libtool通过一种抽象的版本机制最终在创建库的时候才映射到具体的系统版本机制上。这是为了方便在交叉编译的时候可以用一种机制去管理不同平台上的各种版本机制。

libtool 库版本号系统有下面三个部分:

- current

接口的修改次数

- revision

上次修改后源码的修改次数(注意这里指的是只改动了实现,没有修改接口,如果改了接口的话应该要改current号，并且把revision置零)

- age

当前版本可以向前兼容的版本数

[官方文档](http://www.gnu.org/software/libtool/manual/libtool.html#Updating-version-info)是这么描述这三个部分的更新规则的:

> Here are a set of rules to help you update your library version information:
1.    Start with version information of ‘0:0:0’ for each libtool library.
2.    Update the version information only immediately before a public release of your software. More frequent updates are unnecessary, and only guarantee that the current interface number gets larger faster.
3.    If the library source code has changed at all since the last update, then increment revision (‘c:r:a’ becomes ‘c:r+1:a’).
4.    If any interfaces have been added, removed, or changed since the last update, increment current, and set revision to 0.
5.    If any interfaces have been added since the last public release, then increment age.
6.    If any interfaces have been removed or changed since the last public release, then set age to 0. 

翻译过来就是

1. 库版本号应该开始于0.0.0
2. 只有在正式发布库的时候才更新版本号以避免版本号增长过快
3. 当实现代码改变的时候revision加1
4. 当接口改变(无论是添加，删除还是修改接口声明)的时候current加1,同时revision重置为0
5. 如果库只是增加了接口,则age加1
6. 如果库删除或者修改了接口声明,则age重置为0

这三个值可以用-version-info指定

> -version-info current[:revision[:age]] 

revision 和 age都可以省略,例如你这样设置:

```
lib_LTLIBRARIES = libeasylog.la

libeasylog_la_SOURCES = cout_log_interface.cpp \
					 easy_log.cpp 

libeasylog_la_LDFLAGS = -version-info 3:12:1
```

表明接口被修改了三次,第三次修改接口之后又修改了12次源码,接口可以向前兼容1个版本

make install 后可以看到lib目录下生成的库长这样:

```
-rw-r--r-- 1 linjw linjw 263K 3月  26 12:11 libeasylog.a
-rwxr-xr-x 1 linjw linjw  987 3月  26 12:11 libeasylog.la
lrwxrwxrwx 1 linjw linjw   20 3月  26 12:11 libeasylog.so -> libeasylog.so.2.1.12
lrwxrwxrwx 1 linjw linjw   20 3月  26 12:11 libeasylog.so.2 -> libeasylog.so.2.1.12
-rwxr-xr-x 1 linjw linjw 143K 3月  26 12:11 libeasylog.so.2.1.12
```

为啥是libeasylog.so.2.1.12而不是libeasylog.so.3.12.1呢？

原来这几个数字是这样计算的:

库名.so.current-age.age.revision

这样会引发一个问题:

> 假设你的库有两个【3：0：1】【4：0：2】。 再假设在你编译程序的机器上安装了最新的【4：0：2】， 且你在程序中使用了该版本中新加的接口。当你程序编译好后， 你ldd发现你的程序依赖libraryname.so.2， 同时你将程序安装在了只安装了【3：0：1】的机器上， 你会发现你的程序能搜索到动态库， 却在运行的时候发现未定义的符号， 因为【3：0：1】中没有新添加的接口。 故你需要在运行机器上保证安装了同一主版本号最新的library， 以保证你的程序能正确运行。

从 [libtool动态库版本系统之个人理解 ](http://blog.csdn.net/zlyong0018/article/details/16846325) 这篇博客引用

## 手动指定版本号

我之前了解到的so的命名规范其实和libtool的版本号系统的so库命名规范不一样:

> 库名.so.主版本号.次版本号.发布版本号

[官方文档](http://www.gnu.org/software/libtool/manual/libtool.html#Release-numbers)也有提到这一点:


> Often, people want to encode the name of the package release into the shared library so that it is obvious to the user what package their programs are linked against. This convention is used especially on GNU/Linux:
> 
> trick$ ls /usr/lib/libbfd*
> /usr/lib/libbfd.a           /usr/lib/libbfd.so.2.7.0.2
> /usr/lib/libbfd.so
> trick$
> 
> On ‘trick’, /usr/lib/libbfd.so is a symbolic link to libbfd.so.2.7.0.2, which was distributed as a part of ‘binutils-2.7.0.2’.
> 
> Unfortunately, this convention conflicts directly with libtool’s idea of library interface versions, because the library interface rarely changes at the same time that the release number does, and the library suffix is never the same across all platforms.
> 
> So, to accommodate both views, you can use the -release flag to set release information for libraries for which you do not want to use -version-info. For the libbfd example, the next release that uses libtool should be built with ‘-release 2.9.0’, which will produce the following files on GNU/Linux:
> 
> trick$ ls /usr/lib/libbfd*
> /usr/lib/libbfd-2.9.0.so     /usr/lib/libbfd.a
> /usr/lib/libbfd.so
> trick$
> 
> In this case, /usr/lib/libbfd.so is a symbolic link to libbfd-2.9.0.so. This makes it obvious that the user is dealing with ‘binutils-2.9.0’, without compromising libtool’s idea of interface versions.
> 
> Note that this option causes a modification of the library name, so do not use it unless you want to break binary compatibility with any past library releases. In general, you should only use -release for package-internal libraries or for ones whose interfaces change very frequently. 

可以使用-release去手动指定版本号,虽然官方不推荐用这种方式:

```
lib_LTLIBRARIES = libeasylog.la

libeasylog_la_SOURCES = cout_log_interface.cpp \
					 easy_log.cpp 

libeasylog_la_LDFLAGS = -release 0.0.3
```

安装之后lib目录如下:

```
-rwxr-xr-x 1 linjw linjw 143K 3月  26 14:48 libeasylog-0.0.3.so
-rw-r--r-- 1 linjw linjw 263K 3月  26 14:48 libeasylog.a
-rwxr-xr-x 1 linjw linjw  993 3月  26 14:48 libeasylog.la
lrwxrwxrwx 1 linjw linjw   19 3月  26 14:48 libeasylog.so -> libeasylog-0.0.3.so
```

可以在[这里](https://github.com/bluesky466/automake-demo/tree/v0.0.3)查看完整的项目代码
