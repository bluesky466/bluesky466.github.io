title: kotlin到底好在哪里
date: 2018-02-20 20:51:08
tags:
	- 技术相关
	- kotlin
---

最近在学kotlin,虽然还没有像其他博主一样立马就爱上它.但是不得不说,kotlin对比起java还是有不少优势的.

# 语法简洁

首先是语法比较简洁,能不简单处理的就不啰嗦.

## 变量与常量

变量通过var关键字定义,常量通过val关键字定义.既支持类型推倒也支持显示声明类型.这样的话定义变量和常量写起来会比java简洁一些:

```
var intVar = 1 // 通过取值可以自动推倒出变量的类型
var stringVar: String = "abc" // 也可以显式声明变量类型
val INT_VALUE = 1 // 使用val关键字定义常量
```

## 类

在kotlin中,类可以通过class关键字定义.支持在主构造方法中用var或者val关键字直接定义成员变量,例如下面的name和author,当然也支持在类结构体中定义成员变量,如price.

```
// 类,这个类有name,author和price三个成员变量
class Book(var name: String, val author: String, p: Int) {
    var price = p
        private set // 将price的set操作设置为私有,只能在类内部赋值,外部只能读取

    fun isTheSameAuthor(other: Book): Boolean {
        return author == other.author
    }
}
```

与java中不同的是,kotlin中所有的类成员变量都是通过类访问器get和set去访问成员变量的,例如下面的代码中对price的调用并不是直接操作price变量,而是通过get/set访问器去访问的.所以我们可以将price的set访问器设置为私有的,这样的话就不能在外部去设置price的值了:

```
var book = Book("book", "author", 100)
println(book.price) // 调用price的get访问器获取price的值
book.price = 123 // error,通过price的set访问器设置price的值,但是我们已经将它声明为私有了所以外部不能调用
```

假设我们用java去实现上面的Book类,我们需要写成下面这个样子.是不是感觉java的语法会啰嗦很多?实际上下面的java代码我还去掉了空指针处理和final关键字修饰.所以实际上代码量会更大.kotlin中短短的七行代码,java中要实现完全一模一样的功能,起码需要数十行代码.

```
public class Book {
   private int price;
   private String name;
   private String author;

   public Book(String name, String author, int p) {
      this.name = name;
      this.author = author;
      this.price = p;
   }

   public boolean isTheSameAuthor(Book other) {
      return Intrinsics.areEqual(this.author, other.author);
   }

   public int getPrice() {
      return this.price;
   }

   private void setPrice(int var1) {
      this.price = var1;
   }

   public String getName() {
      return this.name;
   }

   public void setName(String var1) {
      this.name = var1;
   }

   public String getAuthor() {
      return this.author;
   }
}
```

## 数据类

在kotlin中,我们还有可以用data class去定义单纯只是保存数据的数据类:

```
// 数据类,除了有name和age两个成员变量之外还自动生成了equals,hashCode,toString等方法
data class Person(var name: String, var age: Int)
```

有的同学可能会会,数据类和普通的类有什么区别呢?让我们看下下面的代码就一目了然了:

```
data class PersonData(var name: String, var age: Int)
class PersonClass(var name: String, var age: Int)

var dataA = PersonData("jack", 18)
var dataB = PersonData("jack", 18)
println(dataA) // PersonData(name=jack, age=18)
println(dataB) // PersonData(name=jack, age=18)
println(dataA == dataB) // true

var classA = PersonClass("peter", 20)
var classB = PersonClass("peter", 20)
println(classA) // me.linjw.demo.TestKotlin.PersonClass@5fcfe4b2
println(classB) // me.linjw.demo.TestKotlin.PersonClass@6bf2d08e
println(classA == classB) // false
```

假设用java去实现的话就不知道这一行的kotlin代码需要多少行才能实现了.

## object 对象

我们能使用object关键字直接实现单例模式:

```
object DataBaseHelper {
  ...
}
```

它翻译成java是这样的:

```
public class DataBaseHelper {
  public static final DataBaseHelper INSTANCE;

  private DataBaseHelper() {

  }

  static {
    INSTANCE = new DataBaseHelper();
  }
}
```

# 空指针安全

在kotlin中,每个类型都有其对应的可空类型,只有可空类型才能被赋值为null:

```
var a: Int = 1 // 非空类型需要初始化为非null
var b: Int? // 可空类型可以不初始化,默认为null
var c: String = "" // 非空类型需要初始化为非null
var d: String? = null // 可空类型也可以显式赋值为null
var e: Float = 1.0f // 非空类型需要初始化为非null
var f: Float? = 2.0f // 可空类型也可以显式赋值为非null

a = null // error类型不能赋值为null,编译不通过
b = null // ok
c = null // error类型不能赋值为null,编译不通过
d = null // ok
e = null // error类型不能赋值为null,编译不通过
f = null // ok
```

非空类型不可能为null,所以不用考虑空指针的问题.只有可空类型有可能为null,这个时候必须指定它为null的时候的行为.

可以用"?."做判空处理,或者使用"!!."在空指针的情况下抛出空指针异常:

```
fun foo(str : String?){
    println(str?.length) // str?.length表达式结果为null,可以正常运行
    println(str!!.length) // 抛出空指针异常
}
```

foo方法翻译成java是这样的:

```
public final void foo(@Nullable String str) {
  Integer var2 = str != null?Integer.valueOf(str.length()):null;
  System.out.println(var2);
  if(str == null) {
    Intrinsics.throwNpe(); // 抛出空指针异常
  }

  int var3 = str.length();
  System.out.println(var3);
}
```

很多文章在介绍kotlin的可空类型的时候都是以简化了判空处理的优点来介绍的.例如下面java方法中的判空处理,用kotlin只需要一行代码就可以了:

```
// java中需要自己判空
public Character front(String str) {
    if (str != null){
        return str.charAt(0);
    }
    return null;
}

// kotlin中用可空类型只需要一行代码
fun front(str: String?): Char? {
   return str?.get(0);
}
```

但是在我看来,可空类型非空类型的意义并不在帮我们做简化了判空处理的代码.更重要的一点是明确指出了哪些变量是可能为空的,同时明确了为空的时候的调用应该怎么处理.

对于非空类型,它永远不可能为空,我们不需要考虑它的空指针问题.而对于可空类型,我们需要明确指定在它为空的时候的调用抛出异常亦或进行执行.这样其实缩小了空指针异常的范围,同时也减小了空指针对程序稳定安全的破坏性.

# 字符串模板

在kotlin字符串中可以用"$"符号标识一个表达式,该表达式的值会被计算出来放到字符串中:

```
val str = "abc"
println("$str.length is ${str.length}") // 结果为 "abc.length is 3"
```

比起java用字符串拼接或者String.format的方式去处理都要优雅得多.

# 高阶函数和Lambda表达式的支持


在java中实现观察者模式,我们往往需要额外定义一个监听者的接口,这种监听者接口往往也只有一个方法,其实实际上属于比较冗余的代码,没有很大的实际价值:

```
public class Button {
    private OnClickListener mListener;

    public void setOnClickListener(OnClickListener listener) {
        mListener = listener;
    }

    public interface OnClickListener {
        void onClick(Button button);
    }
}
```

而在kotlin中我们可以将函数直接作为一个对象保存起来,再也不用为监听者单独定义一个接口了:

```
class Button {
  var listener: ((Button) -> Unit)? = null

  fun click() {
    listener?.invoke(this)
  }
}

var btn = Button()
btn.listener = { print("on click") }
btn.click()
```

# 泛型

在java泛型中存在类型通配符,用"? extends T"指定类参数的上限,用"? super T"指定类型参数的下限.

至于什么时候使用哪一种,在《Effect java》 中给出了PECS原则:

> PECS: producer-extends,consumer-super

在生产者中使用"? extends T",在消费者中使用"? super T".

java中使用类型的关系去设计了通配符,这样做的确在实现上是比较清晰的.但是我个人其实更加喜欢kotlin中直接通过功能去规定类型安全的类型上下界.

```
public <T> void copy(List<? super T> dest, List<? extends T> src) {
  ...
}
```

List<? super T> dest 是消费者,数据将会放到这里.而List<? extends T> src是生产者,数据从这里提供.

实际上看copy这个java方法,在定义的时候需要稍微思考一下才能确定哪个用super哪个用extends.但是如果你用kotlin的话想都不用想,消费者用in,生产者用out:

```
fun <T> copy(dest: Array<in T>, src: Array<out T>) {
  ...
}
```

直接用in/out这种功能描述是不是比java用super/extends这种类型描述直接了很多?

泛型这部分参考了《Kotlin极简教程》,这本书真的不错,强烈推荐.而大家如果对java泛型有兴趣的话可以去可空《Effect java》的相关章节或者也可以去看看我之前写的两篇文章 [《java泛型那些事》](http://blog.islinjw.cn/2018/01/06/java%E6%B3%9B%E5%9E%8B%E9%82%A3%E4%BA%9B%E4%BA%8B/)、[《再谈Java泛型》](http://blog.islinjw.cn/2018/02/04/%E5%86%8D%E8%B0%88Java%E6%B3%9B%E5%9E%8B/)

# 无缝调用java

当然,一个语言就算做的再好,但是没有一个健全的生态的话是很难被大众接受的.kotlin能够火起来,甚至被谷歌爸爸钦定为安卓的官方推荐语言.一个很重要的原因就是它可以无缝与java相互调用.以前写的java代码不用任何处理就能直接在kotlin中使用,而java也能无缝调用kotlin代码.

实际上在安卓中,编译的时候kotlin代码就会被编译成java代码,所以它们其实是等价的.
