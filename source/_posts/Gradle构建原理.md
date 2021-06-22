title: Gradle构建原理
date: 2021-06-22 22:26:55
tags:
    - 技术相关
    - Android
---


# Gradle的构建的三个阶段

根据Gradle的[官方文档](https://docs.gradle.org/current/userguide/build_lifecycle.html),Gradle的构建分成三个阶段:

- **Initialization** (初始化)

Gradle允许multi-project，也就是android studio里面的项目+模块的形式。安卓项目被称为Root Project，而每个模块其实都是一个子project。Root Project肯定是要参与编译的，但Gradle是怎么知道哪些子Project需要参与编译呢?其实它就是在Initialization阶段执行settings.gradle脚本得到的，这个脚本会将子Project给include进来。

- **Configuration** (配置)

每个Project都与一个build.gradle文件一一对应，build.gradle可以说就是Project的配置脚本。在配置阶段Gradle会执行所有参与编译的Project的build.gradle脚本，这个脚本对Project进行配置并创建一系列Task，这些Task之间会有依赖关系，构成一个有向无环图。

- **Execution** (执行)

执行阶段我们可以选择一个或者多个Task去执行。被执行到的Task可能会依赖其他的Task，Gradle保证这些Task按照依赖关系的顺序执行，并且每个任务只执行一次。

我们可以通过添加打印查看settings.gradle和build.gradle被执行的时机:

```groovy
//settings.gradle
include ':app'
rootProject.name='My Application'

println("---> settings.gradle")
```

```groovy
// 根目录的build.gradle
println("---> build.gradle for Root Project")

buildscript {
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.1'
    }
}
...
```

```groovy
// app下的build.gradle
println("---> build.gradle for app Project")

apply plugin: 'com.android.application'

android {
    ...
}
...
```

然后执行.gradlew build查看打印:

```shell
./gradlew build
---> settings.gradle

> Configure project :
---> build.gradle for Root Project

> Configure project :app
---> build.gradle for app Project

> Task :app:lint
...
```

# Project的配置

上面我们讲到每个Project都与一个build.gradle文件一一对应，每个Project都有一个Project实例，而build.gradle实际上调用的是这个实例的方法。这个实例名字是project，可以被省略，也就是说build.gradle的完整形态应该是这样的:

```groovy
project.apply([plugin: 'com.android.application'])

project.android({
    compileSdkVersion 30
    buildToolsVersion "30.0.2"
    defaultConfig {
        ...
    }
    buildTypes {
        ...
    }
})

project.dependencies({
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'androidx.appcompat:appcompat:1.3.0'
    implementation 'androidx.constraintlayout:constraintlayout:1.1.3'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'androidx.test.ext:junit:1.1.2'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.3.0'
})
```

所以build.gradle配置Project的本质其实就是对project对象调用了一系列的方法。

# Task

Task表示构建的单个原子工作，例如编译类或生成javadoc，Project本质上是Task对象的集合。

我们可以通过Project的task方法创建一个Task:

```groovy
// app build.gradle
def ADemoTask = project.task("ADemoTask")
```

这样创建的Task默认分组是other:

{% img /Gradle构建原理/1.jpeg %}

我们可以在创建的时候给它指定分组:

```groovy
def ADemoTask = project.task("ADemoTask", group: "build")
```

{% img /Gradle构建原理/2.jpeg %}

Task是由一个Action列表组成的，Action实际上是一个*闭包函数*。我们可以用doFirst往列表头插入Action，也能用doLast往列表尾插入Action:

```groovy
def ADemoTask = project.task("ADemoTask", group: "build")
ADemoTask.doFirst({ println("1") })
ADemoTask.doFirst({ println("2") })
ADemoTask.doLast({ println("3") })
ADemoTask.doLast({ println("4") })
//基于groovy语法我们也可以省略圆括号，写成下面的样子:
//ADemoTask.doFirst {println("1")}
//ADemoTask.doFirst {println("2")}
//ADemoTask.doLast {println("3")}
//ADemoTask.doLast {println("4")}
```

执行这个Task，最终得到的打印是这样的:

```
> Task :app:ADemoTask
2
1
3
4
```

## Task DSL定义原理

由于task方法支持直接传入一个闭包函数进行配置，所以我们可以写成下面的方式:

````groovy
project.task("ADemoTask",
        {
            //下面的方法调用都省略了圆括号
            it.doFirst{ println("1") }
            it.doFirst{ println("2") }
            it.doLast{ println("3") }
            it.doLast{ println("4") }
        }
)
````

也就是说在Configuration阶段，build.gradle脚本会调用project.task方法，将Task的配置以闭包函数的形式传入，task方法内部会去执行这个闭包。

基于groovy语法，我们可以将这个闭包移出圆括号:

```groovy
project.task("ADemoTask") {
    it.doFirst{ println("1") }
    it.doFirst{ println("2") }
    it.doLast{ println("3") }
    it.doLast{ println("4") }
}
```

为了更加的简洁,我们可以省略project和it对象，然后把task方法函数调用的圆括号也省略:

```groovy
task "ADemoTask" {
    doFirst{ println("1") }
    doFirst{ println("2") }
    doLast{ println("3") }
    doLast{ println("4") }
}
```

Gradle在处理build.gradle的时候会有一个转换，将task identifier arg转换成task "identifier" arg(下面的是Gladle的源码从[知乎](https://zhuanlan.zhihu.com/p/365577252)上看到的):

```java
if (args.getExpression(0) instanceof MapExpression && args.getExpression(1) instanceof VariableExpression) {
    // Matches: task <name-value-pairs>, <identifier>, <arg>?
    // Map to: task(<name-value-pairs>, '<identifier>', <arg>?)
    transformVariableExpression(call, 1);
} else if (args.getExpression(0) instanceof VariableExpression) {
    // Matches: task <identifier>, <arg>?
    transformVariableExpression(call, 0);
}
```

也就是说我们的task名称的双引号也是可以省略的，虽然这不属于groovy的语法糖，但在构建的时候Gradle会对我们的代码进行转换。于是就变成了我们常见的task定义方式:

```groovy
task ADemoTask {
    println("ADemoTask...")
    doFirst{ println("1") }
    doFirst{ println("2") }
    doLast{ println("3") }
    doLast{ println("4") }
}
```

所以ADemoTask后方的闭包内的代码(如println)在配置阶段执行，而doFirst/doLast后方的闭包实际是Action，会在执行阶段执行。

## Task的依赖关系

如同第一节所说Task之间是会有依赖关系的:

```groovy

task TaskA {
    doLast {
        println("TaskA run...")
    }
}

task TaskB {
    doLast {
        println("TaskB run...")
    }
}


task TaskC {
    doLast {
        println("TaskC run...")
    }
}

TaskC.dependsOn TaskA
TaskC.dependsOn TaskB
//上面的代码实际是下面代码的省略形式
//project.TaskC.dependsOn(project.TaskA)
//project.TaskC.dependsOn(project.TaskB)
```

task方法为project对象拓展了Task成员变量，我们可以调用Task的dependsOn方法给它添加依赖，Gradle在执行一个Task之前，会先执行它所依赖的Task:

```
./gradlew TaskC

> Task :app:TaskA
TaskA run...

> Task :app:TaskB
TaskB run...

> Task :app:TaskC
TaskC run...
```

## 增量编译

编译一般情况下是通过一些输入的文件来执行编译动作，然后输出另外的文件。在输入文件没有改变的情况下，实际上不需要每次都执行边编译:

```groovy
task ADemoTask {
    def srcFile = new File("src.txt")
    def destFile = new File("dest.txt")

    inputs.file(srcFile)
    outputs.file(destFile)

    doLast {
        println("run...")
        destFile.delete()
        srcFile.eachLine {line ->
            destFile.append("${line}\n")
        }
    }
}
```

像上面的例子我们指定了task的inputs和outputs，只要outputs已经被生成，且inputs没有修改，那么doLast加入的Action就不会被执行。

# Project、Task、Action间的关系

经过上面的讲解我们大概可以理解Project、Task、Action之间的关系大概如下图:

{% img /Gradle构建原理/3.png %}

Project间存在依赖，每个Project包含多个存在依赖的Task，Task内部有一个Action列表，Task的执行实际上就是内部Action的顺序执行。



## 插件

让我们回到build.gradle脚本，里面默认调用了project的apply、android、dependencies方法:

```groovy
project.apply([plugin: 'com.android.application'])

project.android({
    compileSdkVersion 30
    buildToolsVersion "30.0.2"
    defaultConfig {
        ...
    }
    buildTypes {
        ...
    }
})

project.dependencies({
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'androidx.appcompat:appcompat:1.3.0'
    implementation 'androidx.constraintlayout:constraintlayout:1.1.3'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'androidx.test.ext:junit:1.1.2'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.3.0'
})
```

dependencies方法配置了Project的依赖，Project可以依赖编译好的库，也能依赖其他的Project(android studio里面的表现是module间的依赖)。

android方法配置了一些安卓相关的配置项，但是实际上Gradle并不是专门给安卓使用的，为什么Project里面会有android这个方法呢?

答案就在apply方法传入的插件。

Gradle执行com.android.application这个插件的时候会将Project的对象传给插件，插件内部就能为这个Project新增一些拓展方法和Task。我们熟悉的build、assemble、assembleDebug这些Task都是在这个插件里面定义的。

插件的原理我就不细讲了，刚兴趣的同学可以看[这篇笔记](https://blog.islinjw.cn/2019/04/14/%E4%B8%80%E8%B5%B7%E5%85%A5%E9%97%A8gradle%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8F%92%E4%BB%B6%E7%BC%96%E5%86%99%E4%B8%80/)，这里只简单举个添加拓展方法的例子:

```groovy
// app build.gradle

// 创建自定义类
class MyExt {
    String data1
    String data2

    MyExt() {
        this.data1 = null
        this.data2 = null
    }

    void data1(String data1) {
        this.data1 = data1
    }

    void data2(String data2) {
        this.data2 = data2
    }
}

//实例化对象
MyExt ext = new MyExt()

//为project拓展myExt方法
project.extensions.add("myExt", ext)

// 下面的myExt实际是project.myExt省略project对象，函数调用的圆括号也被省略了
myExt {
    data1("111")
    data2("222")
}

println(myExt.data1)
println(myExt.data2)
```