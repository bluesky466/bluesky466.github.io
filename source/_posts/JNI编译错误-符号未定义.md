title: JNI编译错误-符号未定义
date: 2020-09-27 23:25:59
tags:
    - 技术相关
    - Android
    - C/C++
---
这篇笔记记录了一次编译问题的排查过程，还简单介绍了一些C/C++编译的知识，希望对jni编译错误的排查能有点帮助。

没有接触过C/C++的安卓程序员可能在遇到so库出现编译问题的时候会有点束手无措，如果这个库是公司内部开发的还能丢给负责的同事分析，如果是第三方的开源库可能就需要我们自己去分析了。

# Alexa rapidjson符号未定义

我司基于亚马逊开源的Alexa应用分为两个工程，一个是[avs-device-sdk](https://github.com/alexa/avs-device-sdk)用于打包出相关的动态链接库(当然我们对这个开源项目做了一些定制化修改)。另外一个是安卓应用工程，包含java层的ui展现、jni接口和部分c++的alexa初始化逻辑。

在更新Alexa sdk版本打包apk的过程中出现了这样一个编译错误:

```c++
error: undefined reference to 'alexaClientSDK::avsCommon::utils::json::jsonUtils::findNode(rapidjson::GenericValue<rapidjson::UTF8<char>, rapidjson::MemoryPoolAllocator<rapidjson::CrtAllocator> > const&, std::__ndk1::basic_string<char, std::__ndk1::char_traits<char>, std::__ndk1::allocator<char> > const&, rapidjson::GenericMemberIterator<true, rapidjson::UTF8<char>, rapidjson::MemoryPoolAllocator<rapidjson::CrtAllocator> >*)'
```

这个错误的意思是找不到findNode这个方法的定义。

# c/c++编译基础

在这里要大概的介绍下c/c++的编译流程，C/C++的编译可以分为下面几个步骤:

{% img /JNI编译错误-符号未定义/1.png %}

代码通过前面的预处理、编译、汇编之后就生成了包含机器指令的.o文件，一个.c或者.cpp文件就会生成一个.o文件。每个.o文件都只有自己那部分的代码，需要将他们合并到一起才能组成一个可执行程序，这个合并的过程叫做链接。

{% img /JNI编译错误-符号未定义/2.png %}

而且.o文件之间是有依赖关系的，a.o可能调用到b.o的代码，如果调用的代码的实现找不到了，就会出现上面所说的undefined reference错误。

实际上链接过程中除了.o文件之外还有有动态链接库、静态链接库参与进来。链接库是一些可复用的代码，

动态链接库、静态链接库windows上对应的是.lib、.dll linux上对应的是.a、.so它们的区别在于静态链接库在链接的时候会和.o文件一起打包到可执行文件中，而动态链接库不会被打包进可执行文件，而是在运行过程中被加载。

.o文件依赖的代码除了在其他.o文件中，还有可能在静态链接库或者动态链接库中。(关于c/c++的编译我之前有写一篇博客[写给安卓程序员的C/C++编译入门](https://blog.islinjw.cn/2018/07/29/写给安卓程序员的cc-编译入门/),感兴趣的同学可以去看看)

# nm命令使用

出现了上面的错误，我的第一反应是so库没有链接进来。该函数的定义在libAVSCommon.so，查看CMakeLists.txt之后发现该库是有正常链接的:

```cmake
target_link_libraries( AlexaJni AVSCommon)
```

而且如果将这行代码注释掉，不去链接它，会出现更多其他的符号未定义，所以这个库肯定是正常链接的。

这种情况下可能考虑是不是出现了宏没有打开的情况，例如下面这样的代码:

```c++
#ifdef EN_XXX
// code
#endif
```

如果这个EN_XXX的宏没有定义，中间的代码就会在预编译的时候被清除。

这样的情况我们有两种方法去判断，第一种是看代码看看能不能找到这样的宏，并且看看它是否真的没有打开。另一种方法是通过nm命令列举so文件中的符号:

> nm libAVSCommon.so

我们在里面是可以找到findNode符号的:

{% img /JNI编译错误-符号未定义/3.png %}

```shell
0013f768 T _ZN14alexaClientSDK9avsCommon5utils4json9jsonUtils8findNodeERKN9rapidjson12GenericValueINS4_4UTF8IcEENS4_12CrtAllocatorEEERKNSt6__ndk112basic_stringIcNSC_11char_traitsIcEENSC_9allocatorIcEEEEPNS4_21GenericMemberIteratorILb1ES7_S8_EE
```

我们首先可以看到的是方法的名字好像和之前不一样了，感觉多了很多内容。

## 编译后C++函数名会被修改

其实C/C++在编译之后函数的名字就会被修改，改成编译器内部的名字，每个编译器都有一套自己内部的名字，这里就看看g++编译器的实现:

```c++
namespace mytest {
  void foo(int a) {
  }
  void foo(int a, int b) {
  }
}
```

namespace 关键字顾名思义是定义一个命名空间。编程的时候一个很令人头大的问题是起名字，很多的库可能都会有个叫做Utils的类，也有可能有些比较常见的函数如PrintLog，当导入多个不同的第三方库的时候很容易出现命名冲突的问题，namespace的作用就在于将这些同名的类或者函数区分开来，一般的第三方库都会以库的名字起一个命名空间，这样就能尽可能的减少命名冲突。

c++编译器实现命名空间的方式也很简单，就在将命名空间的名字拼到函数名前面，可以看到编译之后foo的前面多了mytest这个命名空间:

```shell
0000000100000f60 T __ZN6mytest3fooEi
0000000100000f70 T __ZN6mytest3fooEii
```

然后我们可以在上面的例子看到有两个名字一样的函数，但是他们的参数数量不一样，像这种函数名字一样但是参数数量或者类型不一样的情况叫做函数重载，其实C语言不支持函数重载，但是C++支持的。C++支持重载的原理就是在函数的后面拼接参数类型:

例如void foo(int a)有一个int类型的参数，于是就在后面加了一个i:

```shell
__ZN6mytest3fooEi
```

而void foo(int a, int b)又两个int类型的参数，于是就在后面加了两个i:

```shell
__ZN6mytest3fooEii
```

同理的如果有泛型参数也是会拼接在函数名字后面。

像我们在使用JNI的时候经常能看到这样的代码

```
#ifdef __cplusplus
extern "C" {
#endif

...

#ifdef __cplusplus
}
#endif

```

就是为了在c++编译器声明C代码防止函数名被修改，因为在静态注册的情况下，java层的natvie方法名和c的函数名是有对应关系的，一旦函数名被修改了就对应不上了。

## nm符号类型

我们可以看到上面nm命令的打印，在函数名签名都会有一个 T，这个T指的是该符号的类型。符号类型有很多种，它和C/C++的内存模型有很大关系，作为安卓程序员我们只需要大概了解就够了，对于每一个符号来说，其类型如果是小写的，则表明该符号是local的；大写则表明该符号是global(external)的。下面这张表我是在网上摘抄的别人的[博客](https://www.cnblogs.com/LiuYanYGZ/p/5536607.html):

| **符号 类型** | **说明**                                                     |
| ------------- | ------------------------------------------------------------ |
| A             | 该符号的值是绝对的，在以后的链接过程中，不允许进行改变。这样的符号值，常常出现在中断向量表中，例如用符号来表示各个中断向量函数在中断向量表中的位置。 |
| B             | 该符号的值出现在非初始化数据段(bss)中。例如，在一个文件中定义全局static int test。则该符号test的类型为b，位于bss section中。其值表示该符号在bss段中的偏移。一般而言，bss段分配于RAM中 |
| C             | 该符号为common。common symbol是未初始话数据段。该符号没有包含于一个普通section中。只有在链接过程中才进行分配。符号的值表示该符号需要的字节数。例如在一个c文件中，定义int test，并且该符号在别的地方会被引用，则该符号类型即为C。否则其类型为B。 |
| D             | 该符号位于初始话数据段中。一般来说，分配到data section中。例如定义全局int baud_table[5] = {9600, 19200, 38400, 57600, 115200}，则会分配于初始化数据段中。 |
| G             | 该符号也位于初始化数据段中。主要用于small object提高访问small data object的一种方式。 |
| I             | 该符号是对另一个符号的间接引用。                             |
| N             | 该符号是一个debugging符号。                                  |
| R             | 该符号位于只读数据区。例如定义全局const int test[] = {123, 123};则test就是一个只读数据区的符号。注意在cygwin下如果使用gcc直接编译成MZ格式时，源文件中的test对应_test，并且其符号类型为D，即初始化数据段中。但是如果使用m6812-elf-gcc这样的交叉编译工具，源文件中的test对应目标文件的test,即没有添加下划线，并且其符号类型为R。一般而言，位于rodata section。值得注意的是，如果在一个函数中定义const char *test = “abc”, const char test_int = 3。使用nm都不会得到符号信息，但是字符串“abc”分配于只读存储器中，test在rodata section中，大小为4。 |
| S             | 符号位于非初始化数据区，用于small object。                   |
| T             | 该符号位于代码区text section。                               |
| U             | 该符号在当前文件中是未定义的，即该符号的定义在别的文件中。例如，当前文件调用另一个文件中定义的函数，在这个被调用的函数在当前就是未定义的；但是在定义它的文件中类型是T。但是对于全局变量来说，在定义它的文件中，其符号类型为C，在使用它的文件中，其类型为U。 |
| V             | 该符号是一个weak object。                                    |
| W             | The symbol is a weak symbol that has not been specifically tagged as a weak object symbol. |
| -             | 该符号是a.out格式文件中的stabs symbol。                      |
| ?             | 该符号类型没有定义                                           |

函数的符号类型是T或者小写的t，当然如果这个函数是在其他动态链接库里面定义的，它的类型就是U。

那local和global有什么区别吗？什么样的函数是local的，什么样的函数是global的？

一般情况下函数默认都是global的，这意味着so库里面的函数可以被外部使用，但是如果有些函数只想so库内部使用不暴露给外部，就可以添加static关键字:

```c++
static void foo() {
}
```

它的符号类型就是t，在so外部是看不到这个函数不能使用的:

```
0000000100000fb0 t __ZL3foov
```

## rapidjson编译配置

我们使用nm命令打印libAVSCommon.so的符号表可以看到，alexaClientSDK::avsCommon::utils::json::jsonUtils::findNode这个函数在so里面是存在的，而且是有定义的(符号类型是T不是U):

```shell
0013f768 T _ZN14alexaClientSDK9avsCommon5utils4json9jsonUtils8findNodeERKN9rapidjson12GenericValueINS4_4UTF8IcEENS4_12CrtAllocatorEEERKNSt6__ndk112basic_stringIcNSC_11char_traitsIcEENSC_9allocatorIcEEEEPNS4_21GenericMemberIteratorILb1ES7_S8_EE
```

一开始看到这里我是懵逼的，很不科学。于是我用了一个方法，修改了findNode函数的名字，重新编译libAVSCommon.so，这样的话在其他so库使用它的地方就出现了找不到符号的错误:

```c++
error: undefined reference to 'alexaClientSDK::avsCommon::utils::json::jsonUtils::findNode(rapidjson::GenericValue<rapidjson::UTF8<char>, rapidjson::CrtAllocator> const&, std::__ndk1::basic_string<char, std::__ndk1::char_traits<char>, std::__ndk1::allocator<char> > const&, rapidjson::GenericMemberIterator<true, rapidjson::UTF8<char>, rapidjson::CrtAllocator>*)'
```

而这里的打印和我们在编译apk的时候的打印好像不太一样，编译apk的时候的打印如下:

```c++
error: undefined reference to 'alexaClientSDK::avsCommon::utils::json::jsonUtils::findNode(rapidjson::GenericValue<rapidjson::UTF8<char>, rapidjson::MemoryPoolAllocator<rapidjson::CrtAllocator> > const&, std::__ndk1::basic_string<char, std::__ndk1::char_traits<char>, std::__ndk1::allocator<char> > const&, rapidjson::GenericMemberIterator<true, rapidjson::UTF8<char>, rapidjson::MemoryPoolAllocator<rapidjson::CrtAllocator> >*)'
```

它们的参数类型不一样,这个时候我们再去看看它的函数定义:

```c++
bool findNode(
    const rapidjson::Value& jsonNode,
    const std::string& key,
    rapidjson::Value::ConstMemberIterator* iteratorPtr);
```

rapidjson::Value的定义如下:

```c++

template <typename Encoding, typename Allocator = RAPIDJSON_DEFAULT_ALLOCATOR >
class GenericValue {
    ...
  typedef typename GenericMemberIterator<true,Encoding,Allocator>::Iterator ConstMemberIterator;  //!< Constant member iterator for iterating in object.
  ...
};

//! GenericValue with UTF8 encoding
typedef GenericValue<UTF8<> > Value;
```

这里我们可以看到，泛型参数Allocator由RAPIDJSON\_DEFAULT\_ALLOCATOR这个宏决定，默认情况下是:

```c++
#ifndef RAPIDJSON_DEFAULT_ALLOCATOR
#define RAPIDJSON_DEFAULT_ALLOCATOR MemoryPoolAllocator<CrtAllocator>
#endif
```

也就是我们在编译apk的时候出现的类型。问题到这里实际就已经找到原因了，编译so库的时候和编译apk的时候RAPIDJSON\_DEFAULT\_ALLOCATOR这个宏的定义不一致，导致findNode参数类型不一致。那肯定是编译配置导致的,于是在工程下grep搜索一下RAPIDJSON\_DEFAULT\_ALLOCATOR可以发现编译so的cmake配置里有这几条:

```cmake
if(RAPIDJSON_MEM_OPTIMIZATION STREQUAL "OFF")
    ...
elseif(RAPIDJSON_MEM_OPTIMIZATION STREQUAL "CUSTOM")
    ...
else()
    # Use Memory Optimization
    message(STATUS "rapidjson memory optimization used")
    add_definitions(-DRAPIDJSON_DEFAULT_ALLOCATOR=CrtAllocator)
    add_definitions(-DRAPIDJSON_VALUE_DEFAULT_OBJECT_CAPACITY=1)
    add_definitions(-DRAPIDJSON_VALUE_DEFAULT_ARRAY_CAPACITY=1)
    add_definitions(-DRAPIDJSON_DEFAULT_STACK_ALLOCATOR=CrtAllocator)
endif()
~
```

而RAPIDJSON\_MEM\_OPTIMIZATION这个配置再grep搜索下发现没有地方配置，所以默认使用了rapidjson内存优化选项。

所以只需要把这几行配置拷贝到apk工程的CMakeList.txt就好:

```cmake
add_definitions(-DRAPIDJSON_DEFAULT_ALLOCATOR=CrtAllocator)
add_definitions(-DRAPIDJSON_VALUE_DEFAULT_OBJECT_CAPACITY=1)
add_definitions(-DRAPIDJSON_VALUE_DEFAULT_ARRAY_CAPACITY=1)
add_definitions(-DRAPIDJSON_DEFAULT_STACK_ALLOCATOR=CrtAllocator)
```

