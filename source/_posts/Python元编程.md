title: Python元编程
date: 2025-08-22 14:53:55
tags:
    - 技术相关
    - Python
---


小组内已经有蛮多的项目是通过python去实现的,但是大部分人可能只是用到了它的一些简单特性,像`元编程`这样的高级特性就没有去了解过。

虽然过渡使用元编程会造成代码复杂度,但在某些特定场景下可能是个比较优雅的解决思路。所以我们可以不去用它，但是我们需要了解他。

# 元编程

[Python元编程](https://python3-cookbook.readthedocs.io/zh-cn/latest/chapters/p09_meta_programming.html)是指在运行时对Python代码进行操作的技术，它可以动态地生成、修改和执行代码，从而实现一些高级的编程技巧,这里介绍喜爱比较常用的技术几种技术:装饰器、魔法函数、元类。

# 装饰器

装饰器从形式看类似java里面的注解,但它的原理是基于"python里面函数是一等公民"或者说"python里面一切皆对象,包括函数"。我们先来看一个例子:

```python
import json
USERS = {
    "tom": {"permissions": ["read", "write"]},
    "jerry": {"permissions": ["read"]},
    "mary": {"permissions": []},
}

DB = {}

class Request:
    def __init__(self, headers, body = None):
        self.headers = headers
        self.body = body

def on_get_request(request):
    body = json.loads(request.body)
    return 200, DB.get(body.get("key"), "null")

def on_dump_request(request):
    return 200, json.dumps(DB)

def on_list_request(request):
    return 200, str(DB.keys())

def on_set_request(request):
    body = json.loads(request.body)
    DB[body.get("key")] = body.get("value")
    return 200, "OK"

def on_delete_request(request):
    body = json.loads(request.body)
    del DB[body.get("key")]
    return 200, "OK"
```

如果我们想要在`on_xxx_request`里面将请求的header和body还有response都打印出来,最简单直接的方法是类似下面这种方式修改每个函数:


```python
def on_get_request(request):
    print("on_get_request")
    print(f"\trequest : headers({request.headers}) body({request.body})")
    body = json.loads(request.body)
    (response_code, response_body) = 200, DB.get(body.get("key"), "null")
    print(f"\tresponse : code({response_code}) body({response_body})")
    return response_code, response_body
```

但是这种方式重复代码过高,一旦有日志输出方式或者格式需要修改的情况下改动会很多。我们可以先将打印的操作统一到一个函数中:

```python
def log_request(func):
    def wrapper(request):
        print(func.__name__)
        print(f"\trequest : headers({request.headers}) body({request.body})")
        (response_code, response_body) = func(request)
        print(f"\tresponse : code({response_code}) body({response_body})")
        return response_code, response_body
    return wrapper
```

我们可以将函数当成一个普通的变量,这就意味着我们可以这样操作:

```python
def on_get_request(request):
    ...

def on_dump_request(request):
    ...

def on_list_request(request):
    ...

def on_set_request(request):
    ...

def on_delete_request(request):
    ...

on_get_request = log_request(on_get_request)
on_dump_request = log_request(on_dump_request)
on_list_request = log_request(on_list_request)
on_set_request = log_request(on_set_request)
on_delete_request = log_request(on_delete_request)
```

这样一来,后面调用的`on_xxx_request`函数实际上是`log_request.wrapper`这个嵌套函数,通过闭包的特性保存了原始的函数,然后在调用原始函数前后可以添加一些打印。

装饰器的原理就是如此,上面的代码等同于:

```python
@on_get_request
def on_get_request(request):
    ...

@on_get_request
def on_dump_request(request):
    ...

@on_get_request
def on_list_request(request):
    ...

@on_get_request
def on_set_request(request):
    ...

@on_get_request
def on_delete_request(request):
    ...
```

#### 带参装饰器

如果我们需要让下面的单元测试通过,需要怎么修改python代码:

```python
for user, properties in USERS.items():
    request = Request({"username": user}, json.dumps({"key": "KEY", "value": "VALUE"}))
    read_response_code = 200 if "read" in properties["permissions"] else 403
    write_response_code = 200 if "write" in properties["permissions"] else 403
    
    status, body = on_get_request(request)
    assert status == read_response_code
    
    status, body = on_dump_request(request)
    assert status == read_response_code

    status, body = on_list_request(request)
    assert status == read_response_code

    status, body = on_set_request(request)
    assert status == write_response_code

    status, body = on_delete_request(request)
    assert status == write_response_code
```

还是从嵌套函数出发,我们可以写出这样的多重嵌套函数:

```Python
def check_permission(permission):
    def wrapper(func):
        def wrapper(request):
            permissions = USERS.get(request.headers.get("username"), {}).get("permissions", [])
            if permission in permissions:
                return func(request)
            return 403, "Forbidden"
        return wrapper
    return wrapper

on_get_request = check_permission("read")(on_get_request)
on_dump_request = check_permission("read")(on_dump_request)
on_list_request = check_permission("read")(on_list_request)
on_set_request = check_permission("write")(on_set_request)
on_delete_request = check_permission("write")(on_delete_request)
```

上面的代码就是带参装饰器的原理,函数复制修改的部分等同于下面的代码:

```python
@check_permission("read")
def on_get_request(request):
    ...

@check_permission("read")
def on_dump_request(request):
    ...

@check_permission("read")
def on_list_request(request):
    ...

@check_permission("write")
def on_set_request(request):
    ...

@check_permission("write")
def on_delete_request(request):
    ...
```

#### 装饰器顺序

如果同时使用两个装饰器的情况下,装饰器的顺序对最终的执行结果也是会有影响的。

```python
@check_permission("read")
@log_request
def on_get_request(request):
    body = json.loads(request.body)
    return 200, DB.get(body.get("key"), "null")
```

例如上面这个代码最终的打印是

```
on_get_request
        request : headers({'username': 'tom'}) body({"key": "KEY", "value": "VALUE"})
        response : code(200) body(null)
on_get_request
        request : headers({'username': 'jerry'}) body({"key": "KEY", "value": "VALUE"})
        response : code(200) body(null)
```

因为`on_get_request`先被`@log_request`替换了,再被`@check_permission`替换。

如果我们将两个装饰器的顺序调换一下:

```python
@log_request
@check_permission("read")
def on_get_request(request):
    body = json.loads(request.body)
    return 200, DB.get(body.get("key"), "null")
```

`on_get_request`会先被`@check_permission`再被替换`@log_request`替换,最终的打印就变成了:

```
wrapper
        request : headers({'username': 'tom'}) body({"key": "KEY", "value": "VALUE"})
        response : code(200) body(null)
wrapper
        request : headers({'username': 'jerry'}) body({"key": "KEY", "value": "VALUE"})
        response : code(200) body(null)
wrapper
        request : headers({'username': 'mary'}) body({"key": "KEY", "value": "VALUE"})
        response : code(403) body(Forbidden)
```


# 魔法函数

在我们的机器里会有一个开放接口提供给第三方应用调用,为了做自动化测试我们会在python里面编写测试脚本去调用接口,下面是demo代码:

```python
import json

CACHE = {}

def send_request_to_device(request):
    print(f"send_request_to_device: {request}")

    # 模拟发送到机器,并且获得了机器上执行结果的返回
    index = request["code"].index("(")
    method = request["code"][:index]
    params = request["code"][index + 1: -1].split(",")
    if method == "SDKAudioHelper.I.setVolume":
        CACHE["volume"] = int(params[0])
        return json.loads('{"value": true}')
    elif method == "SDKAudioHelper.I.getVolume":
        return json.loads('{"value": ' + str(CACHE["volume"]) + '}')

if __name__ == "__main__":
    assert send_request_to_device({"code" : "SDKAudioHelper.I.setVolume(50)"})["value"] 
    assert send_request_to_device({"code" : "SDKAudioHelper.I.getVolume()"})["value"] == 50
```

如果我们想让测试用例的代码尽量贴近开放接口的原始代码,即写成下面的形式要怎么做?

```python
assert SDKAudioHelper.I.setVolume(50) 
assert SDKAudioHelper.I.getVolume() == 50
```

比较粗暴的方式是这样的:

```python
class SDKAudioHelper:
    def setVolume(self, volume):
        return send_request_to_device({"code" : f"SDKAudioHelper.I.setVolume({volume})"})["value"] 

    def getVolume(self):
        return send_request_to_device({"code" : "SDKAudioHelper.I.getVolume()"})["value"]


if __name__ == "__main__":
    SDKAudioHelper.I = SDKAudioHelper()
    assert SDKAudioHelper.I.setVolume(40) 
    assert SDKAudioHelper.I.getVolume() == 40
```

但这样会有个问题就是如果开放接口有100个方法,那么就需要写100个方法,如果开放接口的方法名有变化,那么就需要修改100个方法,所以我们需要一种方式让python能够自动帮我们生成这些方法。

python 中有许多魔法函数（Magic Methods），一般格式为 `__func__`,详细的可以参考[这篇文章](https://pycoders-weekly-chinese.readthedocs.io/en/latest/issue6/a-guide-to-pythons-magic-methods.html):

|魔术方法|调用方式|解释|
|-|-|-|
|\_\_new\_\_(cls [,...])|instance = MyClass(arg1, arg2)|\_\_new\_\_ 在创建实例的时候被调用|
|\_\_init\_\_(self [,...])|instance = MyClass(arg1, arg2)|\_\_init\_\_ 在创建实例的时候被调用|
|\_\_cmp\_\_(self, other)|self == other, self > other, 等。|在比较的时候调用|
|\_\_pos\_\_(self)|+self|一元加运算符|
|\_\_neg\_\_(self)|-self|一元减运算符|
|\_\_invert\_\_(self)|~self|取反运算符|
|\_\_index\_\_(self)|x[self]|对象被作为索引使用的时候|
|\_\_nonzero\_\_(self)|bool(self)|对象的布尔值|
|\_\_getattr\_\_(self, name)|self.name # name 不存在|访问一个不存在的属性时|
|\_\_setattr\_\_(self, name, val)|self.name = val|对一个属性赋值时|
|\_\_delattr\_\_(self, name)|del self.name|删除一个属性时|
|\_\_getattribute(self, name)|self.name|访问任何属性时|
|\_\_getitem\_\_(self, key)|self[key]|使用索引访问元素时|
|\_\_setitem\_\_(self, key, val)|self[key] = val|对某个索引值赋值时|
|\_\_delitem\_\_(self, key)|del self[key]|删除某个索引值时|
|\_\_iter\_\_(self)|for x in self|迭代时|
|\_\_contains\_\_(self, value)|value in self, value not in self|使用 in 操作测试关系时|
|\_\_concat\_\_(self, value)|self + other|连接两个对象时|
|\_\_call\_\_(self [,...])|self(args)|“调用”对象时|
|\_\_enter\_\_(self)|with self as x:|with 语句环境管理|
|\_\_exit\_\_(self, exc, val, trace)|with self as x:|with 语句环境管理|
|\_\_getstate\_\_(self)|pickle.dump(pkl_file, self)|序列化|
|\_\_setstate\_\_(self)|data = pickle.load(pkl_file)|序列化|

还记得我们前面强调的"python里面函数是一等公民"吗?调用SDKAudioHelper的setVolume实际上分了两步:

1. 获取SDKAudioHelper的setVolume这个成员变量,它的类型是函数
2. 调用获取到的函数

所以我们可以利用`__getattr__`返回一个内嵌函数是拼接函数名和参数,实现任意方法调用:

```python
class OpenSdkHelper:
    def join_args(self, *args):
        return ",".join([str(a) for a in args])
    
    def invoke_send_request_to_device(self, method_name, *args):
        code = f"{self.__class__.__name__}.I.{method_name}({self.join_args(*args)})"
        result = send_request_to_device({"code" : code})
        return result["value"]
   
    def __getattr__(self, name):
        return lambda *args : self.invoke_send_request_to_device(name, *args)
    

class SDKAudioHelper(OpenSdkHelper):
    pass
    
if __name__ == "__main__":
    SDKAudioHelper.I = SDKAudioHelper()
    assert SDKAudioHelper.I.setVolume(40) 
    assert SDKAudioHelper.I.getVolume() == 40
```

这里我们抽象出来一个OpenSdkHelper的父类,这样一来其他的Helper只需要继承他就能实现任意方法调用了。

# 元类

上面的代码我们还需要对每个Helper的I变量做初始化:

```python
if __name__ == "__main__":
    SDKAudioHelper.I = SDKAudioHelper()
```

当Helper类型的数量变多的时候每个Helper都需要初始化一下也是比较麻烦的,这一步有没有办法优化呢?

在Python中一切都是对象,包括类。通常我们用类来定义对象，类本身也是一个对象。而负责创建这些类的“类”就是元类。如果类是对象的模板,那么元类就是类的模板。

type是默认元类,默认情况下类是由type去创建的,例如下面的两种方法都可以用来创建MyClass这个类:

```python
# 通过类定义去创建MyClass类
class MyClass:
    pass 

# 通过type去创建MyClass类
MyClass = type('MyClass', (), {})
```
类定义的背后实际上python就是通过type帮我们创建的类,我们也可以通过继承type来创建我们自己的元类:

```python

class OpenSdkHelperMeta(type):
    def __new__(self, name, bases, attrs):
        new_cls = super().__new__(self, name, bases, attrs)
        new_cls.I = new_cls()
        return new_cls
    
class OpenSdkHelper(metaclass = OpenSdkHelperMeta):
    ...

class SDKAudioHelper(OpenSdkHelper):
    pass
```
这样一来OpenSdkHelper和它的子类就都是由OpenSdkHelperMeta这个元类创建的,而OpenSdkHelperMeta在创建类的时候会自动初始化I变量,这样我们就不需要手动初始化了。

python元类的一种重要应用场景就是搭建数据库的ORM框架。