title: lua泛型for的原理
date: 2016-10-23 23:10:47
tags:
	- 技术相关
	- lua
---
学习lua的时候,一直觉得泛型for是个很有用的东西,也觉得它很神奇,但因为它是语法层面就支持的东西,所以就没有去深入思考其中的原理。直到最近看《Lua程序设计》才知道它底层的工作原理原来那么巧妙。

## 泛型for原理

泛型for的用法如下:
```lua
local person = {name="Jim", age=18}
for k,v in pairs(person) do
	print(k,v)
end 
```

输出结果如下:
>name    Jim
>age     18

它究竟是怎么做到通过循环,把table中的key和value通通打印出来的呢？

其实泛型for语句:

```lua
for var_1, ..., var_n in <explist> do <block> end
```

等价以下的代码:

```lua
do 
    local _f,_s,_var = <explist>
    while true do
        local var_1, ..., var_n = _f(_s, _var)
        _var = var_1
        if _var==nil then break end
        <block>
    end
end
```

## pairs函数原理

这里要先介绍一个基本函数next。如果k是table t的一个key,在调用next(t,k)的时候,会返回t的下一个key和对应的值,如果key为nil,则返回t的第一组key和value,如果没有下一组key和value则返回nil。

其实pairs的定义很简单:

```lua
function pairs(t)
    return next, t, nil
end
```

它只是简单的返回了next函数和原来的table,所以泛型for又能这么写:

```lua
local person = {name="Jim", age=18}
for k,v in next, person do
    print(k,v)
end 
```
等价于:

```lua
local person = {name="Jim", age=18}
do
	local _f,_s,_var = next, person
	while true do
		k,v = _f(_s, _var)
		_var = k
		if _var==nil then break end
		print(k,v)
	end
end
```

## ipairs函数原理

ipairs函数比pairs函数要复杂一点。如果按照pair的做法,会出现以下情况:

```lua
local test1 = {"one", "two", three=3}

for k,v in next, test1 do
	print(k,v)
end

print("================")

for i,v in ipairs(test1) do
	print(i,v)
end
```

输出如下:

>1       one
>2       two
>three   3
>\================
>1       one
>2       two

还会出现下面的这种情况:

```lua
local test1 = {"one", nil, "three"}

for k,v in next, test1 do
	print(k,v)
end

print("================")

for i,v in ipairs(test1) do
	print(i,v)
end
```

输出:
>1       one
>3       three
>\================
>1       one

我们在使用ipairs的时候是想以数组的方式遍历table,但pairs会把table中的所有键值对都遍历一遍。使用ipairs的时候会从下标为1开始逐一遍历table,直到遇到value等于nil的时候停止,它的工作原理如下:

```lua
function iter(t, index)
	index = index + 1
	local var = t[index]
	if var then
		return index, var
	end
end

function ipairs(t)
	return iter,t,0
end
```

或者简化成下面的形式:

```lua
local test1 = {"one", "two", "three"}

function iter(t, index)
	index = index + 1
	local var = t[index]
	if var then
		return index, var
	end
end

for i,v in iter,test1,0 do
	print(i,v)
end
```