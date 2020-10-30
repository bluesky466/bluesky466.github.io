title: python与c/c++相互调用
date: 2020-10-30 23:52:12
tags:
    - 技术相关
    - python
    - C/C++
---

最近的项目使用python语言，其中一个功能需要对接c++的sdk。于是学习了下python与c/c++的相互调用方法，这里做下笔记，方便以后查找。

python里面调用c/c++代码基本上有三种方式: ctypes库、cffi库和c/c++拓展模块。这篇笔记主要讲的是拓展模块，不过ctypes和cffi也会稍微介绍一下:

# ctypes

使用ctypes模块十分简单，这里直接上demo。我们的c代码如下:

```c
// demo.c
int add(int a, int b) {
    return a + b;
}
```

使用下面命令编译demo.so

> gcc demo.c -shared -fPIC -o demo.so

然后python里面只需要用ctypes.cdll.LoadLibrary方法加载so库，就可以通过方法名去调用c的函数了:

```python
import ctypes
lib = ctypes.cdll.LoadLibrary("./demo.so")
print(lib.add(1, 3))
```

基本上不需要过多的介绍，不过有个坑是如果写的是c++，那么需要用extern "C"包裹下给python调用的函数:

```c++
class Utils {
public:
        static int add(int a, int b) {
                return a + b;
        }
};

extern "C" {

int add(int a, int b) {
        return Utils::add(a, b);
}

}
```

这么做的原因在于c++编译之后会修改函数的名字，add函数在编译之后变成了__ZN5Utils3addEii，而且不同编译器的修改规则还不一样，所以在python里面用add找不到对应的函数。

加上extern "C"包裹之后能让编译器按照c的方式去编译这个函数，不对函数名做额外的修改，这样python里面才能通过函数名去调用它。

# cffi

cffi和ctypes类似，但是稍微复杂一些。

cffi的功能其实是在python里面写c代码，我们可以通过python里面写的c代码去调用第三库的c代码。

这么说可能有点抽象，我举个例子大家可能就好理解了:

我们在c里面实现了一个foo方法，它的作用是打印传入的字符串，并且返回字符串的长度:

```c
#include <stdio.h>
#include <string.h>

int foo(char* str) {
    printf("%s\n", str);
    return strlen(str);
}
```

我们将上面的c代码编译成ffidemo.so，然后用下面的python代码去调用这个foo方法:

```python
from cffi import FFI

ffi = FFI()
lib = ffi.dlopen("./ffidemo.so")           # 导入so
ffi.cdef("int foo(char* str);")            # 声明foo方法
param = ffi.new("char[]", b"hello world!") # 创建char数组
print(lib.foo(param))                      # 调用之前声明的foo方法,它的实现在ffidemo.so
```



# c/c++拓展模块

使用ctypes的方式虽然简便，但是在使用上能明显的感觉出来是在调用so库的代码。

更不用说使用cffi会在python代码里面嵌入c的语句，总有种莫名的不协调感。

而且c/c++编码规范里一般方法名会用驼峰，但是python编码规范里建议方法名用下划线分割单词，上面的两种方法都会造成python里面调用so和python脚本的方法有两种命名规范，逼死强迫症。

有没有一种方法可能让python无感调用c/c++代码，就像调用普通的python代码一样呢？

答案就是使用c/c++为Python编写扩展模块。虽然有[官方文档](https://docs.python.org/zh-cn/3/extending/extending.html)可以参考，但这个文档其实讲的不是很全，当初也遇到了不少问题，这里也整理下。

我们希望Python里面像这样去调用c/c++:

```python
import demo
demo.foo()
```

c/c++的完整代码如下:

```c++
#define PY_SSIZE_T_CLEAN
#include <Python.h>
#include <iostream>
#include <string>

using namespace std;

PyObject* Foo(PyObject* self, PyObject* args) {
    cout<<"Foo"<<endl;
    return Py_BuildValue("");
}

static PyMethodDef g_moduleMethods[] = {
        {"foo", Foo, METH_NOARGS, "function Foo"},
        {NULL, NULL, 0, NULL}
};

static PyModuleDef g_moduleDef = {
        PyModuleDef_HEAD_INIT,
        "ExtendedDemo",                /* name of module */
        "C/C++ Python extension demo", /* module documentation, may be NULL */
        -1,                            /* size of per-interpreter state of the module, or -1 
                                          if the module keeps state in global variables. */
        g_moduleMethods
};

PyMODINIT_FUNC PyInit_demo(void) {
        return PyModule_Create(&g_moduleDef);
}

```

我们用下面命令将这个代码编译成demo.so (mac系统下):

```shell
g++ demo.cpp -shared -fPIC -o demo.so -I /usr/local/Frameworks/Python.framework/Versions/3.7/include/python3.7m -L /usr/local/Frameworks/Python.framework/Versions/3.7/lib -lpython3.7m
```

python的import demo语句就会去动态链接这个demo.so，并且调用PyInit_demo方法。也就是说**so的名字要和PyInit_XXX这个方法名对应**，要不然python里面会报找不到init方法的异常。

这个init方法很简单，就是创建了一个module。这个module的定义在g\_moduleDef这个全局变量里面，它定义了module的name、documentation等，这里的name可以和so的名字不一样，它在python里module的\_\_name\_\_、\_\_doc\_\_里面体现:

```python
import demo
print(demo.__name__)  # ExtendedDemo
print(demo.__doc__)   # C/C++ Python extension demo
```

g\_moduleDef里面最重要的是最后一个成员g\_moduleMethods，它定义的module里面的方法。这货是个PyMethodDef结构体数组，定义了方法名字，方法的指针，参数类型，和文档描述：

```c++
struct PyMethodDef {
    const char  *ml_name;   /* The name of the built-in function/method */
    PyCFunction ml_meth;    /* The C function that implements it */
    int         ml_flags;   /* Combination of METH_xxx flags, which mostly
                               describe the args expected by the C func */
    const char  *ml_doc;    /* The __doc__ attribute, or NULL */
};
typedef struct PyMethodDef PyMethodDef;
```

ml\_name、ml\_meth和ml\_doc都很好理解，ml\_flags有点小坑。它可以是下面几种类型:

- METH\_NOARGS 没有参数
- METH_VARARGS 可变参数
- METH_VARARGS | METH\_KEYWORDS 可变参数+关键字参数

## METH\_NOARGS

我们上面的demo里foo方法就是没有参数的，这里可能有同学会说怎么就没有参数了？明明它有两个参数:

```C++
PyObject* Foo(PyObject* self, PyObject* args) {
    cout<<"Foo"<<endl;
    return Py_BuildValue("");
}
```

是的，虽然在c/c++这里的声明它是有两个参数的，但是由于我们在g\_moduleMethods里面给它的声明是METH\_NOARGS，在python里面如果给它传参就会出现异常:

```python
import demo
demo.foo(1)

# 出现异常
# Traceback (most recent call last):
#   File "test.py", line 2, in <module>
#     demo.foo(1)
#  TypeError: foo() takes no arguments (1 given)
```

所以对于METH\_NOARGS类型的方法来说，c/c++里面的args参数其实是没有意义的，它总是NULL。

而 *self* 参数，对模块级函数指向模块对象，对于对象实例则指向方法。


## METH_VARARGS

当我们将一个方法声明成METH_VARARGS，这个函数的args就会变成一个元组，我们可以通过PyArg\_Parse方法解析出里面的值，例如下面的add方法:

```c++
PyObject* Add(PyObject* self, PyObject* args) {
    int a,b;
    PyArg_Parse(args, "(ii)", &a, &b);
    return Py_BuildValue("i", a+b);
}

static PyMethodDef g_moduleMethods[] = {
        ...
        {"add", Add, METH_VARARGS, "function Add"},
        ...
}
```

这个方法接收两个int的参数，然后返回a+b的值:

```python
import demo
print(demo.add(1,2)) # 3
```

我们看到PyArg\_Parse和Py\_BuildValue都有个字符串去配置数据类型，它们很相似，只不过一个是解析PyObejct\*一个是生成PyObejct\*，这里用Py\_BuildValue举例(左侧是调用，右侧是Python值结果):

```c++
Py_BuildValue("")                        None
Py_BuildValue("i", 123)                  123
Py_BuildValue("iii", 123, 456, 789)      (123, 456, 789)
Py_BuildValue("s", "hello")              'hello'
Py_BuildValue("y", "hello")              b'hello'
Py_BuildValue("ss", "hello", "world")    ('hello', 'world')
Py_BuildValue("s#", "hello", 4)          'hell'
Py_BuildValue("y#", "hello", 4)          b'hell'
Py_BuildValue("()")                      ()
Py_BuildValue("(i)", 123)                (123,)
Py_BuildValue("(ii)", 123, 456)          (123, 456)
Py_BuildValue("(i,i)", 123, 456)         (123, 456)
Py_BuildValue("[i,i]", 123, 456)         [123, 456]
Py_BuildValue("{s:i,s:i}",
              "abc", 123, "def", 456)    {'abc': 123, 'def': 456}
Py_BuildValue("((ii)(ii)) (ii)",
              1, 2, 3, 4, 5, 6)          (((1, 2), (3, 4)), (5, 6))
```

不过需要注意的是，**虽然Py\_BuildValue不用加括号也能自动解析成元组，但是如果要用PyArg\_Parse解析元组的话必须加上括号**,当然你也可以直接用PyArg\_ParseTuple去元组，这样的话就不需要带括号。

## METH\_KEYWORDS

关于METH\_KEYWORDS，文档里面有这样一句话(好像漏了METH\_NOARGS，我测试验证这个也是可以用的):

> 这个标志指定会使用C的调用惯例。可选值有 `METH_VARARGS` 、 `METH_VARARGS | METH_KEYWORDS` 

也就是说METH\_KEYWORDS是不能单独使用的，必须要和METH\_VARARGS一起。我一开始没有注意，单独使用之后一直报错。

这样配置的方法参数类似python里面的func(*args, **kwargs)，而c/c++里面的函数声明和METH\_NOARGS、METH\_VARARGS不一样，有三个参数。可以看下下面的demo:

```c++
PyObject* Subtract(PyObject* self, PyObject* args, PyObject* keywds) {
    int a,b;
    char *kwlist[] = {"a", "b", NULL};
    PyArg_ParseTupleAndKeywords(args, keywds, "ii", kwlist, &a, &b);
    return Py_BuildValue("i", a-b);
}
static PyMethodDef g_moduleMethods[] = {
        ...
        {"subtract", (PyCFunction)(void(*)(void))Subtract, METH_VARARGS|METH_KEYWORDS, "function Subtract"},
        ...
};
```

python里面就能用可变参数和关键字参数的方式传参:

```python
import demo
print(demo.subtract(1, 2))     # -1
print(demo.subtract(1, b=2))   # -1
print(demo.subtract(b=1, a=2)) # 1
```

## c/c++回调python

通过上面的讲解我们可以轻松实现python对c/c++函数的调用。但是我们的项目还出现了python往c/c++里面设置回调函数的需求，我们接下来就来看看这个需求要怎么实现。

下面是c++部分的代码，它注册了个方法，参数是一个PyObject\*，实际上它是个回调函数，我把可以用PyEval\_CallObject去调用它计算两个字符串的字符总数，得到一个PyObject\*的返回值。我们可以用PyLong\_AsLong将它解析成c的long类型:

```c++
PyObject* SetCountCallback(PyObject* self, PyObject* args) {
    PyObject* callback;
    PyArg_Parse(args, "(O)", &callback);

    PyGILState_STATE state = PyGILState_Ensure();
    PyObject* callbackArgs = Py_BuildValue("(ss)", "hello ", "world");
    PyObject* result = PyEval_CallObject(callback, callbackArgs);
    cout<<"count result : "<<PyLong_AsLong(result)<<endl;

    Py_DECREF(callbackArgs);
    PyGILState_Release(state);

    return Py_BuildValue("");
}

static PyMethodDef g_moduleMethods[] = {
        ...
        {"setCountCallback", SetCountCallback, METH_VARARGS, "function SetCountCallback"},
        ...
};

```

python的代码如下:

```python
import demo
def count(str1, str2):
    return len(str1) + len(str2)
demo.setCountCallback(count) # c++会打印count result : 11
```

类似的我们可以用PyFloat\_AsDouble从PyObject\*解析出double类型的数据。但是如果是字符串类型的话解析比较麻烦需要先转换成bytes类型的数据再转成char\*，可以用下面这个方法转换:

```c++
string GetStringFromPyObject(PyObject* pObject) {
    PyObject* bytes = PyUnicode_AsUTF8String(pObject);
    string str = PyBytes_AsString(bytes);
    Py_DECREF(bytes);
    return str;
}
```

如果是返回的元组的话可以用遍历的方法去读取，用PyTuple\_Size读取数量然后用PyTuple\_GetItem读取item，然后再用上面的转换方法转换:

```c++
PyObject* SetSplicingAndCountCallback(PyObject* self, PyObject* args) {
    PyObject* callback;
    PyArg_Parse(args, "(O)", &callback);

    PyGILState_STATE state = PyGILState_Ensure();
    PyObject* callbackArgs = Py_BuildValue("(ss)", "hello ", "world");
    PyObject* result = PyEval_CallObject(callback, callbackArgs);
    cout<<"result size : "<<PyTuple_Size(result)<<endl;
    cout<<"item0 : "<<GetStringFromPyObject(PyTuple_GetItem(result, 0))<<endl;
    cout<<"item1 : "<<PyLong_AsLong(PyTuple_GetItem(result, 1))<<endl;
    Py_DECREF(callbackArgs);
    PyGILState_Release(state);

    return Py_BuildValue("");
}
```

Python里面这么调用:

```python
import demo

def splicing_and_count(str1, str2):
    return str1+str2, len(str1)+len(str2)
    
demo.setSplicingAndCountCallback(splicing_and_count)
```

不过实际调试的时候使用PyArg\_Parse也能解析出返回值元组的数据，但是这个方法的名字用在解析返回值这里总感觉怪怪的，说不好有什么坑，这块文档里面也没有讲。

```c
char* s;
int i;
PyArg_Parse(result, "(si)", &s, &i);
```

# 完整demo

```c++
#define PY_SSIZE_T_CLEAN
#include <Python.h>
#include <iostream>
#include <string>

using namespace std;

PyObject* Foo(PyObject* self, PyObject* args) {
    cout<<self<<endl;
    cout<<args<<endl;
    cout<<"Foo"<<endl;
    return Py_BuildValue("");
}

PyObject* Add(PyObject* self, PyObject* args) {
    int a,b;
    PyArg_Parse(args, "(ii)", &a, &b);
    return Py_BuildValue("i", a+b);
}

PyObject* Subtract(PyObject* self, PyObject* args, PyObject* keywds) {
    int a,b;
    char *kwlist[] = {"a", "b", NULL};
    PyArg_ParseTupleAndKeywords(args, keywds, "ii", kwlist, &a, &b);
    return Py_BuildValue("i", a-b);
}

PyObject* SetCountCallback(PyObject* self, PyObject* args) {
    PyObject* callback;
    PyArg_Parse(args, "(O)", &callback);

    PyGILState_STATE state = PyGILState_Ensure();
    PyObject* callbackArgs = Py_BuildValue("(ss)", "hello ", "world");
    PyObject* result = PyEval_CallObject(callback, callbackArgs);
    cout<<"count result : "<<PyLong_AsLong(result)<<endl;

    Py_DECREF(callbackArgs);
    PyGILState_Release(state);

    return Py_BuildValue("");
}

string GetStringFromPyObject(PyObject* pObject) {
    PyObject* bytes = PyUnicode_AsUTF8String(pObject);
    string str = PyBytes_AsString(bytes);
    Py_DECREF(bytes);
    return str;
}

PyObject* SetSplicingCallback(PyObject* self, PyObject* args) {
    PyObject* callback;
    PyArg_Parse(args, "(O)", &callback);

    PyGILState_STATE state = PyGILState_Ensure();
    PyObject* callbackArgs = Py_BuildValue("(ss)", "hello ", "world");
    PyObject* result = PyEval_CallObject(callback, callbackArgs);
    cout<<"splicing result : "<<GetStringFromPyObject(result)<<endl;

    Py_DECREF(callbackArgs);
    PyGILState_Release(state);

    return Py_BuildValue("");
}

PyObject* SetSplicingAndCountCallback(PyObject* self, PyObject* args) {
    PyObject* callback;
    PyArg_Parse(args, "(O)", &callback);

    PyGILState_STATE state = PyGILState_Ensure();
    PyObject* callbackArgs = Py_BuildValue("(ss)", "hello ", "world");
    PyObject* result = PyEval_CallObject(callback, callbackArgs);
    cout<<"result size : "<<PyTuple_Size(result)<<endl;
    cout<<"item0 : "<<GetStringFromPyObject(PyTuple_GetItem(result, 0))<<endl;
    cout<<"item1 : "<<PyLong_AsLong(PyTuple_GetItem(result, 1))<<endl;

    char* s;
    int i;
    PyArg_Parse(result, "(si)", &s, &i);
    cout<<"s="<<s<<endl;
    cout<<"i="<<i<<endl;

    Py_DECREF(callbackArgs);
    PyGILState_Release(state);

    return Py_BuildValue("");
}

static PyMethodDef g_moduleMethods[] = {
        {"foo", Foo, METH_NOARGS, "function Foo"},
        {"add", Add, METH_VARARGS, "function Add"},
        {"subtract", (PyCFunction)(void(*)(void))Subtract, METH_VARARGS|METH_KEYWORDS, "function Subtract"},
        {"setCountCallback", SetCountCallback, METH_VARARGS, "function SetCountCallback"},
        {"setSplicingCallback", SetSplicingCallback, METH_VARARGS, "function SetSplicingCallback"},
        {"setSplicingAndCountCallback", SetSplicingAndCountCallback, METH_VARARGS, "function SetSplicingAndCountCallback"},
        {NULL, NULL, 0, NULL}
};

static PyModuleDef g_moduleDef = {
        PyModuleDef_HEAD_INIT,
        "ExtendedDemo",
        "C/C++ Python extension demo",
        -1,
        g_moduleMethods
};

PyMODINIT_FUNC PyInit_demo(void) {
        return PyModule_Create(&g_moduleDef);
}
```

```python
import demo

print(demo.__name__)
print(demo.__doc__)


print(demo.add.__name__)
print(demo.add.__doc__)

def splicing(str1, str2):
    return str1+str2

def count(str1, str2):
    return len(str1) + len(str2)

def splicing_and_count(str1, str2):
    return splicing(str1, str2), count(str1, str2)

demo.foo()
print(demo.add(1,2))
print(demo.subtract(1, 2))
print(demo.subtract(1, b=2))
print(demo.subtract(b=1, a=2))

demo.setCountCallback(count)
demo.setSplicingCallback(splicing)
demo.setSplicingAndCountCallback(splicing_and_count)

def foo(a,b):
    return a-b
```