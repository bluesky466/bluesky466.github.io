title: C++ pimpl与unique_ptr
date: 2023-05-30 21:55:27
tags:
  - 技术相关
	- C/C++
---

最近协助c++组的项目时使用unique\_ptr实现pimpl遇到了个编译问题,虽然比较初级但其原理也挺有意思的,这里记录一下。

简化后的代码如下:

```C++
// widget.h
#include <memory>

class Widget {
public:
	Widget();

private:
	class Impl;
	std::unique_ptr<Impl> pimpl_;
};


// widget.cpp
#include "widget.h"

class Widget::Impl {};

Widget::Widget() : pimpl_(std::make_unique<Impl>()) {}

// main.cpp
#include "widget.h"

int main(int argc, char *argv[ ]) {
	Widget w;
	return 0;
}
```

编译之后出现下面的error(如果换成裸指针或者shared\_ptr就不会有问题):

```
In file included from main.cpp:1:
In file included from ./widget.h:1:
/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1/memory:1424:19: error: invalid application of 'sizeof' to an incomplete type 'Widget::Impl'
    static_assert(sizeof(_Tp) > 0,
                  ^~~~~~~~~~~
/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1/memory:1689:7: note: in instantiation of member function 'std::default_delete<Widget::Impl>::operator()' requested here
      __ptr_.second()(__tmp);
      ^
/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1/memory:1643:19: note: in instantiation of member function 'std::unique_ptr<Widget::Impl>::reset' requested here
  ~unique_ptr() { reset(); }
                  ^
./widget.h:3:7: note: in instantiation of member function 'std::unique_ptr<Widget::Impl>::~unique_ptr' requested here
class Widget {
      ^
./widget.h:8:8: note: forward declaration of 'Widget::Impl'
        class Impl;
              ^
1 error generated.
```

从报错上可以看出unique\_ptr在析构的时候会调用默认的删除策略default\_delete去释放内存。而在这个default\_delete里面会用static\_assert去确保原始指针指向的类型不是一个未完成类型,避免无法正确执行其析构函数或delete操作符的调用，从而导致未定义的行为（如崩溃）:

```c++
template <class _Up>
  _LIBCPP_INLINE_VISIBILITY
  default_delete(const default_delete<_Up>&,
                 typename enable_if<is_convertible<_Up*, _Tp*>::value>::type* =
                     0) _NOEXCEPT {}

  _LIBCPP_INLINE_VISIBILITY void operator()(_Tp* __ptr) const _NOEXCEPT {
    static_assert(sizeof(_Tp) > 0,
                  "default_delete can not delete incomplete type");
    static_assert(!is_void<_Tp>::value,
                  "default_delete can not delete incomplete type");
    delete __ptr;
}
```

由于c++标准规定就算是空类的大小也大于0,所以只要\_Tp是已定义的类型,sizeof(\_Tp) > 0就必定成立,而这里的报错就是因为判断到Widget::Impl是个未完成类型。

但是我们其实已经在widget.cpp里面对其进行了定义,为什么这里还会未完整呢?

[Effective Modern C++](https://cntransgroup.github.io/EffectiveModernCppChinese/4.SmartPointers/item22.html)里面的解释是这样的:

```
在对象w被析构时（例如离开了作用域），问题出现了。在这个时候，它的析构函数被调用。我们在类的定义里使用了std::unique_ptr，所以我们没有声明一个析构函数，因为我们并没有任何代码需要写在里面。根据编译器自动生成的特殊成员函数的规则（见 Item17），编译器会自动为我们生成一个析构函数。 在这个析构函数里，编译器会插入一些代码来调用类Widget的数据成员pImpl的析构函数。 pImpl是一个std::unique_ptr<Widget::Impl>，也就是说，一个使用默认删除器的std::unique_ptr。 默认删除器是一个函数，它使用delete来销毁内置于std::unique_ptr的原始指针。然而，在使用delete之前，通常会使默认删除器使用C++11的特性static_assert来确保原始指针指向的类型不是一个未完成类型。 当编译器为Widget w的析构生成代码时，它会遇到static_assert检查并且失败，这通常是错误信息的来源。 这些错误信息只在对象w销毁的地方出现，因为类Widget的析构函数，正如其他的编译器生成的特殊成员函数一样，是暗含inline属性的。 错误信息自身往往指向对象w被创建的那行，因为这行代码明确地构造了这个对象，导致了后面潜在的析构。
```

整段读下来比较难晦涩,我也是仔细琢磨之后才理解,它有下面的几个关键点:

1. 编译器会生成inline的Widget析构函数
2. inline Widget析构函数里面会调用inline的default\_delete函数
3. default\_delete里面会有一个sizeof(\_Tp)

也就是说最终在main函数里面编译器会在w对象析构的地方inline插入sizeof(\_Tp)代码，类似这个样子:

```c++
int main(int argc, char *argv[]) {
    Widget w;

    //插入了一堆Widget析构函数产生的代码
    sizeof(Widget::Impl); // 中间有一句default_delete inline插入的sizeof
    // 插入了一堆Widget析构函数产生的代码

    return 0;
}
```

在编译C++代码时，[编译器通常会将每个源文件编译为一个目标文件（obj或者o文件）](https://blog.islinjw.cn/2018/07/29/%E5%86%99%E7%BB%99%E5%AE%89%E5%8D%93%E7%A8%8B%E5%BA%8F%E5%91%98%E7%9A%84cc-%E7%BC%96%E8%AF%91%E5%85%A5%E9%97%A8/),在编译main.cpp的时候并不会去读取widget.cpp里面的定义。所以在main里面inline插入的sizeof(Widget::Impl)找不到Widget::Impl的具体大小,于是报错。


解决的方式也很简单,就是主动声明一个析构函数,然后在Impl类的定义之后进行定义:


```C++
// widget.h
#include <memory>

class Widget {
public:
	Widget();
	~Widget();

private:
	class Impl;
	std::unique_ptr<Impl> pimpl_;
};



// widget.cpp
#include "widget.h"

class Widget::Impl {};

Widget::Widget() : pimpl_(std::make_unique<Impl>()) {}

Widget::~Widget() {}
```

这样的话在main函数编译的时候生成的代码大概是这样的:

```C++
int main(int argc, char *argv[ ]) {
    Widget w;

    w.~Widget(); // 由于Widget的析构函数不是inline的,所以它只是做函数的调用
    return 0;
}
```

然后在Widget的析构函数里面就算default\_delete代码inline插入sizeof(Widget::Impl)也没有关系,因为这个时候Widget::Impl已经定义了:

```c++
class Widget::Impl {}; // Impl在~Widget之前定义
...
Widget::~Widget() {
	// 插入了一堆Widget内存释放代码
    sizeof(Widget::Impl); // 中间有一句default_delete inline插入的sizeof,但这个时候Widget::Impl已经定义了
    // 插入了一堆Widget内存释放代码
}
```

