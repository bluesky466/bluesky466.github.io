title: 再探C++自动注册
date: 2024-01-12 22:09:21
tags:
	- 技术相关
	- C++
---

最近的c++项目里面需要使用配置文件配置的值去决定运行时具体实例化的类。如果是java或者kotlin直接使用反射去创建就好,但是c++里没有类似的东西所以只能通过一些取巧的方式实现。

早年间有[研究过GTest的测试用例注册机制](https://blog.islinjw.cn/2016/02/15/GTest%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90-%E6%B5%8B%E8%AF%95%E4%BB%A3%E7%A0%81%E7%9A%84%E6%B3%A8%E5%86%8C/),其原理是利用类的静态成员变量在初始化的时候在它构造函数里面执行注册代码。

包括我们部门现在更常用的[Catch2](https://github.com/catchorg/Catch2)也是类似的,只不过它用的是全局的Catch::AutoReg const常量而不是类静态成员变量:

```c++
#define TEST_CASE( ... ) INTERNAL_CATCH_TESTCASE( __VA_ARGS__ )

#define INTERNAL_CATCH_TESTCASE2( TestName, ... ) \
        static void TestName(); \
        CATCH_INTERNAL_START_WARNINGS_SUPPRESSION \
        CATCH_INTERNAL_SUPPRESS_GLOBALS_WARNINGS \
        CATCH_INTERNAL_SUPPRESS_UNUSED_VARIABLE_WARNINGS \
        namespace{ const Catch::AutoReg INTERNAL_CATCH_UNIQUE_NAME( autoRegistrar )( Catch::makeTestInvoker( &TestName ), CATCH_INTERNAL_LINEINFO, Catch::StringRef(), Catch::NameAndTags{ __VA_ARGS__ } ); } /* NOLINT */ \
        CATCH_INTERNAL_STOP_WARNINGS_SUPPRESSION \
        static void TestName()
    #define INTERNAL_CATCH_TESTCASE( ... ) \
        INTERNAL_CATCH_TESTCASE2( INTERNAL_CATCH_UNIQUE_NAME( CATCH2_INTERNAL_TEST_ ), __VA_ARGS__ )

struct AutoReg : Detail::NonCopyable {
    AutoReg( Detail::unique_ptr<ITestInvoker> invoker, SourceLineInfo const& lineInfo, StringRef classOrMethod, NameAndTags const& nameAndTags ) noexcept;
};

AutoReg::AutoReg( Detail::unique_ptr<ITestInvoker> invoker, SourceLineInfo const& lineInfo, StringRef classOrMethod, NameAndTags const& nameAndTags ) noexcept {
    CATCH_TRY {
        getMutableRegistryHub()
                .registerTest(
                    makeTestCaseInfo(
                        extractClassName( classOrMethod ),
                        nameAndTags,
                        lineInfo),
                    CATCH_MOVE(invoker)
                );
    } CATCH_CATCH_ALL {
        // Do not throw when constructing global objects, instead register the exception to be processed later
        getMutableRegistryHub().registerStartupException();
    }
}
```

所以我们可以类似的去定义一个AutoRegister类在构造函数里面将类的工厂函数注册给Factory:

```c++
// factory.h
typedef std::shared_ptr<IComponent> (*CreatorFunc)();

class Factory {
public:
	static Factory& Instance();
	
	Factory();

	void RegisterCreatorFunc(const std::string& name, CreatorFunc func);

	std::shared_ptr<IComponent> Create(const std::string& name);

private:
	std::map<std::string, CreatorFunc> creators_;
};

class AutoRegister {
public:
	AutoRegister(const std::string& name, CreatorFunc func);
};

// factory.cpp
void Factory::RegisterCreatorFunc(const std::string& name, CreatorFunc func) {
	creators_[name] = func;
}

std::shared_ptr<IComponent> Factory::Create(const std::string& name) {
	auto it = creators_.find(name);
    if (it == creators_.end()) {
        std::cout<<"Unknown : " << name <<std::endl;
        return nullptr;
    }
    auto instance = it->second();
    return instance;
}

AutoRegister::AutoRegister(const std::string& name, CreatorFunc func) {
    Factory::Instance().RegisterCreatorFunc(name, func);
}
```

然后定义一个宏去协助声明全局变量:

```c++
#define AUTO_REGISTER(NAME) \
static std::shared_ptr<IComponent> __##NAME##_ComponentCreatorFunc(){return std::make_shared<NAME>();} \
const AutoRegister __##NAME##_ComponentRegister(#NAME, __##NAME##_ComponentCreatorFunc);

typedef std::shared_ptr<IComponent> (*CreatorFunc)();
```

最后只需要在定义IComponent子类的时候使用AUTO\_REGISTER注明一下需要自动注册即可:

```c++
// component_a.h
class ComponentA : public IComponent {
public:
	void Init() override;
};

AUTO_REGISTER(ComponentA)

// component_a.cpp
void ComponentA::Init() {
	std::cout << "ComponentA::Init()" <<std::endl;
}
```

原理和实现都比较简单,无法是利用全局变量在main函数执行前初始化的机制,在全局变量的构造函数里面做事情。但实际使用的时候会有不少的坑。

# 全局变量的初始化顺序不确定

正如我之前的[博客](https://blog.islinjw.cn/2016/02/15/GTest%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90-%E6%B5%8B%E8%AF%95%E4%BB%A3%E7%A0%81%E7%9A%84%E6%B3%A8%E5%86%8C/#%E4%B8%80%E7%A7%8D%E6%9C%89%E9%97%AE%E9%A2%98%E7%9A%84%E6%96%B9%E6%B3%95)讨论的全局变量的初始化顺序时是不能确定的。

```c++
static const std::string kGlobalStr = "GlobalStr";
#define GLOBAL_STR "GLOBAL_STR"

Factory& Factory::Instance() {
	static Factory instance;
	return instance;
}

Factory::Factory() {
    std::cout << "kGlobalStr : " << kGlobalStr << std::endl;
    std::cout << "GLOBAL_STR : " << GLOBAL_STR << std::endl;
}
```

单例的instance我们可以通过局部静态变量的方式保证在第一次调用Factory::Instance的时候会初始化,但是如果在Factory的构造函数里面使用了其他的全局变量可能就会拿到还未初始化的变量:

```
kGlobalStr :
GLOBAL_STR : GLOBAL_STR
```

我们可以用宏或者再写个函数把全局变量变成局部静态变量包在函数里面去解决。

# 静态库依赖的情况下不会自动注册

还有另外一个问题是如果我们的组件是放在静态库里面去依赖的话,没有include的全局变量不会初始化:

```cmake
add_library(${PROJECT_NAME}-Lib STATIC
    ../factory.cpp
    ../component_a.cpp
    ../component_b.cpp
)

add_executable(${PROJECT_NAME}
    ../main.cpp
)

target_link_libraries(${PROJECT_NAME} ${PROJECT_NAME}-Lib)
```

```c++
#include "factory.h"
#include "component_b.h"

int main(int argc,char **argv) {
	// 没有#include "component_a.h", 静态库依赖的时候ComponentA没有自动注册
	auto a = Factory::Instance().Create("ComponentA");
	if(a != nullptr) {
		a->Init(); 
	}

	// 有#include "component_a.h", 静态库依赖的时候ComponentB有自动注册
	auto b = Factory::Instance().Create("ComponentB");
	if(b != nullptr) {
		b->Init(); 
	}
	return 0;
}
```

```
kGlobalStr :
GLOBAL_STR : GLOBAL_STR
Unknown : ComponentA
ComponentB::Init() - 0x600001848288
```

在我们的事件项目中把Catch2编写的测试用例改成静态库依赖同样也会出现Catch2找不到测试用例的问题,网上没有找到什么太清晰的解答,但是chatgpt给了下面的回答:

```
关于全局静态变量的初始化行为，C和C++标准并没有明确规定。具体的行为可能会因编译器和链接器的实现而有所不同。

然而，根据常见的编译器和链接器的实践，可以得出以下一般性规则：

1. 静态库：
   - 在静态库中，未被引用的全局静态变量通常不会被初始化。
   - 这是因为链接器会进行优化，只将被引用的目标文件和符号与主程序进行链接，未被引用的目标文件和符号会被优化掉。

2. 源码依赖：
   - 在源码依赖的情况下，全局静态变量通常会被初始化，无论是否被引用。
   - 这是因为编译器会对源码进行逐行解析和编译，将所有的全局静态变量初始化代码转换为可执行的指令。

虽然没有明确的权威文件规定这些行为，但这些规则是根据广泛的实践和经验总结得出的。

如果您对特定编译器和链接器的行为有疑问，建议查阅它们的官方文档或相关规范，以了解更具体的行为和规定。不同的编译器和链接器可能会有不同的实现和行为。
```

在mac上使用nm名搜索符合表,也的确可以看到没有__ComponentA\_ComponentRegister这个符号:

```
nm ./Demo| grep -e ComponentA_ComponentRegister -e ComponentB_ComponentRegister
0000000100010040 b __ZL30__ComponentB_ComponentRegister
0000000100010000 b __ZL30__ComponentB_ComponentRegister
```

# enable\_shared\_from\_this的问题

这个问题和自动注册没有直接关系,但是设计不好可能会遇到所以就放一起了。假如我们的工厂函数返回值不是shared\_ptr<IComponent>而是IComponent*,那么子类在使用enable\_shared\_from\_this的时候就会出现问题:

```c++
// factory.h
#define AUTO_REGISTER(NAME) \
static IComponent* __##NAME##_ComponentCreatorFunc(){return new NAME();} \
const AutoRegister __##NAME##_ComponentRegister(#NAME, __##NAME##_ComponentCreatorFunc);

typedef IComponent* (*CreatorFunc)();

// factory.cpp
std::shared_ptr<IComponent> Factory::Create(const std::string& name) {
	auto it = creators_.find(name);
    if (it == creators_.end()) {
        std::cout<<"Unknown : " << name <<std::endl;
        return nullptr;
    }
    auto instance = it->second();
    return std::shared_ptr<IComponent>(instance);
}

// component_b.h
class ComponentB : public IComponent, public std::enable_shared_from_this<ComponentB> {
public:
	void Init() override;
};

AUTO_REGISTER(ComponentB)

// component_b.cpp
void ComponentB::Init() {
	std::cout << "ComponentB::Init() - " << this->shared_from_this() <<std::endl; // 抛出bad_weak_ptr异常
}
```

调用到ComponentB::Init之后就会抛出异常:

```
libc++abi: terminating with uncaught exception of type std::__1::bad_weak_ptr: bad_weak_ptr
```

从源码来看用裸指针创建shared\_ptr的时候会调用\_\_enable\_weak\_this:

```c++
template<class _Yp>
    explicit shared_ptr(_Yp* __p,
                        typename enable_if<__compatible_with<_Yp, element_type>::value, __nat>::type = __nat());

template<class _Tp>
template<class _Yp>
shared_ptr<_Tp>::shared_ptr(_Yp* __p,
                            typename enable_if<__compatible_with<_Yp, element_type>::value, __nat>::type)
    : __ptr_(__p)
{
    unique_ptr<_Yp> __hold(__p);
    typedef typename __shared_ptr_default_allocator<_Yp>::type _AllocT;
    typedef __shared_ptr_pointer<_Yp*, __shared_ptr_default_delete<_Tp, _Yp>, _AllocT > _CntrlBlk;
    __cntrl_ = new _CntrlBlk(__p, __shared_ptr_default_delete<_Tp, _Yp>(), _AllocT());
    __hold.release();
    __enable_weak_this(__p, __p);
}
```

只有裸指针的类型是enable\_shared\_from\_this的时候才会去初始化它的\_\_weak_this\_:

```c++
template <class _Yp, class _OrigPtr>
    _LIBCPP_INLINE_VISIBILITY
    typename enable_if<is_convertible<_OrigPtr*,
                                      const enable_shared_from_this<_Yp>*
    >::value,
        void>::type
    __enable_weak_this(const enable_shared_from_this<_Yp>* __e,
                       _OrigPtr* __ptr) _NOEXCEPT
    {
        typedef typename remove_cv<_Yp>::type _RawYp;
        if (__e && __e->__weak_this_.expired())
        {
            __e->__weak_this_ = shared_ptr<_RawYp>(*this,
                const_cast<_RawYp*>(static_cast<const _Yp*>(__ptr)));
        }
    }

_LIBCPP_INLINE_VISIBILITY void __enable_weak_this(...) _NOEXCEPT {}
```

也就是说如果子类继承了enable\_shared\_from\_this,但是转换成非enable\_shared\_from\_this的基类指针去创建shared\_ptr由于已经不是enable\_shared\_from\_this,所以根据入参会匹配到`_LIBCPP_INLINE_VISIBILITY void __enable_weak_this(...) _NOEXCEPT {}`这个空实现。未对\_\_weak\_this\_赋值。

所以在子类里面调用`this->shared\_from\_this()`就会因为\_\_r.\_\_cntrl\_为0抛出bad\_weak\_ptr异常:

```c++
// enable_shared_from_this:
hared_ptr<_Tp const> shared_from_this() const
        {return shared_ptr<const _Tp>(__weak_this_);}


// shared_ptr:
template<class _Tp>
template<class _Yp>
shared_ptr<_Tp>::shared_ptr(const weak_ptr<_Yp>& __r,
                            typename enable_if<is_convertible<_Yp*, element_type*>::value, __nat>::type)
    : __ptr_(__r.__ptr_),
      __cntrl_(__r.__cntrl_ ? __r.__cntrl_->lock() : __r.__cntrl_)
{
    if (__cntrl_ == 0)
        __throw_bad_weak_ptr();
}
```

从这个案例我们得到的教训就是使用智能指针就都是用智能指针,不要和裸指针混着用,哪天就踩坑了。


# 总结

技术选型的时候要充分了解技术的原理和可能出现的问题，然后根据具体业务场景去考虑是否合适。

例如这里的自动注册技术用于单元测试的测试用例注册是合适的，在单元测试的场景里不会遇到全局变量初始化问题，正常情况下也不会把测试用例写在单独的静态库里而是直接编译测试的可执行程序。

而对我们这个项目的场景就不合适了，因为两者我们都实际有可能遇到问题。当然可以在文档里面写明最佳实践方式，但是一旦不小心后人就会踩坑。

所以我会选择稳妥点，在一处地方集中手动编写注册代码。虽然这样每加一个组件除了.h和.cpp还需要再在另外的地方添加注册代码，但起码不会留意料之外的坑。


# 完整代码

完整的demo代码可以见[Github](https://github.com/bluesky466/CppAutoRegisterDemo)