title: 深入理解Dart Mixin
date: 2022-04-27 23:00:23
tags:
    - 技术相关
    - Dart
---


假设我们需要实现一个动物世界的功能。Animal作为基类派生出哺乳类、鸟类、鱼类三种类型,各个类型又能派生出具体的动物。每种动物都具有步行、游泳、飞行三种能力中的某几种能力:

{% img /深入理解Dart_Mixin/1.png %}

由于Java和kotlin都不允许多继承，我们可以将walk、swim、fly定义成interface，让各个具体的动物类去实现这几个接口。在java7里面需要在不同动物类中写同样的实现代码，但如果用java8或者kotlin，可以在interface中编写默认实现去避免重复代码。而在dart中我们要怎么实现呢?

# dart extends & implements

首先dart里面是没有interface的, 但是我们可以把class当做接口被实现。使用implements把某个class当做接口来实现要求我们重写这个class的**所有**方法, 而使用extends继承某个class则可以继承父类实现了的方法:

```
class Base {
  void foo1() {
  }

  void foo2() {
  }
}

class Child1 implements Base {
  @override
  void foo1() {
    // TODO: implement foo1
  }

  @override
  void foo2() {
    // TODO: implement foo2
  }
}

class Child2 extends Base {
  
}
```

implements会将class的实现抹掉就不存在默认实现一说,而dart也是不允许多继承的。那么我们只能将walk、swim、fly三个接口在不同动物类中重复实现一遍吗?

其实dart里有个叫做mixin的概念可以解决上面的问题

# mixin

mixin实际上也是面向对象编程中的概念,在[维基百科](https://zh.wikipedia.org/wiki/Mixin)上对它的解释如下:

> Mixin是面向对象程序设计语言中的类，提供了方法的实现。其他类可以访问mixin类的方法而不必成为其子类。[1]Mixin有时被称作"included"而不是"inherited"。mixin为使用它的class提供额外的功能，但自身却不单独使用（不能单独生成实例对象，属于抽象类）。因为有以上限制，Mixin类通常作为功能模块使用，在需要该功能时“混入”，而且不会使类的关系变得复杂。用户与Mixin不是“is-a”的关系，而是“-able”关系

dart语言里面我们可以使用with关键字实现mixin,将一个或者多个class混入另一个类:

```
class Base1 {
  void foo1() {
    print("foo1");
  }
}

class Base2 {
  void foo2() {
    print("foo2");
  }
}

class Child2 with Base1, Base2 {

}
```

没错,通过with多个类,可以实现类似多继承的效果。

既然允许with多个类,那么如果这些类中有个相同方法,那会出现什么事情。实际上kotlin、java8使用接口的默认实现也会出现一样的问题，他们的处理方法是当出现相同方法的时候实现类需要手动指定使用哪个接口的默认实现,要不然编译会报错:

```java
// java8
interface IBase1 {
    default void foo() {
        System.out.println("1");
    }
}

interface IBase2 {
    default void foo() {
        System.out.println("2");
    }
}

class Child implements IBase1, IBase2 {

    @Override
    public void foo() {
        IBase2.super.foo();
    }
}
```

```kotlin
//kotlin
interface IBase1 {
    fun foo() {
        println("1")
    }
}

interface IBase2 {
    fun foo() {
        println("2")
    }
}

class Child : IBase1, IBase2 {
    override fun foo() {
        super<IBase2>.foo()
    }
}
```

# 线性化

而在dart with里面越后面的类优先级越高:

```
class Base1 {
  void foo() {
    print("1");
  }
}

class Base2 {
  void foo() {
    print("2");
  }
}

class Base3 {
  void foo() {
    print("3");
  }
}

class Child extends Base1 with Base2, Base3 {}
```

这个时候调用Child.foo方法实际会优先调用Base3.foo。原因是dart实际是通过创建中间类继承实现的mixin,上面的代码相当于:


{% plantuml %}
class Base1 {
  + foo()
}

class Base1WithBase2 extends Base1 {
  + foo()
}

class Base1WithBase2WithBase3 extends Base1WithBase2 {
  + foo()
}

class Child extends Base1WithBase2WithBase3
{% endplantuml %}

通过从左到右的顺序生成中间父类去继承将extends、with线性化成一个单继承链。所以**Base2、Base3实际上不是Child的父类**

# mixin关键字

在上面的例子中我们使用普通的class去with，但dart实际上提供了一个mixin关键字，它定义了不能实例化，也不能extends只能with的类:

```
mixin Base {

}

// 编译失败: mixin类不能extends
// class Child extends Base {
//
// }

// 编译成功: mixin类可以with
class Child with Base {

}

void main() {
  // 编译失败: mixin类不能实例化
  // Base()
}
```

这样的类实际上和java、kotlin里面的interface已经很像了。

另外我们可以通过mixin ... on 限定某个类只能由某些类去with:

```
class Base1 {
  void foo() {
    print("1");
  }
}

class Base2 {
  void foo() {
    print("2");
  }
}

mixin Base3 on Base1 {
  void foo() {
    super.foo();
    print("3");
  }
}

class Child extends Base1 with Base2, Base3 {}
```

上面的demo中Base3只能由Base1去with,那就以为着这个with Base3的类一定是继承或者with了 Base1,所以可以调用这个类的super.foo方法。要注意的是，这个super.foo并不指定一定调用的是Base1.foo。例如上面的代码调用Child().foo()之后的打印实际上是:

```
2
3
```

它们线性化的到的继承关系和前面全是class的代码并没有差别:


{% plantuml %}
class Base1 {
  + foo()
}

class Base1WithBase2 extends Base1 {
  + foo()
}

class Base1WithBase2WithBase3 extends Base1WithBase2 {
  + foo()
}

class Child extends Base1WithBase2WithBase3

note left of Base1::foo
  print("1");
end note

note left of Base1WithBase2::foo
  print("2");
end note

note left of Base1WithBase2WithBase3::foo
  super.foo();
  print("3");
end note
{% endplantuml %}


从上面的uml图我们就能理解为什么打印是23了

理解了这个简单的例子之后我们再来看一个复杂一点的例子:

```
class Base1 {
  void foo1() {
    print("Base1.foo1");
    foo2();
  }

  void foo2() {
    print("Base1.foo2");
  }
}

mixin Base2 on Base1 {
  void foo1() {
    super.foo1();
    print("Base2.foo1");
  }

  void foo2() {
    print("Base2.foo2");
  }
}

class Child with Base1,Base2 {}

void main() {
  Child().foo1();
}
```

它的输出是:

```
Base1.foo1
Base2.foo2
Base2.foo1
```

原因是Base2.foo1中的super.foo1实际上调用的是Base1.foo1,而Base1.foo1中的foo2,由于继承的多态特性,调用的是Base2.foo2。我们可以通过下面uml图辅助理解,**注意看继承关系里面是没有Base1、Base2的因为它们都是通过with混入的,并不是Child的父类**:

{% plantuml %}
class WithBase1 {
  + foo1()
  + foo2()
}

class WithBase1WithBase2 extends WithBase1 {
  + foo1()
  + foo2()
}

class Child extends WithBase1WithBase2


note left of WithBase1::foo1
  print("Base1.foo1");
  foo2();
end note

note left of WithBase1WithBase2::foo1
  super.foo1();
  print("Base2.foo1");
end note

note left of WithBase1WithBase2::foo2
  print("Base2.foo2");
end note
{% endplantuml %}

# with的类不能有构造函数

另外,with的class和mixin类型都是不允许有构造函数的,因为mixin机制语义上是向一个类混入其他类的方法或者成员变量,使得我们可以在混合类中访问到混入类的方法或者属性。而混入其他类的构造函数实际上是没有意义的,因为不会有人手动去调用这个混入类的构造函数。

```
class Base1 {
  Base1() {}
}

// 编译失败: 不能with一个带有构造函数的类
// class Child with Base1 {}

// 编译失败: mixin类型只能with,所以不能有构造函数
// mixin Base2 {
//   Base2() {}
// }
```

# 参考博客

https://medium.com/flutter-community/dart-what-are-mixins-3a72344011f3

https://www.jianshu.com/p/f4efaa6b8fe6