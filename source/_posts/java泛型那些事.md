title: java泛型那些事
date: 2018-01-06 01:11:02
tags:
	- 技术相关
	- java
---

# 泛型的类型安全性

有许多原因促成了泛型的出现，而最引人注意的一个原因，就是为了创建容器类。

如果没有泛型,如果我们需要实现一个通用的队列,那么只能使用Obejct数组去实现,并且add方法的参数和get方法的返回值都为Object:

```
public class MyList {
    private Object[] mData;

    public void add(Object obj) {
        ...
    }

    public Object get(int index) {
        ...
    }
    ...
}
```

但是这样的话其实是很不安全的,类型安全需要靠用户去自己维护。但用户往往都是愚蠢的:

```
MyList myList = new MyList();
myList.add("1");
myList.add("2");
myList.add(3);

String val1 = (String) myList.get(0);
String val2 = (String) myList.get(1);
String val3 = (String) myList.get(2);
```

上面的代码在编译的时候没有问题,但是真正运行的时候程序跑着跑着就挂了,这就叫做类型不安全的设计。

使用泛型的意义在于它是类型安全的,如果使用泛型规定了参数和返回值的类型的话,上面的代码在编译的时候就会失败:


```
public class MyList<E> {
  private Object[] mData;

  ...

  public void add(E obj) {
    ...
  }

  public E get(int index) {
    ...
    return (E) mData[index];
  }
}

MyList<String> myList = new MyList<>();
myList.add("1");
myList.add("2");
myList.add(3); //这里会编译失败

String val1 = myList.get(0);
String val2 = myList.get(1);
String val3 = myList.get(0);
```

# 类型标识符

在MyList&lt;E&gt;声明尖括号里面的就是类型标识符,它其实是一个占位符,代表了某个类型,我们在类里面就能用这个占位符代表某种类型。例如add方法的参数或者get的返回值,当然也能用来声明一个成员变量。

可能有人会说经常看到都是用T泛型作为泛型标识符,为什么这里我们用E呢?

其实用什么字母做标识符在java里面并没有硬性规定,甚至你也可以不用仅一个字符，用一个单词也是可以的。

不过我们通常会按照习惯在不同场景下用不同的字母标识符:

- E - Element (在集合中使用)
- T - Type（Java 类）
- K - Key（键）
- V - Value（值）

# 泛型通配符

在泛型中有个很重要的知识点就是__泛型类型之间是不具有继承关系的__,也就是说List&lt;Object&gt;并不是List&lt;String&gt;的父类:

```
public void printList(List<Object> list) {
    for (int i = 0; i < list.size(); i++) {
        System.out.println(list.get(i));
    }
}


List<String> strList = Arrays.asList("a", "b", "c", "d", "e");
printList(strList); //错误,List<Object>不是List<String>的父类
```

为了实现上面的printList方法,类型通配符就出现了:


```
public void printList(List<?> list) {
    for (int i = 0; i < list.size(); i++) {
        System.out.println(list.get(i));
    }
}

List<String> strList = Arrays.asList("a", "b", "c", "d", "e");
printList(strList);
```

List&lt;?&gt;可以匹配List&lt;String&gt;、List&lt;Integer&gt;等等的各种类型。

大家有可能会听过类型通配符上限和下限,这两个东西是怎样的概念呢？有时候我们会需要限定只能传入某些型的子类或者父类的容器:


- 上限：<? extends T> 只能匹配T和T的子类

- 下限：<? super T> 只能匹配T和T的父类

```
//只能传入ClassA的子类的容器
public void printList(List<? extends ClassA> list) {
    for (int i = 0; i < list.size(); i++) {
        System.out.println(list.get(i));
    }
}

//只能传入ClassA的父类的容器
public void printList(List<? super ClassA> list) {
    for (int i = 0; i < list.size(); i++) {
        System.out.println(list.get(i));
    }
}
```

除了上面的通配符"?"，我们还可以直接用泛型方法去实现printListde,可以指定所有类型的列表或者ClassA的子类的列表:

```
public <T> void printList(List<T> list) {
    for (int i = 0; i < list.size(); i++) {
        System.out.println(list.get(i));
    }
}

public <T extends ClassA> void printList(List<T> list) {
    for (int i = 0; i < list.size(); i++) {
        System.out.println(list.get(i));
    }
}
```

当然我们也能使用泛型的方式直接指定参数的上限,比如下面的foo方法就只能接收Number的子类:

```
public <T extends Number> void foo(T arg){
    ...
}
```

但是如果直接使用泛型的方式的话我们不能指定指定它的下限,例如下面两种写法都是__不能通过编译__的:

```
//错误.不能直接指定泛型的下限
public <T super Number> void printList(List<T> list) {
    ...
}

//错误.不能直接指定泛型的下限
public <T super Number> void foo(T arg){
    ...
}
```

# 类型擦除

可能很多同学都会听说过泛型类型擦除的概念,这个类型擦除具体指的是怎样一回事？

可以看看下面的foo方法,它本来想实现的功能是:如果传入的参数非空,就将它返回。否则,就创建一个同类型的实例返回。但是这段代码是不能通过编译的:

```
//错误,泛型的类型被擦除了,不能直接new出来
public <T> void foo(T arg) {
    return arg != null ? arg : new T();
}
```

原因在于java的泛型实现中有个叫做类型擦除的机制。简单来讲就是运行的时候是无法获取到泛型使用的实际类型的。

例如上面的T类型,因为我们在运行时不能知道它到底是什么类型,所以也就无法将它new出来。

java代码生成的Java字节代码中是不包含泛型中的类型信息的,所有泛型类的类型参数在编译时都会被擦除。虚拟机中没有泛型,只有普通类和普通方法。因此泛型的类型安全是在编译的时候去检测的。

所以我们创建泛型对象时需要指明类型，让编译器尽早的做参数检查。

像下面的代码可以顺利通过,甚至可以正常运行,直到将获取到的数值类型的数据强转成字符串的时候才报ClassCastException异常:

```
List list = new ArrayList<String>();
list.add("abc");
list.add(123);
String elemt1 = (String) list.get(0);
String elemt2 = (String) list.get(1); // java.lang.ClassCastException: java.lang.Integer cannot be cast to java.lang.String
```

我们可以用反射的方法的验证一下类型擦除:

```
List<Integer> list = new ArrayList<Integer>();
System.out.println("type : " + Arrays.toString(list.getClass().getTypeParameters()));
```

它得到的类型仅仅是一个占位符而已:

> type : [E]

# 类型擦除机制的历史原因

有人会问,为什么java会在编译的时候将类型擦除,而不像ｃ++一样通过在编译的时候将泛型类实例化为多个具体的类去实现泛型呢？

其实“实例化为多个具体的类”这样的实现方式也是比较容易实现的,但是为了保持兼容性,所以java在泛型的实现上选取类型擦除的方式。实际上是做了一定的取舍的。

为什么说选用类型擦除是为了保持兼容性呢?因为泛型并不是java与生俱来的。实际上到了java５的时候才引入了泛型。

要让以前编译的程序在新版本的JRE还能正常运行，就意味着以前没有的限制不能突然冒出来。

例如在泛型出来之前java就已经有了容器的存在,而且它具有可以存储不同类型的的特性:

```
ArrayList things = new ArrayList();
things.add(Integer.valueOf(123));
things.add("abc");
```

那么这段代码在Java 5引入泛型之后还必须要继续可以运行。

这里有两种设计思路：

1. 需要泛型化的类型（主要是容器（Collections）类型），以前有的就保持不变，然后平行地加一套泛型化版本的新类型；

2. 直接把已有的类型泛型化，让所有需要泛型化的已有类型都原地泛型化，不添加任何平行于已有类型的泛型版。

.NET在1.1 -> 2.0的时候选择了上面选项的1，而Java则选择了2。

从Java设计者的角度看，这个取舍很明白：.NET在1.1 -> 2.0的时候，实际的应用代码量还很少（相对Java来说），而且整个体系都在微软的控制下，要做变更比较容易；

而在Java 1.4.2 -> 5.0的时候，Java已经有大量程序部署在生产环境中，已经有很多应用和库程序的代码。如果这些代码在新版本的Java中，为了使用Java的新功能（例如泛型）而必须做大量源码层修改，那么新功能的普及速度就会大受影响，而且新功能会被吐槽。

在原地泛型化后，java.util.ArrayList这个类型变成了java.util.ArrayList&lt;E&gt;。但是以前的代码直接用ArrayList，在新版本里必须还能继续用，所以就引出了“raw type”的概念——一个类型虽然被泛型化了，但还可以把它当作非泛型化的类型用。

```
ArrayList         - raw type
ArrayList<E>      - open generic type (assuming E is type variable)
ArrayList<String> - closed generic type
ArrayList<?>      - unbounded wildcard type
```

下面这样的代码必须可以编译运行：

```
ArrayList<Integer> ilist = new ArrayList<Integer>();
ArrayList<String> slist = new ArrayList<String>();
ArrayList list; // raw type
list = ilist;   // assigning closed generic type to raw type
list = slist;   // ditto
```

所以java的设计者在考虑了这一点之后选用类型擦除也就显而易见了。类型擦除实际上是将泛型类型转换了Obejct。由于所有的java类都是Object的子类,所以实现起来就很简单了。只需要在编译的时候将所有的泛型占位符都换成Object就可以了:

```
//源码的泛型代码
public <T> void foo(T arg){
    ...
}

//编译时转换成的代码
public void foo(Object arg){
    ...
}
```

而在擦除类型的同时,java编译器会对该方法的调用进行类型检查,防止非法类型的调用。

但如果在编写代码的时候就已经用raw type的话,编译器就不会做类型的安全性检查了。

这样的实现导致了一个问题,List<E>泛型参数E被擦除后就变成了Object,那么就不能在泛型中使用int、long等原生数据类型了,因为它们并不是Object的子类。

据说当时设计java语言的程序员和产品经理打了一架,并且在打赢之后成功劝服产品经理在提出兼容性这样奇葩的需求之后做出一点小小的让步。（虽然只是我胡说八道的脑补,但谁知道当时的实际情形是不是这样的呢?）

于是乎我们现在在泛型中只能使用Integer、Long等封箱类型而不能用int、long等原生类型了。


ps: 上面这段类型擦除机制的历史原因参考了RednaxelaFX大神知乎上的一个回答,有兴趣的同学可以去知乎看看原来的[完整回答](https://www.zhihu.com/question/28665443/answer/118148143)
