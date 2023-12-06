title: mockk使用入门
date: 2023-12-06 19:35:25
tags:
    - 技术相关
    - Android
    - 单元测试
---

为了方便讲解我们写了一个小工具,支持把java的链式调用代码入去执行,它的核心调用逻辑如下:

```kotlin
invoker = Invoker()
invoker.addPrefix("context.", Invoker.ClassInstance(Context::class.java, context))
val path = invoker.invoke("context.getFilesDir().getAbsolutePath()") // 返回执行`context.getFilesDir().getAbsolutePath()`代码后的结果
```

假设我们我们实现上面三行代码的功能,可以先写一个最简单的解析调用空参数列表方法的Invoker:

```kotlin

class Invoker {
    private val prefixes = mutableMapOf<String, ClassInstance>()

    fun addPrefix(prefix: String, classInstance: ClassInstance) {
        prefixes[prefix] = classInstance
    }

    fun invoke(code: String): Any? {
        val matches = prefixes.entries.find { code.startsWith(it.key) }
            ?: throw Exception("can't match prefix for $code")

        Log.d(TAG, "invoke $code")

        val parts = code
            .substring(matches.key.length) // 删除前缀,例如 "context.getFilesDir().getAbsolutePath()" 删除 "context." 之后剩下 "getFilesDir().getAbsolutePath()"
            .split(".")         // 使用 "." 分割链式方法调用,例如 "getFilesDir().getAbsolutePath()" 分割出 ["getFilesDir()", "getAbsolutePath()"]
            .toList()
        return invoke(parts, 0, matches.value)
    }

    private fun invoke(codes: List<String>, curIndex: Int, instance: ClassInstance): Any? {
        if (curIndex >= codes.size) {
            return instance.instance
        }
        val code = codes[curIndex]
        val (methodName, params) = code
            .substring(0, code.length - 1)    // 删除方法调用的右花括号,例如 "getFilesDir()" 得到 "getFilesDir("
            .split("(")            // 使用方法调用的左花括号进行分割方法名和参数列表,例如 "getFilesDir(" 得到 ["getFilesDir, ""]

        instance.clazz.methods
            .filter { it.name == methodName } // 遍历类的所有方法,找到 getFilesDir 这个名字的方法
            .forEach { method ->
                // 目前只先支持空参数列表的方法调用
                if (method.parameterTypes.isEmpty()) {
                    // 反射调用 context.getFilesDir() 得到ret
                    val ret = ClassInstance(method.returnType, method.invoke(instance.instance))
                    // 将ret传入下一层去执行 "getAbsolutePath()"
                    return invoke(codes, curIndex + 1, ret)
                }
            }

        throw Exception("no match method for $code in ${instance.clazz}")
    }

    data class ClassInstance(
        val clazz: Class<*>,
        val instance: Any?,
    )
}
```

代码写完之后需要如果确认功能呢?是加个打印编译运行到真机或者模拟器上看看打印是否如预期?

但是这么做的话会有下面的问题:

1. 编译运行查看打印的耗时会比较久
2. 每次修改bug或者新增功能(例如添加方法参数支持),可能会引入bug导致前面已经测试通过的功能出现问题
3. 后面接手这个项目的人没有办法确认目前已经有哪些调用方式是已经支持的

解决这些问题最好的方式就是使用单元测试。

假设我们使用单元测试去测上面的三行代码,就会遇到一个问题:context如何获取?有两种方式:

一是使用androidTest在整机或者模拟器里面运行单元测试然后使用"InstrumentationRegistry.getInstrumentation().targetContext"获取。
二是使用mock技术mock出一个假的context在电脑上执行单元测试。

这里我们只讲第二种。

[mock技术](https://zh.wikipedia.org/wiki/%E6%A8%A1%E6%8B%9F%E5%AF%B9%E8%B1%A1)简单来讲就是创建一个可以控制方法返回值的假对象,用于传入需要测试的方法,去测试其代码逻辑。java上可以使用[PowerMock](https://github.com/powermock/powermock)、[mockito](https://github.com/mockito/mockito)而kotlin则使用[mockk](https://github.com/mockk/mockk),java的话之前早年间写过一篇[博客](https://blog.islinjw.cn/2017/08/26/%E7%94%A8PowerMock%E8%BF%9B%E8%A1%8CAndroid%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E4%B8%8EBDD%E8%A1%8C%E4%B8%BA%E9%A9%B1%E5%8A%A8%E5%BC%80%E5%8F%91/),这里说下mockk。


实际上mockk的[官方文档](https://mockk.io/)已经蛮详细的了,但是缺少了点安卓上场景化的使用方式,我这边就用一个实际的例子去介绍。

# mockk

导入mockk的方式很简单:

> testImplementation "io.mockk:mockk:1.12.0"

然后就可以开始测试了:

```kotlin
class InvokerTest {
    private lateinit var invoker: Invoker

    // 使用注解的方式声明需要mock的对象
    @MockK
    private lateinit var context: Context

    // 调用每个@Test测试用例前会调用@Before方法做初始化
    @Before
    fun setUp() {
        // 遍历this的所有@MockK成员变量,为他们创建实例
        MockKAnnotations.init(this)

        // 创建我们需要测试的对象
        invoker = Invoker()

        // 将我们mock出来的context传入Invoker使用
        invoker.addPrefix("context.", Invoker.ClassInstance(Context::class.java, context))

        // mockk支持mock静态方法
        mockkStatic(Log::class)

        // 调用Log.d传入任意的参数都返回0
        every { Log.d(any(), any()) } returns 0
    }

    // 调用每个@Test测试用例后会调用@After方法做清理动作
    @After
    fun cleanUp() {
        // 解除静态方法的mock
        unmockkStatic(Log::class)
    }

    @Test
    fun testNoParamFun() {
        // 配置调用context.getFilesDir()返回File("/data/user/0/com.cvte.udi.proxy/files")
        every { context.filesDir } returns File("/data/user/0/com.cvte.udi.proxy/files")

        // 实际调用我们需要测试的方法
        val path = invoker.invoke("context.getFilesDir().getAbsolutePath()")

        // 校验测试方法的返回值是否如预期
        assertEquals("/data/user/0/com.cvte.udi.proxy/files", path)
    }
}
```

除了使用注解"@MockK"注解之外,我们也可以用mockk方法去创建mock对象:

```kotlin
context = mockk()
```

## mock静态方法

Invoker.invoke里面调用到了Log.d,而它的具体实现在framework.jar里面,如果不运行在安卓环境,直接在电脑上跑单元测试执行到会报下面的问题:

```shell
Method d in android.util.Log not mocked. See http://g.co/androidstudio/not-mocked for details.
java.lang.RuntimeException: Method d in android.util.Log not mocked. See http://g.co/androidstudio/not-mocked for details.
    at android.util.Log.d(Log.java)
```

为了解决这个问题我们可以直接mock Log.d,或者在build.gradle里面添加配置:

```groovy
android {
    ...
    testOptions {
        unitTests.returnDefaultValues = true
    }
}
```

或者如这里的例子用mockkStatic去mock Log,这样调用到Log.d的时候就会执行我们mock出来的Log的d静态方法:

```kotlin
@Before
fun setUp() {
    ...
    // mockk支持mock静态方法
    mockkStatic(Log::class)

    // 调用Log.d传入任意的参数都返回0
    every { Log.d(any(), any()) } returns 0
}

@After
fun cleanUp() {
    // 解除静态方法的mock
    unmockkStatic(Log::class)
}
```

PS: kotlin里面更多的是使用object,可以使用`mockkObject`和`unmockkObject`去mock object

## 方法调用次数

有时候会需要确认mock对象方法被调用的次数,可以使用verify方法去校验:

```kotlin
@Test
fun testInvokeTime() {
    // 配置context.getApplicationContext()返回context
    every { context.applicationContext } returns context

    // 执行测试用例
    invoker.invoke("context.getApplicationContext().getApplicationContext().getApplicationContext()")

    // 校验Context.getApplicationContext()被调用了3次
    verify(exactly = 3) { context.applicationContext }
}
```

可以用下面的参数去校验方法调用次数:
- exactly : 具体的被调用次数
- atLeast : 最少被调用次数
- atMost : 最多被调用次数
- inverse : 为true表示方法没有被执行过, 相当于exactly=0


## 参数校验

有时候我们会需要校验传入mock对象方法的参数,可以用MockKMatcherScope的eq、any这些方法去匹配参数,也可以直接把具体的参数值填入去匹配相等的参数:

```kotlin
@Test
fun testTowParam() {
    every { context.getDir(eq("dir1"), any()) } returns File("dir1")
    every { context.getDir(eq("dir2"), any()) } returns File("dir2")

    val dir1 = proxy.invoke("context.getDir(\"dir1\", 123).getName()") as String
    val dir2 = proxy.invoke("context.getDir(\"dir2\", 456).getName()") as String

    assertEquals("dir1", dir1)
    assertEquals("dir2", dir2)

    verify(exactly = 1) { context.getDir("dir1", 123) }
    verify(exactly = 1) { context.getDir("dir2", 456) }
}
```

除了上面这样两条verify语句去校验,我们也可以用下面的方式校验多条调用:

```kotlin
@Test
fun testTowParam() {
    every { context.getDir(eq("dir1"), any()) } returns File("dir1")
    every { context.getDir(eq("dir2"), any()) } returns File("dir2")

    val dir1 = proxy.invoke("context.getDir(\"dir1\", 123).getName()") as String
    val dir2 = proxy.invoke("context.getDir(\"dir2\", 456).getName()") as String

//  verifyAll{ // 无视顺序,只要context.getDir的所有调用都在里面即可
//  verifySequence { // context.getDir的所有调用都在里面,且必须按顺序执行
    verifyOrder { // 只要下面的两条调用是按顺序执行的就行,中间或者前后可以插入其他参数调用
        context.getDir("dir1", 123)
        context.getDir("dir2", 456)
    }
}
```

## 参数捕获

有时候我们会需要捕获传给mock对象方法的参数,例如拿到传入的callback然后主动调用callback,又例如拿到传给线程池或者handler的Runnable去直接run。

或者参数的方式有两种:

1. 设置answer方法,调用到mock对象方法的时候会转发给到设置的answer方法,可以在里面进行保存
2. 使用capture机制去获取参数

```kotlin
 @Test
fun testInterfaceParam() {
    // 设置Log.d的answer处理函数,用于获取传给Log.d的参数
    var log: String? = null
    every { Log.d(any(), any()) } answers {
        log = it.invocation.args[1] as String
        0
    }

    // 使用slot去获取传给Context.registerComponentCallbacks的参数
    val slot = slot<ComponentCallbacks>()
    every { context.registerComponentCallbacks(capture(slot)) } returns Unit

    proxy.invoke("context.registerComponentCallbacks(new Proxy())")

    verify(exactly = 1) { context.registerComponentCallbacks(any()) }

    // 调用Context.registerComponentCallbacks设置的callback
    slot.captured.onLowMemory()

    // "new Proxy()"创建的代理里面会调用Log.d去打印,对比打印的值和预期值是否一致
    assertEquals("callback --> ComponentCallbacks.onLowMemory()", log)
}
```

capture除了slot捕获最后一次传入的参数之外也可以传入MutableList捕获多次传入的参数:

```kotlin
@Test
fun testSleep() {
    // 传入MutableList去捕获多次传入Log.d的参数
    val params = mutableListOf<String>()
    every { Log.d(any(), capture(params)) } returns 0

    proxy.addPrefix("Executors.", Invoker.ClassInstance(Executors::class.java, null))
    proxy.invoke("Executors.newScheduledThreadPool(1).schedule(new Proxy(), 1, SECONDS)")

    // 等待Log.d被执行两次,超时时间为2s
    verify(exactly = 2, timeout = 2000) { Log.d(any(), any()) }

    assertEquals("invoke Executors.newScheduledThreadPool(1).schedule(new Proxy(), 1, SECONDS)", params[0])
    assertEquals("callback --> Runnable.run()", params[1])
}
```

## mock构造函数

类似Handler很多情况下是在类内部直接new出来的:

```kotlin
class MyClass {
    private val handler = Handler(Looper.getMainLooper())

    fun post(r: Runnable) {
        handler.post(r)
    }
}
```

如果我们想捕获传给Handler.post的Runnable去主动run,就需要mock在类内部new出来的的Handler。这种情况就可以使用mock类构造函数的方式去实现了:

```kotlin
@Test
fun testMockConstructed() {
    // mock Looper.getMainLooper
    mockkStatic(Looper::class)
    every { Looper.getMainLooper() } returns null

    // mock Handler的构造函数
    mockkConstructor(Handler::class)
    every { anyConstructed<Handler>().post(any()) } returns true

    val r = Runnable { }
    val myClass = MyClass()
    myClass.post(r)

    // 验证MyClass.post内部有调用Handler.post
    verify(exactly = 1) { anyConstructed<Handler>().post(r) }

    // 取消Looper和Handler的mock
    unmockkStatic(Looper::class)
    unmockkConstructor(Handler::class)
}
```

# 单元测试的作用

上面的几个技巧已经足够我们使用mockk去编写测试用例了,其他更完整的用法可以直接看官方[文档](https://mockk.io/)

1. 脱离复杂的运行环境检测代码逻辑 - 有些功能依赖了比较复杂的外部输入,比方说http请求的返回,可以直接模拟出返回数据进行代码逻辑的验证

2. 监控所有功能的可用性 - 对各个功能编写测试用例,一旦修改bug出现bug就能立马发现

3. 列举所有的可用功能 - 用测试用例列举所有可用的功能和调用方式

4. 可测试性越高的代码,可维护性也会越高 - 如果发现你写的代码不知道怎么写测试用例,或者写测试用例需要mock一堆乱七八糟的构造函数、私有方法就代表可能代码的结构就有问题,可维护性不行,起码代码的解耦没有做好

5. 监控出现过的bug - 将出现过的bug写成测试用例,确保以后修改代码再次出现可以立马发现