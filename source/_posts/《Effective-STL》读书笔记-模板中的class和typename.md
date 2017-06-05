title: 《Effective STL》读书笔记 - 模板中的class和typename
date: 2017-06-06 06:14:52
tags:
	- 读书笔记
	- c++
---

借来的《Effective STL》已经放在书架上很久了,想想这段时间不是在写lua做业务,就是在学安卓准备做业务,已经很久没有看过c++这个老伙计了。为了不把老本行丢了,也为了赶紧把书还回去给更多的人阅读。于是下定决心重头开始把它读完。

# 模板中使用class和typename的区别

还没翻几页,当看到这段代码的时候就楞了一下。印象中上次也是看到这里一下子没弄懂,还特地搜索过的。结果再来一遍的时候还是忘了。果然好记性不如烂笔头,赶紧写篇博客mark一下。

这里讲的是作者在声明模板的时候使用typename而不是class。一般情况下,使用typename或者class只是编码风格的问题。但是在遇到从属类型(dependent type)的时候,为了避免潜在的预防解析二义性,必须用typename而不能用class。

```
template<typename C>
bool lastGreaterThanFirst(const C& container)
{
    if(container.empty()) return false;

    typename C::const_iterator begin(container.begin());
    typename C::const_iterator end(container.end());
    return *--end > *begin;
}
```

这里的重点是这两行:

```
typename C::const_iterator begin(container.begin());
typename C::const_iterator end(container.end());
```

如果没有用typename关键字

```
template<typename C>
bool lastGreaterThanFirst(const C& container)
{
    if(container.empty()) return false;

    C::const_iterator begin(container.begin());
    C::const_iterator end(container.end());
    return *--end > *begin;
}
```

就会报错(《Effective STL》中指出有些编译器错误的接受了没有typename的代码,但这样的代码是不可移植的):

```
test.cpp:6:2: error: missing 'typename' prior to dependent type name 'C::const_iterator'
        C::const_iterator begin(container.begin());
        ^~~~~~~~~~~~~~~~~
        typename
test.cpp:7:2: error: missing 'typename' prior to dependent type name 'C::const_iterator'
        C::const_iterator end(container.end());
        ^~~~~~~~~~~~~~~~~
        typename
2 errors generated.
```

如果你在这里使用的是class而不是typename就会报错:

```
test.cpp:8:11: error: elaborated type refers to a typedef
        class C::const_iterator begin(container.begin());
                 ^
test.cpp:15:2: note: in instantiation of function template specialization 'lastGreaterThanFirst<std::__1::vector<int, std::__1::allocator<int> > >' requested here
        lastGreaterThanFirst(vec);
        ^
/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/../include/c++/v1/vector:476:54: note: declared here
    typedef __wrap_iter<const_pointer>               const_iterator;
                                                     ^
1 error generated.
```


# 为什么出现从属类型时需要用typename

我们一步一步来解析。

```
ClassA::foo
```
当你看到上面的代码的时候,你会觉得foo是什么？第一反应应该是ClassA的一个静态成员变量对吧？

那当你继续往后看,看到下面的代码的时候,想想a是什么？

```
ClassA::foo a;
```

a是一个ClassA::foo类型的变量,ClassA::foo是一个内部类:

```
class ClassA {
public:
    class foo {
    };
};
```

或者ClassA内部的一个typedef:

```
class ClassA {
public:
    typedef int foo;
};
```

当foo是ClassA的内部类或者是内部的一个typedef的时候,foo就是一个从属类型。

而对于C::const_iterator,const_iterator可能是C的一个静态成员变量或者是C的一个从属类型,编译器默认是将它解析为一个变量的,所以需要用typename告诉编译器这是一个类型:

```
typename C::const_iterator begin(container.begin());
```

# 出现从属类型时不需要用typename的特例

在遇到从属类型出现在类模板定义中的基类列表的时候,是不需要用typename关键字指明这是一个类型的:

```
class ClassA {
public:
    class foo {
    };
};

template<typename C>
class ClassB : public C::foo {
};
```

因为基类列表中的肯定是一个类型。
