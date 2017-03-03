title: lua元表
date: 2016-10-28 22:45:37
tags:
	- 技术相关
	- lua
---

## 元表是什么？

在lua中,每个值都有一套预定义的操作集合。例如数字可以相加、比较、字符串可以连接,lua将这些操作的定义放在了元表中去描述。lua中的每个值都有一个元表。table和userdata可以有各自独立的元表,而其他类型则共享其类型所属的统一元表。(书上是这么说的，但是我用getmetatable方法只能获取到字符串的元表)lua在创建table的时候不会为它创建元表,所以,table没有加的操作,我们就可以通过给table设置我们自己写的元表,为table定义一套自定义的加的操作。


## 如何获取元表?

lua中通过getmetatable获取值得元表:

```lua
print("str 1 : "..tostring( getmetatable("str1")) )
print("str 2 : "..tostring( getmetatable("str1")) )
print("nil : "..tostring( getmetatable(nil)) )
print("number : "..tostring( getmetatable(1)) )
print("function : "..tostring( getmetatable(function()end)) )
print("table : "..tostring( getmetatable({})) )
```

>str 1 : table: 009D9798
>str 2 : table: 009D9798
>nil : nilw
>number : nil
>function : nil
>table : nil

可以看到不同的字符串用的是同一个元表,而除了字符串之外其他的值的元表都是nil。这里就是我看到和书上不一样的地方，书上说每个值都有一个元表。不过这只是一个小疑点，并不影响我们对元表的理解。

我们看看string的原表到底是个什么东西:

```lua
function dump(t)
	if t==nil then
		print(t)
	else
		print("{")
		for k,v in pairs(t) do
			print("\t"..tostring(k).." = "..tostring(v))
		end
		print("}")
	end
end
dump(getmetatable("str1"))
```

输出如下:
>{
>        \__index = table: 00AF9270
>}

字符串的元表里面只有一个元素:\__index,它也是一个table,我们继续跟踪下看看它到底是什么:
```lua
function dump(t)
	if t==nil then
		print(t)
	else
		print("{")
		for k,v in pairs(t) do
			print("\t"..tostring(k).." = "..tostring(v))
		end
		print("}")
	end
end
dump(getmetatable("str1").__index)
```

输出如下:
>{
>        sub = function: 00ABABF8
>        upper = function: 00ABACB8
>        len = function: 00AB9D00
>        gfind = function: 00AB9CA0
>        rep = function: 00ABAD58
>        find = function: 00AB9E20
>        match = function: 00ABA9F8
>        char = function: 00AB9C40
>        dump = function: 00AB9F20
>        gmatch = function: 00AB9CA0
>        reverse = function: 00ABAC38
>        byte = function: 00AB9CC0
>        format = function: 00AB9C80
>        gsub = function: 00AB9CE0
>        lower = function: 00AB9D40
>}

\__index这个table定义了字符串的一些基本操作,如获取长度,查找子串等。它们的其实就定义在string这个table里,也就是说所有的字符串的元表的\__index都是string:

```lua
function dump(t)
	if t==nil then
		print(t)
	else
		print("{")
		for k,v in pairs(t) do
			print("\t"..tostring(k).." = "..tostring(v))
		end
		print("}")
	end
end

print(getmetatable("str").__index)
print(string)
```
>table: 00C89270
>table: 00C89270


## 如何设置元表?

在lua中,我们可以用setmetatable(table, metatable)这个方法去设置table的元表:

```lua
local t = {}
print(getmetatable(t))
setmetatable(t,{})
print(getmetatable(t))
```

输出如下:

>nil
>table: 0x7feaf1407190

这样就成功为t设置了一个元表,这个元表是一个空的table。

## 元表的作用

那设置元表又有什么用呢？还记得元表的作用是什么吗？对定义对值得操作,例如下面的代码,我们为table设置了一个tostring的操作:
```lua
local t = {x=1,y=2}
print(tostring(t))

local mt = {}
mt.__tostring = function(t)return "("..t.x..","..t.y..")" end
setmetatable(t,mt)
print(tostring(t))
```

>table: 00B49678
>(1,2)

类似的在元表中我们可以定义下面的这些方法:

|方法|作用|
|--|--|
|\__add|加法|
|\__sub|减法|
|\__mul|乘法|
|\__div|除法|
|\__unm|相反数|
|\__mod|取模|
|\__pow|乘幂|
|\__concat|连接操作|
|\__eq|等于|
|\__lt|小于|
|\__le|小于等于|
|\__tostring|转化为字符串|
|\__index|读取不存在的字段|
|\__newindex|设置不存在的字段|

注意lua会把a~=b转化为not(a==b),将a>b转化为b<a,将a>=b转化为b<=a,所以元表中并没有表示这几种操作的字段

## 元表与类机制

如果我们每创建一个table都要这样手动的为它创建一个元表,其实是很麻烦的一件事情,所以我们可以用下面的方法去简化操作:

```lua
local MyTable = {__tostring=function(t)return t.x..","..t.y end}

function MyTable.new(t)
	if t==nil then
		t = {}
	end
	setmetatable(t, MyTable)
	return t
end

print(MyTable.new{x=1,y=2})
```

输出如下:

>1,2


看，这是不是有点像类？还记得\__index吗？它用来定义访问table中没有的字段的时候的操作:

```lua
local mt = {}
mt.__index = function(t,k)return "no key ["..k.."] in table" end

local t = {x=1,y=2}
setmetatable(t,mt)
print(t.x)
print(t.y)
print(t.z)
```

输出如下:

>1
>2
>no key [z] in table

特殊的,如果\__index是一个table的时候,在访问没有的字段的时候lua解释器就会到元表的\__index中去找:

```lua
local t = {}
print(t.x)

local index = {x=123}
setmetatable(t,{__index=index})
print(t.x)

t.x = 321
print(t.x)
print(getmetatable(t).__index.x)
```

它的输出是这样的:
>nil
>123
>321
>123

当在table中找不到字段时,解释器会去元表的\__index字段中去找,如果\__index中可以找到的话就用\__index中的字段。如果table中有该字段的话,解释器就不会再去查询元表。嗯，听起来是不是有点像继承和重写?事实上lua的继承机制也是利用元表的这种特性实现的。
