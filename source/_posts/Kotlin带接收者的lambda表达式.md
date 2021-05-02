title: Kotlin带接收者的lambda表达式
date: 2021-05-02 10:05:49
tags:
    - 技术相关
    - Android
---

在学习kotlin协程原理的时候发现了一个比较有意思的东西:

```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    ...
}
```

这个block参数的类型是:

> suspend CoroutineScope.() -> Unit

suspend是一个关键字，在协程里面用于声明挂起函数，我们先忽略。看后面的**CoroutineScope.() -> Unit**看起来像是个函数类型，但是比起一般的函数类型又多了前面的”**CoroutineScope.**“前缀。

这玩意学名叫做[带有接收者的函数类型](https://www.kotlincn.net/docs/reference/lambdas.html#%E5%B8%A6%E6%9C%89%E6%8E%A5%E6%94%B6%E8%80%85%E7%9A%84%E5%87%BD%E6%95%B0%E5%AD%97%E9%9D%A2%E5%80%BC)，通过文档我们可以知道它和[扩展函数](https://www.kotlincn.net/docs/reference/extensions.html)类似，允许在函数体内部访问接收者对象的成员。也就是说在lambda内部，传给调用的接收者对象成为隐式的this，我们可以用this去访问它的成员变量。这个时候invoke的第一个参数就是接收者也就是this的值:

```kotlin
data class Person(var name: String, var age: Int)

fun with(person: Person, block: Person.() -> R) {
    return block.invoke(person)
}

with(person) {
    println("my name is ${this.name}, ${this.age} years old.")
}
```

然后this也可以省略：

```kotlin
with(person) {
    println("my name is ${name}, ${age} years old.")
}
```

# kotlin with方法的原理

通过kotlin的语法糖我们可以将invoke省略，将with方法写成下面的形式:

```kotlin
fun with(person: Person, block: Person.() -> Unit) {
    block(person)
}

with(person) {
    println("my name is ${name}, ${age} years old.")
}
```

由于block的接收者是Person类型的，通过"Person.()->Unit"这个类型声明，其实我们可以将它看成Person的拓展函数，通过person这个传入的对象去调用，也就是说我们可以将with方法再改成这样的形式:

```kotlin
fun with(person: Person, block: Person.() -> Unit) {
    person.block()
}
```

对kotlin熟悉的同学可能想到了，如果将特定的Person类型改成泛型，然后再将block的返回值返回，就是我们常用的with函数的实现原理:

```kotlin
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    ...
    return receiver.block()
}
```

# kotlin apply方法的原理

如果with方法是Person的成员方法，那么输入的person接收者就是this，可以不用特地在方法参数指定。也就是说可以把它去掉写成下面的形式:

```kotlin
data class Person(var name: String, var age: Int) {
    fun with(block: Person.() -> Unit) {
        this.block()
    }
}

with(person) {
    println("my name is ${name}, ${age} years old")
}
```

这个时候就更像给Person类创建了一个拓展方法。接收者是this，这个this就是我们传入的person变量。然后this同样可以省略，写成下面的形式:

```kotlin
data class Person(var name: String, var age: Int) {
    fun with(block: Person.() -> Unit) {
        block()
    }
}

with(person) {
    println("my name is ${name}, ${age} years old")
}
```

如果我们使用泛型去替代这个特定的Person类，然后将this再返回，就是apply方法的实现原理了:

```kotlin
public inline fun <T> T.apply(block: T.() -> Unit): T {
    ...
    block()
    return this
}
```

# anko原理

很多的kotlin DSL就是用上面的带有接收者的lambda函数去实现的，例如anko。我们可以先给ViewGroup声明一个拓展方法textView用于创建TextView添加成子view，然后TextView作为接收者去调用lambda方法block:

```kotlin
fun ViewGroup.textView(block: TextView.() -> Unit) {
    val textView = TextView(context)
    addView(textView)
    block(textView)
}
```

然后再声明一个Activity的拓展方法verticalLinearLayout用于创建垂直布局的LinearLayout，然后将这个创建的LinearLayout作为接收者去调用lambda方法block:

```kotlin
fun Activity.verticalLinearLayout(block: LinearLayout.() -> Unit): LinearLayout {
    val layout = LinearLayout(this)
    layout.orientation = LinearLayout.VERTICAL
    block(layout)
    return layout
}
```

所以就有了下面这样的写法，创建一个垂直布局的LinearLayout里面包含两个TextView:

```
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val content = verticalLinearLayout {
            textView {
                text = "aaa"
                setBackgroundColor(Color.RED)
            }
            textView {
                text = "bbb"
                setBackgroundColor(Color.GREEN)
            }
        }
        
        setContentView(content)
    }
}
```

我们对这个dsl进行详细的解析，第一步它调用了Activity的拓展方法verticalLinearLayout，它的接收者是Activity，内部创建了一个LinearLayout返回。并且用它做接收者调用lambda函数，在lambda函数内部调用的textView方法是ViewGroup的拓展函数，它内部创建了一个TextView加入到LinearLayout的子view。并用这个TextView做接收者调再用lambda函数，所以可以直接设置TextView的setText方法和setBackgroundColor方法。

# kotlin DSL设计

除了上面kotlin内置的with、apply方法和anko的实现之外，[官方文档](https://www.kotlincn.net/docs/reference/lambdas.html#%E5%B8%A6%E6%9C%89%E6%8E%A5%E6%94%B6%E8%80%85%E7%9A%84%E5%87%BD%E6%95%B0%E5%AD%97%E9%9D%A2%E5%80%BC)也提供了一个比较典型的例子:

```kotlin
class HTML {
    fun body() { …… }
}

fun html(init: HTML.() -> Unit): HTML {
    val html = HTML()  // 创建接收者对象
    html.init()        // 将该接收者对象传给该 lambda
    return html
}

html {       // 带接收者的 lambda 由此开始
    body()   // 调用该接收者对象的一个方法
}
```

html方法接收一个带HTML类型接收者的lambda函数，它在内部自己创建一个HTML的实例去调用这个lambda函数，于是可以在这个lambda函数里面调用HTML的body方法。

通过上面的几个例子，我们已经了解了带接收者的lambda函数的使用方法，我们可以通过这些典型的用法去设计我们自己的DSL使得我们设计的框架更加易用。

