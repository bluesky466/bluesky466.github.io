title: lua面向对象编程
date: 2016-11-16 00:43:45
tags:
	- 技术相关
	- lua
---

lua也支持面向对象编程的,嗯对,就是用table和元表。lua可以在某种程度上实现面向对象的封装、继承和多态三大基本特性。

## 封装

lua实现封装的最简单方法就是将属性和方法放到table之中。首先我们声明一个类table,用来定义一些该类的类方法，其中的new方法就类似其他面向对象语言的new关键字,用来创建一个新的实例出来。同时为了模块化，将它放在一个单独的Position.lua文件中:

```lua
local Position = {}

function Position.new(x,y)
	local instance = {x=x, y=y}
	return instance
end

function Position.getLength(instance)
	return (instance.x^2+instance.y^2)^0.5
end

return Position
```

这样就可以这样使用它:
```lua
local Position = require("Position")
local pos1 = Position.new(1,1)
print(Position.getLength(pos1))
```

输出如下:
>1.4142135623731

看看这个方法的定义:

```lua
function Position.getLength(instance)
	return (instance.x^2+instance.y^2)^0.5
end
```

它其实是一种语法糖,上面的写法和下面的写法是一样的

```lua
Position.getLength = function(instance)
	return (instance.x^2+instance.y^2)^0.5
end
```

但是这样的写法是不是感觉很奇怪,和一般的面向对象的写法和用法都不一样？我们可以把Position.lua改成下面的这个样子:

```lua
local Position = {}

function Position.new(x,y)
	local instance = {x=x, y=y}
	setmetatable(instance, {__index=Position})
	return instance
end

function Position.getLength(instance)
	return (instance.x^2+instance.y^2)^0.5
end

return Position
```

于是用法就有点接近我们熟悉的面向对象语法了

```lua
local Position = require("Position")

local pos1 = Position.new(1,1)
print(pos1.getLength(pos1))

local pos2 = Position.new(3,4)
print(pos2.getLength(pos2))
```

输出如下:

>1.4142135623731
>5

但是每次调用类方法都需要把实例显示的当作参数传入,这样既麻烦又容易出错。有没有办法让lua解释器自动传入类实例呢？答案当然是有的,这就要看另一种语法糖了:

```lua
local Position = require("Position")

local pos1 = Position.new(1,1)
print(pos1:getLength())

local pos2 = Position.new(3,4)
print(pos2:getLength())
```

对的，下面的两种写法是等价的:

```lua
pos1.getLength(pos1)
pos1:getLength()
```

类似的,在Position.lua中也能这么写：

```lua
local Position = {}

function Position.new(x,y)
	local instance = {x=x, y=y}
	setmetatable(instance, {__index=Position})
	return instance
end

function Position:getLength()
	return (self.x^2+self.y^2)^0.5
end

return Position
```

下面两种写法也是等价的:
```lua
function Position.getLength(self)
	return (self.x^2+self.y^2)^0.5
end

function Position:getLength()
	return (self.x^2+self.y^2)^0.5
end
```

这里的self就相当于c++的this

我们还可以把new方法抽象出来,作为公共方法,而不用为每个类都写一个new方法:

```lua
-- function.lua

function class(className)
	local cls = {}
	cls.__cname = className
	cls.new = function(...)
		local instance = {}
		setmetatable(instance, {__index=cls})
		if cls.ctor then cls:ctor(...) end
		return instance
	end
	return cls
end
```

这个new方法将类名保存了下来,方便运行时获取类的类型,同时如果这个类有定义构造函数(ctor)的话,它还会自动的调用类的构造函数。

这个时候类的定义就可以变成下面这个样子了:

```lua
-- Position.lua

local Position = class("Position")

function Position:ctor(x,y)
	self.x = x
	self.y = y
end

function Position:getLength()
	return (self.x^2+self.y^2)^0.5
end

return Position
```

已经和一般的面向对象语言很接近了。当然,这个class方法我们必须在程序的一开始就加载进来作为全局方法使用。我们来看看main.lua是怎么使用它们的吧:

```lua
require("function")

local Position = require("Position")
print(Position.__cname)

local pos1 = Position.new(1,1)
print(pos1:getLength())

local pos2 = Position.new(3,4)
print(pos2:getLength())
```

输出如下:
>Position
>1.4142135623731
>5



## 继承与多态

lua的继承机制在上一篇博客《lua元表》中已经有提到了一些,原理就是使用元表机制,将子类元表的\__index字段设置为父类。所以我们可以这样拓展我们的new方法:

```lua
-- function.lua

function class(className, super)
	local cls = {}

	if super then 
		cls.super = super
		setmetatable(cls, {__index=super})
	end
	
	cls.__cname = className

	cls.new = function(...)
		local instance = {}
		setmetatable(instance, {__index=cls})
		if cls.ctor then cls:ctor(...) end
		return instance
	end
	
	return cls
end
```
这里我们将父类保存到子类的元表的\__index字段中,同时为类添加了super字段用于保存父类

于是基于new方法,我们可以定义Position的子类Position3D:

```lua
-- Position3D.lua

local Position3D = class("Position3D", require("Position"))

function Position3D:ctor(x,y,z)
	self.super:ctor(x, y)
	self.z = z
end

function Position3D:getLength()
	return (self.x^2+self.y^2+self.z^2)^0.5
end

return Position3D
```

子类Position3D重写了Position的ctor方法和getLength方法。如果需要用父类的被重写的方法,就要用super字段显示调用,就如ctor方法中做的一样。

来看看main.lua吧：

```lua
require("function")

local Position3D = require("Position3D")
print(Position3D.__cname)

local pos1 = Position3D.new(1,1,1)
print(pos1:getLength())

local pos2 = Position3D.new(3,4,5)
print(pos2:getLength())
```

输出如下:
>Position3D
>1.7320508075689
>7.0710678118655


## 多继承

lua同样可以实现多继承。由于一个多继承的子类有多个父类,所以我们不能简单的把父类设为元表的\__index属性。但是我们可以将该类的父类保存在一个table里面,然后用一个函数去搜索父类的方法。这时,只有将这个函数赋值个于元表的\__index就好了。

于是我们可以将class函数拓展成下面的样子

```lua
function class(className, ...)
	local cls = {__cname = className}

    local supers = {...}
	for i,super in ipairs(supers) do
        cls.__supers = cls.__supers or {}
        table.insert(cls.__supers, super)

        if cls.super==nil then
            cls.super=super
        end
	end

    if cls.__supers==nil or  #cls.__supers==1 then
        setmetatable(cls, {__index=cls.super})
	else
		local index = function(t,k)
			for i,v in ipairs(cls.__supers) do
				if v[k] then return v[k] end
			end
		end
        setmetatable(cls, {__index=index})
	end	

	cls.new = function(...)
	    local instance = {}
		setmetatable(instance, {__index=cls})
		if cls.ctor then cls:ctor(...) end
		return instance
	end

	return cls
end
```

之后我们就能这样去使用多继承机制了:

```lua
--ClassA.lua

local ClassA = class("ClassA")

function ClassA:ctor()
end

function ClassA:methodA()
	print("ClassA:methodA")
end

return ClassA
```

```lua
--ClassB.lua

local ClassB = class("ClassB")

function ClassB:ctor()
end

function ClassB:methodB()
	print("ClassB:methodB")
end

return ClassB
```

```lua
--ClassC.lua

local ClassC = class("ClassC", require("ClassA"), require("ClassB"))

function ClassC:ctor()
end

return ClassC
```

```lua
--main.lua

require("function")

local c = require("ClassC").new()
c:methodA()
c:methodB()
```

执行结果如下:

>ClassA:methodA
>ClassB:methodB
