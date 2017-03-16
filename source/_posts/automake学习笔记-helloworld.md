title: automake学习笔记 - helloworld
date: 2017-03-17 01:11:40
tags:
	- automake
	- 编译相关
	- 技术相关
---

虽然之前已经用过一段时间的automake,但是总觉得对它的理解不过充分,只是知其然而不知其所以然。于是下定决心好好啃[文档](http://www.gnu.org/software/automake/manual/automake.html),并将学的的东西记录下来。

这篇文章用一个简单的log工具的编译先对automake做一个hello world级别的介绍。

## 代码
我们的demo有6个文件cout\_log\_interface.h, cout\_log\_interface.cpp, log\_interface.h, easy\_log.h, easy\_log.cpp, main.cpp

简单介绍下代码吧,首先有个简单的Log类:

```
class EasyLog {
public:
    EasyLog(std::shared_ptr<LogInterface> interface);

    void Info(const std::string& tag, const std::string& log);

    void Debug(const std::string& tag, const std::string& log);

    void Warn(const std::string& tag, const std::string& log);

    void Error(const std::string& tag, const std::string& log);

private:
    std::string GetLog(const std::string& tag, const std::string& log) const;

    std::shared_ptr<LogInterface> interface_;
};
```

它的实现十分简单，就是将所有的操作代理给LogInterface:

```
EasyLog::EasyLog(shared_ptr<LogInterface> interface)
    : interface_(interface)
{
}

void EasyLog::Info(const string& tag, const string& log)
{
    interface_->DoLog(kInfo, tag, GetLog(tag, log));
}

void EasyLog::Debug(const string& tag, const string& log)
{
    interface_->DoLog(kDebug, tag, GetLog(tag, log));
}

void EasyLog::Warn(const string& tag, const string& log)
{
    interface_->DoLog(kWarn, tag, GetLog(tag, log));
}

void EasyLog::Error(const string& tag, const string& log)
{
    interface_->DoLog(kError, tag, GetLog(tag, log));
}

std::string EasyLog::GetLog(const string& tag, const string& log) const
{
    return "[" + tag + "]" + " " + log;
}
```

LogInterface是一个纯虚类，然后LogLevel是一个枚举体:

```
enum LogLevel {
    kInfo,
    kDebug,
    kWarn,
    kError
};

class LogInterface {
public:
    virtual void DoLog(LogLevel level, const std::string& tag, const std::string& log) = 0;
};
```

我们再写一个使用标准输出打印log的LogInterface:

```
class COutLogInterface : public LogInterface {
public:
    virtual void DoLog(LogLevel level, const std::string& tag, const std::string& log);
};
```

它的实现就是使用cout打印log:

```
void COutLogInterface::DoLog(LogLevel level, const string& tag, const string& log) {
    cout<<log<<endl;
}
```

当然需要有个main函数:

```
int main()
{
    EasyLog log(std::make_shared<COutLogInterface>());
    log.Debug("test", "testlog");
    return 0;
}
```

## Makefil.am

automake使用Makefil.am配置工程的源码,它的内容如下:

```
bin_PROGRAMS = easylog
easylog_SOURCES = cout_log_interface.cpp \
                  easy_log.cpp \
                  main.cpp
```

bin\_PROGRAMS 指定了要编译生成的目标程序的名字,在这里我们最终编译出来的目标程序的文件名是easylog

之后的 easylog\_SOURCES 指定了需要参与编译的源代码。

如果需要同时编译多个目标程序的话可以用下面的方式分别指定各个目标程序的源代码

```
bin_PROGRAMS = program_a program_b
program_a_SOURCES = code_a.cpp
program_b_SOURCES = code_b.cpp 
```

## configure.ac

指定了源代码还不够，因为automake不仅仅可以用来生成编译c/c++的makefile，还可以用来编译生成其他许许多多语言的makefile，所以还需要指定编译器和依赖文件等。automake使用configure.ac配置这些东西,本例子的configure.ac是这么写的:

```
AC_INIT([easylog], [0.0.1], [466474482@qq.com])
AM_INIT_AUTOMAKE([-Wall -Werror foreign])
AC_CONFIG_HEADERS([config.h])

AC_PROG_CXX
AX_CXX_COMPILE_STDCXX_11

AC_CONFIG_FILES([Makefile])

AC_OUTPUT
```

简单分析一下configure.ac的内容:

- AC\_INIT 

指定了工程的名字、版本号、和bug的报告邮箱

- AM\_INIT\_AUTOMAKE

指定了一些选项,-Wall和-Werror指定编译的时候将所有的warning当做error来报错, foreign告诉automake这里不用遵循GNU标准。GNU软件包应该包括一些附加文件去描述如修改项，作者等信息。在这里我们不需要automake去检查这些附加文件的存在。

- AC\_CONFIG\_HEADERS 

我们在AC\_INIT中配置了版本号等信息,c/c++中一般需要用宏来定义它们,这里就指定了生成的配置宏的头文件名。配置了这里,automake就会自动帮我们生成config.h头文件,里面定义了一些VERSION之类的宏

- AC\_PROG\_CXX

该宏用于检查系统中是否有g++编译器

- AX\_CXX\_COMPILE\_STDCXX\_11 

检查系统的c++11编译支持

- AC\_CONFIG\_FILES 

指定了Makefile.am的路径,这里把后缀省略了,同时因为Makefile.am和configure.ac在同级目录,所以直接写Makefile就好了。在文字的后面我会介绍当Makefile.am和configure.ac不在同级目录的时候需要怎么配置

- AC\_OUTPUT

这是一个结束标志,实际上它是一个脚本命令用来创建AC\_CONFIG\_HEADERS和AC\_CONFIG\_FILES所配置的文件

## 生成Makefile

首先要安装autoconf

> sudo apt-get install autoconf

然后使用下面的命令生成configure

> autoreconf --install

除了configure之外，它还会生成一些其他的文件，当然现在我们不需要去管这些文件

之后就能使用configure脚本去生成Makefile和config.h等

> ./configure


## 编译工程

Makefile都已经生成了，现在就可以使用make命令编译工程啦  

编译成功之后就能在当前目录看到easylog程序。我们可以运行它:

> ./easylog

得到下面输出:

> [test] testlog

## 在build目录中编译项目

现在我们编译生成的.o文件和目标程序都混在源代码中间,看起来很不舒服。我们可以创建一个build目录。然后进入build目录执行下面命令

> ../configure

这样就在build目录下生成Makefile了,于是现在我们在build中使用make命令编辑工程就会发现编译产生的.o文件和目标文件都在build中而不会污染源代码了。
