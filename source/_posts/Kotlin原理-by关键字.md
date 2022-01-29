title: Kotlin原理-by关键字
date: 2022-01-29 10:08:38
tags:
    - 技术相关
    - Android
---

委托模式也叫代理模式,指的是一个对象接收到请求之后将请求转交由另外的对象来处理,它也是继承的一种很好的替代方式,可以实现用组合替代继承。

Kotlin内置了一个by关键字,可以很方便的实现代理.

# 委托类

借用[Kotlin中文站](https://www.kotlincn.net/docs/reference/delegation.html)的例子:

```
interface Base {
    fun print()
}

class BaseImpl(val x: Int) : Base {
    override fun print() { print(x) }
}

class Derived(b: Base) : Base by b

fun main() {
    val b = BaseImpl(10)
    Derived(b).print()
}
```

Derived的所有请求会被转发给传入的b对象,它的实现原理实际上是编译器帮我们补全了Derived的方法,将传入的b对象保存起来,然后在补全的方法内去调用b对象:

```
public final class Derived implements Base {
    private final /* synthetic */ Base $$delegate_0;

    public void print() {
        this.$$delegate_0.print();
    }

    public Derived(Base b) {
        Intrinsics.checkNotNullParameter(b, "b");
        this.$$delegate_0 = b;
    }
}
```

by关键字的好处在于如果Base接口有多个方法需要实现,而我们只想对其中一个方法进行改造,例如统计print的调用次数,那么可以在Derived里面只实现print方法,而其他的方法由by关键字自动生成:

```
interface Base {
    fun print()
    fun method1()
    fun method2()
    fun method3()
    fun method4()
    fun method5()
}

class Derived(private val b: Base) : Base by b {
    var printInvokeCount = 0
        private set

    override fun print() {
        printInvokeCount++
        this.b.print()
    }
}
```

by关键字虽然方便但是也有限制,那就是它只能委托接口的方法,如果把Base改成class而不是interface,Derived就不能使用by去委托了。

# 委托属性

除了整个类进行委托之外,我们也可能对类的成员变量进行委托:

```
class StringDelegate {
    private lateinit var str: String

    // 实现get委托方法
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        println("get $thisRef.${property.name}")
        return str
    }

    // 实现set委托方法
    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        println("set $thisRef.${property.name} to $value")
        str = value
    }
}

class Data {
    var str by StringDelegate()
}
```

它的原理实际上是在Data类里面将这个StringDelegate给保存了起来,然后在getStr/setStr里面去调用它的对应方法:

```
public final class Data {
   // $FF: synthetic field
   static final KProperty[] $$delegatedProperties = new KProperty[]{(KProperty)Reflection.mutableProperty1(new MutablePropertyReference1Impl(Data.class, "str", "getStr()Ljava/lang/String;", 0))};

   @NotNull
   private final StringDelegate str$delegate = new StringDelegate();

   @NotNull
   public final String getStr() {
      return this.str$delegate.getValue(this, $$delegatedProperties[0]);
   }

   public final void setStr(@NotNull String var1) {
      Intrinsics.checkNotNullParameter(var1, "<set-?>");
      this.str$delegate.setValue(this, $$delegatedProperties[0], var1);
   }
}
```

## by lazy原理

基于上面的属性委托原理,我们很容易就能实现自己的by lazy:

```
class MyLazy<T>(initializer: () -> T) {
    companion object {
        val UNINITIALIZED_VALUE = Object()
    }

    private var initializer: (() -> T)? = initializer
    private var value: Any? = UNINITIALIZED_VALUE

    operator fun getValue(thisRef: Any?, property: KProperty<*>): T {
        if (value == UNINITIALIZED_VALUE) {
            value = initializer!!()
            initializer = null // 把初始化方法置空,避免其引用外部类引用造成内存泄露
        }
        return value as T
    }
}

class Data {
    val data by MyLazy { null }
}
```

实际上kotlin的lazy原理也差不多是这样了:

```
public actual fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)

private class SynchronizedLazyImpl<out T>(initializer: () -> T, lock: Any? = null) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer

    @Volatile private var _value: Any? = UNINITIALIZED_VALUE

    private val lock = lock ?: this

    override val value: T
        get() {
            val _v1 = _value
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }

            return synchronized(lock) {
                val _v2 = _value
                if (_v2 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST") (_v2 as T)
                } else {
                    val typedValue = initializer!!()
                    _value = typedValue
                    initializer = null
                    typedValue
                }
            }
        }

    ...
}
```

而且可以看到lazy在初始化对象的时候会对初始化的代码块使用synchronized上锁,所以是线程安全的。这个锁我们可以外部传入,也可以默认使用SynchronizedLazyImpl的this指针

当然如果我们觉得这个synchronized加锁会影响性能,也可以使用lazy的重载方法去指定线程安全策略:

```
public actual fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T> =
    when (mode) {
        LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)  // 使用synchronized加锁
        LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer) //使用cas机制保证线程安全
        LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer) // 不加锁,不考虑线程安全问题
    }
```
