title: 怎样获取Java泛型的具体类型
date: 2021-11-20 14:19:38
tags:
  - 技术相关
  - java
---
之前写过一篇[博客](https://blog.islinjw.cn/2018/01/06/java%E6%B3%9B%E5%9E%8B%E9%82%A3%E4%BA%9B%E4%BA%8B/)介绍过Java泛型的类型擦除机制，实际上Java的泛型是使用所有类的公共父类Object去实现:

```java
//错误,泛型的类型被擦除了,T在运行时实际上是Object,不能直接new出来
public <T> void foo(T arg) {
    return arg != null ? arg : new T();
}
```

那是不是运行的时候就真的无法获取到泛型使用的实际类型呢？

我们可以从一些第三方库找到答案,例如Gson,它就支持解析泛型类型:

```java
public static class Foo<T> {
	T data;
}

Type type = new TypeToken<Foo>() {}.getType();
Foo foo = new Gson().fromJson(json, type);
```

它是怎么做到的呢？

# 为什么拿不到泛型的具体类型

我们一步步来探索这个问题。先写一个简单的例子:

```java
import java.lang.reflect.Field;

public class GenericsDemo {
	public static void main(String[] args) throws NoSuchFieldException {
		Foo<String> foo = new Foo<String>();
        Class fooClass = foo.getClass();
        Field field = fooClass.getField("data");
    
        System.out.println(fooClass);
        System.out.println(field.getType());
        System.out.println(field.getGenericType());
	}

	public static class Foo<T> {
		public T data;
	}
}
```

执行之后看到打印里面泛型的实际类型String已经不见了，变成了Object:

> class GenericsDemo$Foo
> class java.lang.Object
> T

所以我们也就没有办法知道new Foo的时候指定的String类型。

# 继承泛型类指定泛型类型

这里还涉及到一个类加载的知识点，getClass方法拿到的实际是类加载器加载的类信息，而类信息在编译的时候就已经确定了。我们用"javac GenericsDemo.java"编译这个java代码，可以看到它生成了两个.class文件:

> GenericsDemo$Foo.class GenericsDemo.class     GenericsDemo.java

GenericsDemo$Foo.class就是内部类Foo的信息了，使用"javap GenericsDemo\$Foo"命令查看内容,可以发现它里面没有运行时的类型信息，**也就是说在编译Foo类的时候，编译器没有办法确定它的具体泛型类型**:

```
Compiled from "GenericsDemo.java"
public class GenericsDemo$Foo<T> {
  public T data;
  public GenericsDemo$Foo();
}
```

但是如果我们继承Foo并且指定泛型类型:

```java
public static class StringFoo extends Foo<String> {
}
```

再使用"javap GenericsDemo\$StringFoo"命令去查看它编译生成的GenericsDemo$StringFoo.class文件:

```
Compiled from "GenericsDemo.java"
public class GenericsDemo$StringFoo extends GenericsDemo$Foo<java.lang.String> {
  public GenericsDemo$StringFoo();
}
```

会发现class文件里携带了泛型的类型信息，**也就是说StringFoo在编译的时候就能确定其泛型类型**。既然class文件携带了信息，那么java虚拟机在加载的时候就能把这些信息加载进去。

但是要怎么获取到它呢？如果使用getClass().getSuperclass()去获取父类信息是拿不到泛型的信息的，但是java提供了一个getGenericSuperclass的方法可以获取:

```java
import java.lang.reflect.Field;

public class GenericsDemo {
	public static void main(String[] args) throws NoSuchFieldException {
		Foo<String> foo = new StringFoo();
        System.out.println(foo.getClass().getSuperclass());
        System.out.println(foo.getClass().getGenericSuperclass());
	}

	public static class Foo<T> {
		public T data;
	}

	public static class StringFoo extends Foo<String> {
	}
}
```

> class GenericsDemo$Foo
> GenericsDemo$Foo<java.lang.String>

如果父类使用了泛型那么getGenericSuperclass拿到的Type实际是ParameterizedType类型，可以通过其getActualTypeArguments方法获取泛型参数:

```java
Foo<String> foo = new StringFoo();
ParameterizedType superGenericSuperclass = (ParameterizedType) foo.getClass().getGenericSuperclass();
for (Type arg : superGenericSuperclass.getActualTypeArguments()) {
	System.out.println(arg);	
}
```

打印如下:

> class java.lang.String

所以我们可以再配合class的getTypeParameters方法确定在运行的时候每个泛型参数对应的具体是什么:

```java
Foo<String> foo = new StringFoo();
ParameterizedType superGenericSuperclass = (ParameterizedType) foo.getClass().getGenericSuperclass();
Class superClass = foo.getClass().getSuperclass();
for (int i = 0; i < superGenericSuperclass.getActualTypeArguments().length; i++) {
    System.out.println(superClass.getTypeParameters()[i] + " -> " + superGenericSuperclass.getActualTypeArguments()[i]);
}
```

打印如下:

> T -> class java.lang.String

# 封装工具类

所以如果需要在运行的时候获取泛型的具体类型，可以写一个子类去继承它，然后使用getGenericSuperclass获取父类的泛型信息。但是这样的方式比较死板，有没有什么好的方式可以不用继承去实现呢?

我们看看下面的代码:

```java
System.out.println(new Foo<String>().getClass().getGenericSuperclass());
System.out.println(new Foo<String>(){}.getClass().getGenericSuperclass());
```

它的打印如下：

> class java.lang.Object
> GenericsDemo$Foo<java.lang.String>

可以看到我们只是在new的后面加上了{},就能通过getClass().getGenericSuperclass()去获取泛型信息了。

这样是因为加上{}之后实际上已经是一个匿名内部类了。通过“javac GenericsDemo.java”命令编译，可以看到编出出来多了个“GenericsDemo$1.class”文件。

匿名内部类的实现原理实际上就是给它创建一个名为GenericsDemo$XXX的子类,所以从他的class信息里面可以看到父类的泛型信息:

```
Compiled from "GenericsDemo.java"
final class GenericsDemo$1 extends GenericsDemo$Foo<java.lang.String> {
  GenericsDemo$1();
}
```

基于上面的知识点:

1. 匿名内部类会生成一个子类
2. 生成的子类携带了父类的泛型信息

我们可以写一个抽象的Token类,强制必须创建匿名内部类。然后通过查找父类的泛型信息的方式获取具体的泛型类型:

```java
public class GenericsDemo {
    @Test
    public void main() {
        new Token<Foo<String>>() {}.parseGenericInfo();
    }

    public static class Foo<T> {
        T data;
    }

    public static abstract class Token<T> {
        public void parseGenericInfo() {
            // GenericsDemo$Token<GenericsDemo$Foo<java.lang.String>>
            ParameterizedType genericSuperclass = (ParameterizedType) this.getClass().getGenericSuperclass();

            // GenericsDemo$Foo<java.lang.String>
            ParameterizedType targetGenericClass = (ParameterizedType) genericSuperclass.getActualTypeArguments()[0];

            // class GenericsDemo$Foo
            Class targetClass = (Class) targetGenericClass.getRawType();

            for (int i = 0; i < targetGenericClass.getActualTypeArguments().length; i++) {
                System.out.println(targetClass.getTypeParameters()[i] + " -> " + targetGenericClass.getActualTypeArguments()[i]);
            }
        }
    }
}
```

可以看到这个代码已经和Gson里面解析泛型的代码类似了:

```java
Type type = new TypeToken<Foo>() {}.getType();
Foo foo = new Gson().fromJson(json, type);
```

