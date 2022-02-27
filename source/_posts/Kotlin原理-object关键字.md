title: Kotlin原理-object关键字
date: 2022-02-27 14:10:07
tags:
    - 技术相关
    - Android
---

object关键字有下面的三种用法:

1. 对象声明
2. 伴生对象
3. 对象表达式

我们逐一来看看它们的实现原理

# 对象声明

object类可以看成是java里面的单例模式在kotlin的便捷实现:

```
object TestObject {
    const val TAG = "TestObject"

    fun foo() {
    }
}

// kotlin 中调用
TestObject.foo()

// java 中调用
TestObject.INSTANCE.foo();
```

从java的用法里面能看出来,对象声明的原理实际上是将其转换成了java的单例模式:

```
public final class TestObject {
    @NotNull
    public static final String TAG = "TestObject";
    @NotNull
    public static final TestObject INSTANCE;

    public final void foo() {
    }

    private TestObject() {
    }

    static {
        TestObject var0 = new TestObject();
        INSTANCE = var0;
    }
}
```

如果与java混用的时候，需要使用INSTANCE获取单例比较麻烦,可以使用@JvmStatic注解将方法修饰成静态方法:

```
object TestObject {
    @JvmStatic
    fun foo() {
    }
}
```

转换出来的java类就会把foo()定义为静态方法:

```
public final class TestObject {
    @NotNull
    public static final TestObject INSTANCE;

    // 这里是静态方法
    @JvmStatic
    public static final void foo() {
    }

    private TestObject() {
    }

    static {
        TestObject var0 = new TestObject();
        INSTANCE = var0;
    }
}
```

# 伴生对象

Kotlin中并没有静态成员的概念,但是我们可以使用伴生对象达到类似的效果:

```
class TestClass {
    companion object {
        var data = 123
        fun foo() {}
    }
}


// kotlin 中调用
TestClass.foo()

// java 中调用
TestClass.Companion.foo();
```

同样可以在java调用中看出来它实际是生成了TestClass的Companion静态内部类:

```
public final class TestClass {
    private static int data = 123;

    @NotNull
    public static final TestClass.Companion Companion = new TestClass.Companion((DefaultConstructorMarker)null);

    ...
   
    public static final class Companion {
        public final void foo() {
        }

        private Companion() {
        }

        public final int getData() {
            return TestClass.data;
        }

        public final void setData(int var1) {
            TestClass.data = var1;
        }
        ...
    }
}
```

同样在java调用的时候需要引用Companion静态成员变量,我们可以用@JvmField和@JvmStatic去简化:

```
class TestClass {
    companion object {
        @JvmField
        var data = 123

        @JvmStatic
        fun foo() {}
    }
}
```

上面kotlin代码转换的java代码如下:

```
public final class TestClass {
    public static int data = 123;

    public static final TestClass.Companion Companion = new TestClass.Companion((DefaultConstructorMarker)null);

    public static final void foo() {
        Companion.foo();
    }

    public static final class Companion {
        @JvmStatic
        public final void foo() {
        }

        private Companion() {
        }

        ...
    }
}
```

# 对象表达式

对象表达式实际上就是java的匿名内部类:

```
var listener = object : View.OnClickListener {
    override fun onClick(p0: View?) {

    }
}
```

它转换成的java代码如下:

```
OnClickListener var10000 = new OnClickListener() {
    public void onClick(@Nullable View p0) {
    }
};
```