title: 一起入门gradle自定义插件编写(二) - 深入理解build.gradle
date: 2019-04-26 20:37:42
tags:
    - 技术相关
    - Android
---

这篇博客我们来通过groovy的语法去深入理解build.gradle的底层实现。

通过分析build.gradle里面的实现原理,我们在写自己的自定义gradle插件的时候就能使用同样的配置方法了。

在上一篇[博客](http://blog.islinjw.cn/2019/04/14/%E4%B8%80%E8%B5%B7%E5%85%A5%E9%97%A8gradle%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8F%92%E4%BB%B6%E7%BC%96%E5%86%99%E4%B8%80/)里面提到,在gradle文件里面默认使用的都是project这个对象的方法或者属性,并且分析了apply方法的完整形式:

```
project.apply(['plugin': 'com.android.application'])
```

其实android,和dependencies代码块也是一样的,省略了project对象,添加上之后变成这样:

```
project.android {
    compileSdkVersion 28
    defaultConfig {
        applicationId "me.linjw.demo"
        minSdkVersion 24
        targetSdkVersion 28
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}

project.dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.android.support:appcompat-v7:28.0.0'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:1.0.2'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.2'
}
```

我们先讲dependencies,按住ctrl键用鼠标点击它可以跳转到到Project接口的void dependencies(Closure configureClosure)方法

也就是说它其实是project的一个方法,传入一个Closure对象作为参数.然后这里是省略了方法的括号,它的完整形式如下:

```
project.dependencies({
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.android.support:appcompat-v7:28.0.0'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:1.0.2'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.2'
})
```

# groovy闭包

这个Closure就是闭包的意思,闭包作为脚本语言里面比较常见的东西我就不过多介绍了,感兴趣的同学可以自行搜索.

groovy里的闭包就是用话括号来定义的,可以看看下面闭包的例子:

```
//定义闭包并且把它赋值给closure变量
def closure = {
    println('hello world!')
}

//调用闭包
closure()
```

这里的closure()会调用闭包的方法,打印出"hello world!"

这里的闭包也是一个省略的写法,它的完整写法如下:

```
def closure = {->
    println('hello world!')
}
```

"->"左边是闭包的输入参数,由于这里不需要输入参数,所以它左边没有东西.我们可以看看下面的例子,这个闭包接收两个参数:

```
def closure = { str1, str2 ->
    println(str1 + ' ' + str2)
}

closure('hello', 'world')
```

特殊的,如果闭包只接收一个参数,也可以省略参数名和"->",它会默认包含一个隐式的参数it:

```
def closure = {
    println(it)
}

closure('hello world!') // 打印hello world!
closure() // 打印null
```

可以看到,如果只有一个参数的话在调用闭包的时候可以不传参数,它会默认传入null.

# delegate

dependencies方法传入的闭包里面的implementation其实也是调用的方法,我们补全它们的括号

```
project.dependencies({
    implementation(fileTree(dir: 'libs', include: ['*.jar']))
    implementation('com.android.support:appcompat-v7:28.0.0')
    testImplementation('junit:junit:4.12')
    androidTestImplementation('com.android.support.test:runner:1.0.2')
    androidTestImplementation('com.android.support.test.espresso:espresso-core:3.0.2')
})
```

那这个implementation方法又是哪里来的呢?是groovy闭包自带的方法?还是全局的方法?

其实都不是,这里我们先从闭包的delegate说起,顾名思义它是闭包的一个委托对象,闭包中没有的方法都会调到它那里去.

我们来看下面的例子,在闭包中调用foo()方法,调用的时候会报错,因为找不到foo()方法:

```
def closure = {
    foo()
}
closure() // 报错,找不到foo()方法
```

如果我们定义一个类,里面实现foo方法,然后将这个类设置成闭包的delegate,则在闭包中找不到foo()方法的时候就会去它的代理中找:

```
class TestClass {
    def foo() {
        println('foo')
    }
}
def closure = {
    foo()
    println(delegate)
}
closure.delegate = new TestClass()
closure() // 先在TestClass.foo方法中打印'foo',然后打印闭包的delegate对象'TestClass@755e1c30'
```

这个时候让我们看看dependencies闭包的delegate:

```
project.dependencies({
    println(delegate)
    implementation(fileTree(dir: 'libs', include: ['*.jar']))
    implementation('com.android.support:appcompat-v7:28.0.0')
    testImplementation('junit:junit:4.12')
    androidTestImplementation('com.android.support.test:runner:1.0.2')
    androidTestImplementation('com.android.support.test.espresso:espresso-core:3.0.2')
})
```

输出为

> org.gradle.api.internal.artifacts.dsl.dependencies.DefaultDependencyHandler_Decorated@ee11179

这个DefaultDependencyHandler_Decorated东西我们不用细究,只要知道它是DefaultDependencyHandler的子类就行

# dependencies的原理

我们都知道当我们在配置了productFlavors的时候,可以为每个product单独配置依赖库

```
android {
	...
	productFlavors {
	    demo {
	    }
	}
}

dependencies {
	...
	demoImplementation 'com.google.code.gson:gson:2.6.2'
}
```

但是这个demoImplementation方法又是怎么生成的呢?

其实最后它们都是调用到了DefaultDependencyHandler.add方法,也就是说我们的dependencies其实实际的形式是这样的:

```
project.dependencies({
    add('implementation', fileTree(dir: 'libs', include: ['*.jar']))
    add('implementation', 'com.android.support:appcompat-v7:28.0.0')
    add('testImplementation', 'junit:junit:4.12')
    add('androidTestImplementation', 'com.android.support.test:runner:1.0.2')
    add('androidTestImplementation', 'com.android.support.test.espresso:espresso-core:3.0.2')
    add('demoImplementation', 'com.google.code.gson:gson:2.6.2')
})
```

这个add方法是怎么调用到的呢?groovy里面可以有几种方法做到,这里就讲一种:


```
class Delegate {
    def invokeMethod(String name, args) {
        println('method : ' + name)
        println('args : ' + args)
    }
}

def closure = {
    demoImplementation 'com.google.code.gson:gson:2.6.2'
}

closure.delegate = new Delegate()
closure()
```

上面的例子,我们在闭包中调用了delegate中也没有的方法demoImplementation,这个时候会调用delegate的invokeMethod,打印如下:

```
method : demoImplementation
args : [com.google.code.gson:gson:2.6.2]
```

所以这个时候我们就可以在这个invokeMethod方法里面给每个product配置依赖了。


# Extension

与project.dependencies不同project.android,project里面并没有一个方法叫做android。

那这个project.android方法是怎么调用的呢?它是通过project的一个Extension,也就是project的一个拓展。

这个拓展是怎么来的呢?可以看看下面的代码:

```
class MyAndroid {
    def compileSdkVersion;

    def compileSdkVersion(compileSdkVersion) {
        this.compileSdkVersion = compileSdkVersion
    }
}

project.extensions.add('myAndroid', new MyAndroid())

project.myAndroid {
    compileSdkVersion 28
}
```

我们只需要使用project.extensions.add方法加入一个名字叫做myAndroid的Extension,gradle就会为我们在project里面添加一个名字叫做myAndroid的方法,接收一个闭包,然后在这个方法里面会将传入的闭包的delegate设置成我们new出来的MyAndroid对象。

## metaClass

这个Extension又是怎么实现的呢？

其实脚本语言一般都支持动态添加方法和属性,groovy同样也支持。

我们在groovy中可以使用metaClass进行运行是元编程,动态创建类、方法等


例如,下面代码中我们给Demo类动态添加了hello属性和sayHello方法:

```
class Demo {

}

Demo.metaClass."hello" = "hello world"
Demo.metaClass."sayHello" = { println("hello world") }

Demo demo = new Demo()
demo.sayHello()
println(demo.hello)
```

甚至当重名的时候它还会根据我们设置的是值还是闭包帮我们分别创建属性和方法:

```
class Demo {

}

Demo.metaClass."hello" = "hello world"
Demo.metaClass."hello" = { println("hello world") }

Demo demo = new Demo()
demo.hello()
println(demo.hello)
```

有了这个元编程的技术,要实现Extension就简单了:


```
def addExtensions(String name, Object handler) {
    project.metaClass."$name" = { it ->
        it.delegate = handler
        it()
    }
    project.metaClass."$name" = handler
}

class MyAndroid {
    def compileSdkVersion;

    def compileSdkVersion(compileSdkVersion) {
        this.compileSdkVersion = compileSdkVersion
    }
}

addExtensions('myAndroid', new MyAndroid())

project.myAndroid {
    compileSdkVersion 28
}

println(project.myAndroid.compileSdkVersion)
```
