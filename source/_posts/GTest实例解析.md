title: GTest实例解析
date: 2016-02-08 17:01:49
tags:
	- 技术相关
	- 单元测试
	- c++
---
gtest是google的一套开源的c++单元测试框架，可以方便程序员对自己的代码进行单元测试。学习这套框架，除了墙外的官方文档之外，我强力推荐[玩转Google开源C\+\+单元测试框架Google Test系列](http://www.cnblogs.com/coderzh/archive/2009/04/06/1426755.html)。

这系列的博客已经将gtest讲的十分详细了，所以我这篇博客就不再详细介绍gtest的基本使用方法了，而是通过一个简单的例子，介绍一下如何在实际的项目中使用它。同时也会通过分析gtest的源代码，使得大家能够更好的理解gtest的工作原理

## __需要测试的类__

首先我实现了一个简单的类TwoDimensionalMark，它的功能相当于bool二维数组，可以将二维下标标记为true或者false。只不过它内部使用位运算，一个字节可以记录八个标记数据，可以大量节省内存。它的声明如下：

```cpp
    class TwoDimensionalMark{
        public:
            TwoDimensionalMark(int row, int col, bool flag = false);
            TwoDimensionalMark(const TwoDimensionalMark& cpy)；
            ~TwoDimensionalMark();
            const TwoDimensionalMark& operator =(const TwoDimensionalMark& cpy)；

            //将所有的位置标记为true或者false
            void clean(bool mark);

            //设置(x,y)下标为true或者false
            void set(int x, int y, bool mark);

            //获取(x,y)下标的数据
            bool check(int x, int y)const;

            int getRow();
            int getCol();

            ...
    };
```

## __完整的测试代码__
我先把完整的测试代码放在这里，可以先简单浏览一遍。
```cpp
    #include "TwoDimensionalMark.h"
    #include "gtest/gtest.h"
    #include <cstdlib>
    #include <vector>

    using namespace std;

    struct MarkSize{
        MarkSize(int r, int c): row(r),col(c){}
        int row, col;
    };
    class TestWithMarkSize : public testing::TestWithParam<MarkSize>{
    public:
        struct Coord{
            int x, y;
        };

        vector<vector<bool> > CreateContrast(int row, int col, bool flag){
            vector<vector<bool> > contrast(row);
            for (int i = 0; i < row; i++){
                contrast[i] = vector<bool>(col, flag);
            }
            return contrast;
        }

        vector<Coord> CreateRandCoords(int row, int col){
            int yRange = row * 3;
            int xRange = col * 3;
            int numRandCoord = row * col / 2;

            vector<Coord> coords;
            for (int i = 0; i < numRandCoord; i++){
                int x = (rand() % xRange) - col;
                int y = (rand() % yRange) - row;
                coords.push_back({ x, y });
            }

            return coords;
        }

        void SetFlag(TwoDimensionalMark* mark,  vector<vector<bool> >* contrast,  
                    const MarkSize& size, const vector<Coord>& coords, bool flag){
            for (auto c : coords){
                mark->set(c.x, c.y, flag);
                //随机位置有可能在row，col之外，要防止溢出
                if (c.x >= 0 && c.x < size.col && c.y >= 0 && c.y < size.row){
                    (*contrast)[c.y][c.x] = flag;
                }
            }
        }
    };

    //行数多于列数的情况
    INSTANTIATE_TEST_CASE_P(RowLTCol, TestWithMarkSize,	testing::Values(MarkSize(200, 100), MarkSize(50, 30)));
    //行数少于列数的情况
    INSTANTIATE_TEST_CASE_P(ColLTRow, TestWithMarkSize,	testing::Values(MarkSize(10, 100), MarkSize(3, 10)));
    //行数等于列数的情况
    INSTANTIATE_TEST_CASE_P(RowEQCol, TestWithMarkSize,	testing::Values(MarkSize(100, 100), MarkSize(3, 3)));


    TEST_P(TestWithMarkSize,testWithTrueClean){
        MarkSize size = GetParam();

        //初始化TwoDimensionalMark和用来做对比的contrast,将他们全部用true填充
        TwoDimensionalMark mark(size.row, size.col);
        mark.clean(true);
        vector<vector<bool> > contrast = CreateContrast(size.row, size.col, true);

        //验证mark和contrast里面都是true
        for (int i = 0; i < size.row; i++){
            for (int j = 0; j < size.col; j++){
                ASSERT_TRUE(mark.check(j, i));
                ASSERT_TRUE(contrast[i][j]);
            }
        }

        //随机将一些坐标位置,这个函数生成的随机位置有可能在row，col之外
        vector<Coord> coords = CreateRandCoords(size.row, size.col);

        //将mark和contrast的之前随机生成的位置设为false
        SetFlag(&mark, &contrast, size, coords, false);

        //检测随机生成的位置上的值有没有设置正确
        //mark若位置超出范围则返回false,而contrast则需要防止溢出
        for (auto c : coords){
            ASSERT_FALSE(mark.check(c.x,c.y));
            if (c.x >= 0 && c.x < size.col && c.y >= 0 && c.y < size.row){
                ASSERT_FALSE(contrast[c.y][c.x]);
            }
        }

        //对比contrast和mark，两者应该相等
        for (int y = 0; y < size.row; y++)
            for (int x = 0; x < size.col; x++){
                ASSERT_EQ(contrast[y][x], mark.check(x, y));
            }
    }

    TEST_P(TestWithMarkSize, testWithFalseClean){
        MarkSize size = GetParam();

        //初始化TwoDimensionalMark和用来做对比的contrast,将他们全部用false填充
        TwoDimensionalMark mark(size.row, size.col);
        mark.clean(false);
        vector<vector<bool> > contrast = CreateContrast(size.row, size.col, false);

        //验证mark和contrast里面都是false
        for (int i = 0; i < size.row; i++){
            for (int j = 0; j < size.col; j++){
                ASSERT_FALSE(mark.check(j, i));
                ASSERT_FALSE(contrast[i][j]);
            }
        }

        //随机将一些坐标位置,这个函数生成的随机位置有可能在row，col之外
        vector<Coord> coords = CreateRandCoords(size.row, size.col);

        //将mark和contrast的之前随机生成的位置设为true
        SetFlag(&mark, &contrast, size, coords, true);

        //检测随机生成的位置上的值有没有设置正确
        //mark若位置超出范围则返回false,而contrast则需要防止溢出
        for (auto c : coords){
            if (c.x >= 0 && c.x < size.col && c.y >= 0 && c.y < size.row){
                ASSERT_TRUE(contrast[c.y][c.x]);
                ASSERT_TRUE(mark.check(c.x, c.y));
            }
            else{
                ASSERT_FALSE(mark.check(c.x, c.y));
            }
        }
    }

    int main(int argc, char* argv[])
    {
        testing::InitGoogleTest(&argc, argv);
        int result =  RUN_ALL_TESTS();
        getchar();
        return result;
    }
```

## __传入多个参数__

为了覆盖各种大小的情况（行和列数量相等，行多于列，行少于列），需要定义多个不同行列数的TwoDimensionalMark。但如果每个测试用例都手动硬编码的话将会有许多的重复代码，这个时候就可以使用参数化的方法去将行列数作为参数传入各个测试用例中，详细的介绍看[这里](http://www.cnblogs.com/coderzh/archive/2009/04/08/1431297.html)。

这篇博客只写到了一个参数的处理。但我们这里需要传入行数和列数两个参数应该怎么办？
其实很方法也很简单，首先定义一个结构体MarkSize用于保存行数和列数

```cpp
    struct MarkSize{
        MarkSize(int r, int c): row(r),col(c){}
        int row, col;
    }
```

用它作为TestWithParam的模板参数，再声明一个类继承于它。

```cpp
    class TestWithMarkSize : public testing::TestWithParam<MarkSize>{
    }
```

然后传入参数，这里我定义了三组参数，每组两个。分别对应行数比较多，列数比较多，行数列数一样多三种情况：

```cpp
    //行数多于列数的情况
    INSTANTIATE_TEST_CASE_P(RowLTCol, TestWithMarkSize,	testing::Values(MarkSize(200, 100), MarkSize(50, 30)));
    //行数少于列数的情况
    INSTANTIATE_TEST_CASE_P(ColLTRow, TestWithMarkSize,	testing::Values(MarkSize(10, 100), MarkSize(3, 10)));
    //行数等于列数的情况
    INSTANTIATE_TEST_CASE_P(RowEQCol, TestWithMarkSize,	testing::Values(MarkSize(100, 100), MarkSize(3, 3)));
```

这样一来，只要是 test_case_name 为 TestWithMarkSize 的测试用例都可以使用GetParam方法获取传入的参数了：

```cpp
    TEST_P(TestWithMarkSize,testWithTrueClean){
        MarkSize size = GetParam();
        ...
    }

    TEST_P(TestWithMarkSize, testWithFalseClean){
		MarkSize size = GetParam();
		...
	}
```


## __TEST\_P源码解析__

TEST\_P 宏的定义如下

```cpp
    # define TEST_P(test_case_name, test_name) \
      class GTEST_TEST_CLASS_NAME_(test_case_name, test_name) \
          : public test_case_name { \
      ...
      }; \
      ...
      void GTEST_TEST_CLASS_NAME_(test_case_name, test_name)::TestBody()
```

GTEST_TEST_CLASS_NAME_又是个什么东西？其实它的功能十分简单，就是将类名拼接出来而已，它的定义如下：

```cpp
    #define GTEST_TEST_CLASS_NAME_(test_case_name, test_name) \
      test_case_name##_##test_name##_Test
```


所以我们使用 TEST_P(TestWithMarkSize,testWithTrueClean){...} 实际上就是定义了一个 TestWithMarkSize 的子类 TestWithMarkSize_testWithTrueClean_Test。

而花括号里面实际上就是void TestWithMarkSize_testWithTrueClean_Test::TestBody()的实现。

也就是说，实际上我们的测试用例都是TestWithMarkSize的子类。所以我们可以将一些公共的方法和数据结构定义在TestWithMarkSize内，只要将他们声明为protected或者public，就能在TEST_P(TestWithMarkSize,testWithTrueClean){...} 和 TEST_P(TestWithMarkSize,testWithFalseClean){...} 内使用。

这样既可以提炼重复代码，又能将它们的作用范围限定在测试用例中，防止提炼出来的函数或者定义的数据结构放在全局影响实际的功能代码。

如我这里定义的 Coord 结构体和CreateContrast、CreateRandCoords 和 SetFlag 方法。

而GetParam()的定义如下：

```cpp
    static const ParamType* parameter_;

    const ParamType& GetParam() const {
        GTEST_CHECK_(parameter_ != NULL)
            << "GetParam() can only be called inside a value-parameterized test "
            << "-- did you intend to write TEST_P instead of TEST_F?";
        return *parameter_;
    }
```

既然 const T* WithParamInterface<T>::parameter_ 是有一个类静态成员变量，那就不难理解为什么所有继承于 TestWithMarkSize 的测试用例都能拿到同样的参数了。

## __测试用例分析__

要知道 TwoDimensionalMark 实际上的功能是和bool二维数组基本一样的，所以我们就很自然的想到使用一个对比用的bool二维数组作为参照，去测试 TwoDimensionalMark 的功能究竟有没有bug。

TEST_P(TestWithMarkSize,testWithTrueClean) 的测试分下面三个步骤：

1.根据传入的数组大小，创建了一个 TwoDimensionalMark 和用两重 vector 实现的 bool 二维数组。

2.将它们全部用true填充，然后验证每一个下标的数据是否均为 true，如此去测试TwoDimensionalMark::clean(true) 的功能

3.随机生成一些坐标，将 TwoDimensionalMark 和 bool 二维数组对应这些坐标的数据都设为 false 。然后检测设置的位置的数据是否都为false，还有对比两者每个下标的数据是否相等。以测试 set 和 check 方法是否正确。同时因为这些下标有些是超出范围之外的，也能测试 TwoDimensionalMark 对超出范围的操作是否正确

代码如下：

```cpp
    TEST_P(TestWithMarkSize,testWithTrueClean){
        MarkSize size = GetParam();

        //初始化TwoDimensionalMark和用来做对比的contrast,将他们全部用true填充
        TwoDimensionalMark mark(size.row, size.col);
        mark.clean(true);
        vector<vector<bool> > contrast = CreateContrast(size.row, size.col, true);

        //验证mark和contrast里面都是true
        for (int i = 0; i < size.row; i++){
            for (int j = 0; j < size.col; j++){
                ASSERT_TRUE(mark.check(j, i));
                ASSERT_TRUE(contrast[i][j]);
            }
        }

        //随机将一些坐标位置,这个函数生成的随机位置有可能在row，col之外
        vector<Coord> coords = CreateRandCoords(size.row, size.col);

        //将mark和contrast的之前随机生成的位置设为false
        SetFlag(&mark, &contrast, size, coords, false);

        //检测随机生成的位置上的值有没有设置正确
        //mark若位置超出范围则返回false,而contrast则需要防止溢出
        for (auto c : coords){
            ASSERT_FALSE(mark.check(c.x,c.y));
            if (c.x >= 0 && c.x < size.col && c.y >= 0 && c.y < size.row){
                ASSERT_FALSE(contrast[c.y][c.x]);
            }
        }

        //对比contrast和mark，两者应该相等
        for (int y = 0; y < size.row; y++)
            for (int x = 0; x < size.col; x++){
                ASSERT_EQ(contrast[y][x], mark.check(x, y));
            }
    }
```

TEST_P(TestWithMarkSize, testWithFalseClean) 和上面的差不多，只不过换成一开始用 false 填充，之后设置随机下标的数据为 true，读者可以自己查看代码，这里就不详细分析了。

## __测试结果__

运行测试代码得到下面的结果：
{% img /GTest实例解析/1.jpg %}

全部测试均通过！

完整代码:[https://github.com/bluesky466/GTestDemo](https://github.com/bluesky466/GTestDemo)
（为了方便，我直接将实现写在了头文件里，好孩子不要学~）