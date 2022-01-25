title: Kotlin原理-拓展函数
date: 2022-01-25 21:06:48
tags:
    - 技术相关
    - Android
---

# 原理

拓展函数是kotlin里一个比较常用的特性,例如我们可以给Context拓展一个toast方法:

```
// MainActivity.kt
fun Context.toast(msg: String) {
    Toast.makeText(this, msg, Toast.LENGTH_SHORT).show()
}

private fun foo(context: Context) {
    context.toast("hello world")
}
```

它的原理其实很简单,就是生成了一个toast方法。拓展函数的this指针实际上是这个生成方法的第一个参数:

```
/* compiled from: MainActivity.kt */
public final class MainActivityKt {

	public static final void toast(Context $this$toast, String msg) {
		//参数判空
	    ...

	    // 拓展函数代码
	    Toast.makeText($this$toast, msg, 0).show();
	}
}
```


所以这个this指针实际上是由函数调用的地方传入的对象引用:

```
private final void foo(Context context) {
    MainActivityKt.toast(context, "hello world");
}
```

# 限制

知道了拓展函数的实现原理之后我们就能从原理去理解拓展函数的种种限制.

## 不能访问私有成员

由于编译成java之后,生成的拓展方法实际是靠第一个参数出入对象引用,然后使用这个对象引用去调用对象的方法。因此我们并没有权限在拓展函数里面调用私有方法:

```
class TestClass {
    fun publicFun() {}
    private fun privateFun() {}
}

fun TestClass.extFun() {
    publicFun() // 正确,可以调用公有方法

    privateFun() // 错误,不能调用私有方法
}
```

## 拓展函数不能实现多态

由于拓展函数并不是真的给类增加一个成员函数,所以父类和子类的同名拓展函数并没有多态的特性。

例如我们为父类和子类拓展同一个foo()函数:

```
open class Parent
class Child : Parent()

fun Parent.foo() {
    println("parent")
}

fun Child.foo() {
    println("child")
}
```

然后只要将子类转换成父类,调用的拓展函数就是父类的拓展函数:

```
val child = Child()
child.foo()
(child as Parent).foo()

// 输出:
// child
// parent
```

## 成员函数优先级高,拓展函数不能实现重写

当拓展函数与类本身或者父类的成员函数相同,在实际调用的时候会优先调用成员函数,并不会出现类似重写的效果.

例如我们为一个类编写了一个与成员函数相同的拓展函数,实际优先调用类成员函数:

```
open class Parent {
    fun foo() {
        println("foo")
    }
}

fun Parent.foo() {
    println("parent")
}

Parent().foo()

// 输出:
// foo
```

就算是为子类编写了一个与父类成员函数相同的拓展函数,也会优先调用父类的成员函数:

```
open class Parent {
    fun foo() {
        println("foo")
    }
}

class Child : Parent()

fun Child.foo() {
    println("child")
}

Child().foo()

// 输出:
// foo
```
