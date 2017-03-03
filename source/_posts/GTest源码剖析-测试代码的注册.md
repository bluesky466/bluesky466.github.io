title: GTest源码剖析 - 测试代码的注册
date: 2016-02-15 22:07:08
tags:
	- 技术相关
	- 单元测试
	- c++
---
单元测试框架，最基本的功能当然就是运行用户所编写的测试用例了。

## __毫无技巧的方法__

一种毫无技巧的方法就是用户手动在 main 函数里面将自己编写的测试代码注册到框架中，就像下面的代码：
```cpp
    void test1() {...}
    void test2() {...}
    void test3() {...}
    ...

    int main(){
        RegisterTestFunc(test1);
        RegisterTestFunc(test2);
        RegisterTestFunc(test3);
        ...
        return 0;
    }
```

这样的代码虽然可以运行，但是将初始化的责任放到的用户那里，这样的代码是不够优秀的。有两种容易出现的情况：一是项目中可能拥有大量的测试代码，用户很有可能会漏掉其中的部分测试代码，忘记把它们注册到测试框架中。二是可能用户去掉了一些测试代码，却又忘了去掉注册的代码。

后者编译器会报错，但前者却没有办法检测（除非对着测试结果一条条的检测，看是否所有测试代码都运行了）。

## __一种有问题的方法__

最好在编写测试代码的时候就能通过一种机制帮用户注册，而不用用户手动去注册。面对这个需求，我脑海里面想到的第一个方法就是利用全局变量和宏定义。

首先定义一个用来管理注册的测试方法的类：
```cpp
	typedef list<function<void()>> FuncList;
    class Test{
    public:
        Test(const function<void()>& test_func){
            test_funcs_.push_back(test_func);
        }

        static void runAllTest(){
            for (auto func : test_funcs_){
                func();
            }
        }

    private:
        static FuncList test_funcs_;
    };
```

它有一个静态的成员变量 test\_funcs\_ ，用来保存测试方法，同时它有一个构造函数用来将传入的测试方法插入 test\_funcs\_ 中

接着定义一个宏：
```cpp
    #define TEST_FUNC(NAME) \
        void NAME(); \
        static Test register_##NAME(NAME); \
        void NAME()
```
它在帮助我们在定义一个测试方法的时候自动注册到 test\_funcs\_ 中。原理其实很简单，就是在声明一个函数的同时声明一个 Test 全局变量，将定义的测试方法传入，这个测试函数就会在 Test 的构造函数中被插入 test\_funcs\_ 。

所以我们只要这样编写测试代码，就能实现自动注册了：
```cpp
	TEST_FUNC(testSomething){
        ...
    }
```

看起来这个方法不错是吧？可惜这种方法是有问题的！至少在我的 vs2013 上会崩溃！

问题就出在 Test 全局变量和 Test::test\_funcs\_ 的初始化顺序上。你无法保证 Test::test\_funcs\_ 比全局变量 Test 先初始化。很奇怪是吧？ Test 的静态成员变量居然比 Test 全局变量的初始化时间晚，也就是说在 Test 这个类还没有完全准备好的时候，就已经拿来创建一个全局变量了。书上一直强调的全局变量的初始化顺序不能确定难道也有这种含义？

## __一种可能可行的方法__

既然是因为初始化顺序导致了内存错误，那我们只要使用某种机制让保存测试函数的容器首先初始化就行了。

让我们将 Test 类的定义修改成下面的样子：

```cpp
    class Test{
    public:
        virtual void run() = 0;

        static void runAllTest(){
            for (auto i : test_list){
                i->run();
            }
        }

    protected:
        static void addTest(Test* test){
            test_list.push_back(test);
        }

    private:
        static list<Test*> test_list;
    };
```

容器里面不再直接放测试函数，改为放 Test 的指针。而 Test 又是一个抽象类，所以事实上放的是 Test 的子类。

再把 TEST_FUNC 宏的定义改成下面的样子：

```cpp
    #define TEST_FUNC(NAME) \
        class NAME : public Test{ \
        public: \
            virtual void run(); \
        private: \
            NAME(){ \
                addTest(this); \
            } \
            static NAME* instance_; \
        }; \
        NAME* NAME::instance_ = new NAME(); \
        void NAME::run()

```

现在实际上用户写的测试方法实现的是 Test 的子类的 run 方法。

依然是需要在定义测试方法的时候顺便定义一个全局变量，但我们换了一种方式，定义了一个类静态变量。子类在构造函数中把自己注册到 Test 的测试容器中，而且子类还包含了一个本类指针静态成员变量（有点拗口，但看代码很容易看出来）。在子类的静态成员变量初始化的之前，父类的静态成员变量应该就已经初始化了。就是根据这种机制，达到了我们的目的。

使用方法还是一样：

```cpp
    TEST_FUNC(testSomething){
        ...
    }
```

为什么说这是“一种可能可行的方法”呢？因为父类的静态成员变量初始化先于子类的静态成员变量初始化这个前提是我自己推论的。可能是我读的书少或者读书不仔细，至今没有在哪里见到有提及父类和子类的静态成员变量的初始化顺序的。所以虽然在我的编译器上它的确能正常的工作，但为了严谨起见，姑且称为“可能”的吧。如果有人有在哪里看到这方面的描述，请务必私信我，让我把“可能”二字去掉或者将标题改成“另一种有问题的方法”

## __GTest 的做法__

讲了这么久我的想法，现状让我们来看看谷歌的大神们是怎么做的吧。

我们从 TEST 宏看起：

```cpp
    #if !GTEST_DONT_DEFINE_TEST
    # define TEST(test_case_name, test_name) GTEST_TEST(test_case_name, test_name)
    #endif
```

这里这么搞，主要是为了防止 TEST 被系统或者其他框架定义了。如果出现这种情况，只要把GTEST\_DONT\_DEFINE\_TEST 定义为 1，之后编写测试用例的时候直接使用 GTEST\_TEST 就好了。不得不说，他们考虑的真仔细。让我们继续跟踪，看 GTEST\_TEST：

```cpp
    #define GTEST_TEST(test_case_name, test_name)\
      GTEST_TEST_(test_case_name, test_name, \
                  ::testing::Test, ::testing::internal::GetTestTypeId())
```

GTEST\_TEST 宏又用到了另一个宏 GTEST\_TEST\_，但我想先说一下 GetTestTypeId，这个东西的用法真的令我眼前一亮，不得不佩服：

```cpp
    TypeId GetTestTypeId() {
      return GetTypeId<Test>();
    }

	...

	template <typename T>
    TypeId GetTypeId() {
      // The compiler is required to allocate a different
      // TypeIdHelper<T>::dummy_ variable for each T used to instantiate
      // the template.  Therefore, the address of dummy_ is guaranteed to
      // be unique.
      return &(TypeIdHelper<T>::dummy_);
    }

	...

    template <typename T>
    class TypeIdHelper {
     public:
      // dummy_ must not have a const type.  Otherwise an overly eager
      // compiler (e.g. MSVC 7.1 & 8.0) may try to merge
      // TypeIdHelper<T>::dummy_ for different Ts as an "optimization".
      static bool dummy_;
    };
```

这里直接用一个类的静态成员变量的地址当作 id 号。当时我就懵逼了，明明很简单，怎么就感觉那么玄幻呢？

膜拜完我们再继续看 GTEST\_TEST\_：
```cpp
    #define GTEST_TEST_(test_case_name, test_name, parent_class, parent_id)\
    class GTEST_TEST_CLASS_NAME_(test_case_name, test_name) : public parent_class {\
     public:\
      GTEST_TEST_CLASS_NAME_(test_case_name, test_name)() {}\
     private:\
      virtual void TestBody();\
      static ::testing::TestInfo* const test_info_ GTEST_ATTRIBUTE_UNUSED_;\
      GTEST_DISALLOW_COPY_AND_ASSIGN_(\
          GTEST_TEST_CLASS_NAME_(test_case_name, test_name));\
    };\
    \
    ::testing::TestInfo* const GTEST_TEST_CLASS_NAME_(test_case_name, test_name)\
      ::test_info_ =\
        ::testing::internal::MakeAndRegisterTestInfo(\
            #test_case_name, #test_name, NULL, NULL, \
            (parent_id), \
            parent_class::SetUpTestCase, \
            parent_class::TearDownTestCase, \
            new ::testing::internal::TestFactoryImpl<\
                GTEST_TEST_CLASS_NAME_(test_case_name, test_name)>);\
    void GTEST_TEST_CLASS_NAME_(test_case_name, test_name)::TestBody()
```

这个宏的做法和我的最后一个方法的 TEST\_FUNC 宏差不多，用户写的测试函数实际上是实现了 ::testing::Test 的子类的 TestBody 方法。也是初始化了子类的一个静态成员变量，但GTest这里没有我那么暴力，它初始化的是一个 TestInfo 类型的的静态成员变量，这里面包含了测试的很多信息。其中最重要的是 ::testing::internal::TestFactoryImpl 这个东西：

```cpp
    template <class TestClass>
    class TestFactoryImpl : public TestFactoryBase {
     public:
      virtual Test* CreateTest() { return new TestClass; }
    };

    ...

	class TestFactoryBase {
     public:
      virtual ~TestFactoryBase() {}

      // Creates a test instance to run. The instance is both created and destroyed
      // within TestInfoImpl::Run()
      virtual Test* CreateTest() = 0;

     protected:
      TestFactoryBase() {}

     private:
      GTEST_DISALLOW_COPY_AND_ASSIGN_(TestFactoryBase);
    };
```

它是一个工厂类，用来创建传入的测试类的实例，也就是 GTEST\_TEST\_CLASS\_NAME\_(test\_case\_name, test\_name)> 这个类，它的 TestBody 就是用户所写的测试代码。可以看看 GTEST\_TEST\_CLASS\_NAME\_ 的定义：

```cpp
    #define GTEST_TEST_CLASS_NAME_(test_case_name, test_name) \
      test_case_name##_##test_name##_Test
```

ok,很简单是吧？就是字符串拼接而已。

好了，让我们继续深入，看看 MakeAndRegisterTestInfo ：
```cpp
	//gtest.cc
    TestInfo* MakeAndRegisterTestInfo(
        const char* test_case_name,
        const char* name,
        const char* type_param,
        const char* value_param,
        TypeId fixture_class_id,
        SetUpTestCaseFunc set_up_tc,
        TearDownTestCaseFunc tear_down_tc,
        TestFactoryBase* factory) {
      TestInfo* const test_info =
          new TestInfo(test_case_name, name, type_param, value_param,
                       fixture_class_id, factory);
      GetUnitTestImpl()->AddTestInfo(set_up_tc, tear_down_tc, test_info);
      return test_info;
    }

	//gtest-internal-inl.h
	inline UnitTestImpl* GetUnitTestImpl() {
      return UnitTest::GetInstance()->impl();
    }

    //gtest.h
    class GTEST_API_ UnitTest {
     public:
      static UnitTest* GetInstance();
      ...
      internal::UnitTestImpl* impl() { return impl_; }
      ...
      internal::UnitTestImpl* impl_;
      ...
    };

    //gtest.cc
    UnitTest* UnitTest::GetInstance() {
      // When compiled with MSVC 7.1 in optimized mode, destroying the
      // UnitTest object upon exiting the program messes up the exit code,
      // causing successful tests to appear failed.  We have to use a
      // different implementation in this case to bypass the compiler bug.
      // This implementation makes the compiler happy, at the cost of
      // leaking the UnitTest object.

      // CodeGear C++Builder insists on a public destructor for the
      // default implementation.  Use this implementation to keep good OO
      // design with private destructor.

    #if (_MSC_VER == 1310 && !defined(_DEBUG)) || defined(__BORLANDC__)
      static UnitTest* const instance = new UnitTest;
      return instance;
    #else
      static UnitTest instance;
      return &instance;
    #endif  // (_MSC_VER == 1310 && !defined(_DEBUG)) || defined(__BORLANDC__)
    }

    //gtest-internal-inl.h
    class GTEST_API_ UnitTestImpl {
    	...
        void AddTestInfo(Test::SetUpTestCaseFunc set_up_tc,
                   Test::TearDownTestCaseFunc tear_down_tc,
                   TestInfo* test_info) {
        // In order to support thread-safe death tests, we need to
        // remember the original working directory when the test program
        // was first invoked.  We cannot do this in RUN_ALL_TESTS(), as
        // the user may have changed the current directory before calling
        // RUN_ALL_TESTS().  Therefore we capture the current directory in
        // AddTestInfo(), which is called to register a TEST or TEST_F
        // before main() is reached.
        if (original_working_dir_.IsEmpty()) {
          original_working_dir_.Set(FilePath::GetCurrentDir());
          GTEST_CHECK_(!original_working_dir_.IsEmpty())
              << "Failed to get the current working directory.";
        }

        GetTestCase(test_info->test_case_name(),
                    test_info->type_param(),
                    set_up_tc,
                    tear_down_tc)->AddTestInfo(test_info);
      }
      ...
      //这个方法从test_cases_里面获取TestCase
      TestCase* GetTestCase(const char* test_case_name,
                        const char* type_param,
                        Test::SetUpTestCaseFunc set_up_tc,
                        Test::TearDownTestCaseFunc tear_down_tc);
      ...
      std::vector<TestCase*> test_cases_;
      ...
    };

    //gtest.cc
    void TestCase::AddTestInfo(TestInfo * test_info) {
      test_info_list_.push_back(test_info);
      test_indices_.push_back(static_cast<int>(test_indices_.size()));
    }

    //gtest.h
    class GTEST_API_ TestCase {
    	...
    	std::vector<TestInfo*> test_info_list_;
        std::vector<int> test_indices_;
        ...
    };
```

代码很多，我简单的描述一下。UnitTest 是一个单例类，它有一个成员变量 internal::UnitTestImpl* impl\_， impl\_ 里面又有成员变量 test\_info\_list\_。最终我们写的测试类就放在 test\_info\_list\_ 里。

九曲十八弯，实际 GTest 用一个单例类 UnitTest 保存了注册的测试代码（放在 ::testing::Test 子类的 TestBody 方法里面）。

那他是怎么解决初始化顺序的问题的？注意看 UnitTest::GetInstance() 方法：

```cpp
	//gtest.cc
    UnitTest* UnitTest::GetInstance() {
      // When compiled with MSVC 7.1 in optimized mode, destroying the
      // UnitTest object upon exiting the program messes up the exit code,
      // causing successful tests to appear failed.  We have to use a
      // different implementation in this case to bypass the compiler bug.
      // This implementation makes the compiler happy, at the cost of
      // leaking the UnitTest object.

      // CodeGear C++Builder insists on a public destructor for the
      // default implementation.  Use this implementation to keep good OO
      // design with private destructor.

    #if (_MSC_VER == 1310 && !defined(_DEBUG)) || defined(__BORLANDC__)
      static UnitTest* const instance = new UnitTest;
      return instance;
    #else
      static UnitTest instance;
      return &instance;
    #endif  // (_MSC_VER == 1310 && !defined(_DEBUG)) || defined(__BORLANDC__)
    }
```

这里使用了局部静态变量，在第一次进入这个方法的时候就会生成一个 UnitTest 实例！不需要靠人品祈祷编译器按照我们设想的顺序创建全局变量！

谷歌大神们不愧是大神，在看 GTest 源码的时候我都不知道被惊艳了多少次，真心学到了不少东西。怪不得别人都说看源码才是最好的提升方式。