title: 再谈Java泛型
date: 2018-02-04 17:37:01
tags:
	- 技术相关
	- java
---


之前其实已经写过一篇泛型的文章[《java泛型那些事》](http://blog.islinjw.cn/2018/01/06/java%E6%B3%9B%E5%9E%8B%E9%82%A3%E4%BA%9B%E4%BA%8B/),但是最近在看《Kotlin极简教程》泛型部分拿java和Kotlin对比泛型机制异同的时候,又发现了一些关于java泛型的,我之前不知道的知识。这里再把它们记录下来。

# 关于泛型通配符

## 关于<? extends T>

首先假设有下面的一个list:

```
List<? extends Number> list = new ArrayList<>();
```

我们是不能向它添加除null以外的任意对象的,即使是Number的子类:

```
list.add(null); // ok
list.add(new Integer(1)); // error
list.add(new Float(1.0f)); // error
```

这是为什么呢?我们来再来看下面的代码:

```
List<Integer> listOri = new ArrayList<>();
List<? extends Number> list = listOri;
```

listOri中只能存放Integer。

但是假设我们能向List<? extends Number>中添加Number的子类,那么我们就能将Float、Double这样的非Integer的类放到list中。

这样的话我们就会打破listOri中的类型一致性。而只有将null,放到list中不会打破listOri的类型一致性。

## 关于<? super T>

和上面的<? extends T>不同,我们可以向List<? super Number>中加入null和Number的任意子对象:

```
List<? super Number> list = listOri;
list.add(null); // ok
list.add(new Integer(1)); // ok
list.add(new Float(1.0f)); // ok
list.add(new Object()); // error
```

因为List<? super Number>中存放的都是Number的父类,而Number的子类都可以转化成Number,也就可以转化成Number的这个父类。所以就能保证list中类型的一致性。

# Collection方法中的Object参数

我有注意到Map的一些方法的参数并不是泛型参数,而是Object：

```
boolean containsKey(Object key);
boolean containsValue(Object value);
V get(Object key);
V remove(Object key);
...
```

其实不止Map包括其他的容器其实都是一样的,我们能在Collectiond接口中看到下面的方法:

```
boolean remove(Object o);
boolean contains(Object o);
...
```

它们都不是用泛型参数,而是直接用的Object,这是为什么呢?

Java 集合框架创办人，Josh Bloch 是这么说的:

> Josh Bloch says (6:41) that they attempted to generify the get method of Map, remove method and some other, but "it simply didn't work". There are too many reasonable programs that could not be generified if you only allow the generic type of the collection as parameter type. The example given by him is an intersection of a List of Numbers and a List of Longs.

他们其实有想过用泛型参数去实现Map的get方法,但是出现了一些状况导致它出问题了,比如说用List<Number>做Key,但却想用List<Long>来get。

stackoverflow上也有大神这么说:

```
Actually, it's very simple! If add() took a wrong object, it would break the collection. It would contain things it's not supposed to! That is not the case for remove(), or contains(). – Kevin Bourrillion Nov 7 '09 at 3:46

Incidentally, that basic rule -- using type parameters to prevent actual damage to the collection only -- is followed absolutely consistently in the whole library.  – Kevin Bourrillion Nov 7 '09 at 3:49
```

因为像add方法这样的往集合中添加元素的方法,如果用Object参数的话,会破坏集合中的类型安全性。但是像remove(),contains()这些方法其实只需要equals成立即可,不需要限制类型。java库的原则就是只用类型参数去保护集合的类型安全性不会被破坏,不做多余的事情。
