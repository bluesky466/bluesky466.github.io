title: 一起入门gradle自定义插件编写(一)
date: 2019-04-14 17:18:23
tags:
    - 技术相关
    - Android
---


相信现在的安卓程序员对gradle脚本的配置应该都或多或少有些了解,例如applicationId、version、混淆等的基本配置应该都是比较熟悉的了,像比较高级的自定义buildTypes、productFlavors可能也多多少少了解一些.

但是对于groovy语言和如何自定义gradle插件,相信很多同学还是比较陌生的.

作为一个有理想的安卓程序员,我觉得这种高阶的技能还是需要懂的.像一些热更新、插件化等高级技能都会涉及到groovy代码的编写甚至自定义gradle插件.

# project.apply方法

我们新建一个Android Studio项目,得到两个build.gradle文件,一个是项目根目录下的,一个是模块目录(如app目录)下的.我们只看模块目录下的:

```
apply plugin: 'com.android.application'

android {
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

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.android.support:appcompat-v7:28.0.0'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:1.0.2'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.2'
}
```

这里的第一行代码指定了com.android.application这个插件的使用,这个插件用来构建apk项目.

```
apply plugin: 'com.android.application'
```

另外比较常见的插件有用于构建aar包的com.android.library插件

```
apply plugin: 'com.android.library'
```

和用于构建jar包的java-library插件

```
apply plugin: 'java-library'
```

我们都知道build.gradle使用的是groovy语法,那这个使用插件的代码的语法含义又是怎样的呢?让我们一起来看看.

第一个知识点是我们在gradle文件里面默认使用的都是project这个对象的方法或者属性,也就是说我们的插件配置代码等价于:

```
project.apply plugin: 'com.android.application'
```

## groovy基本语法

接下来我们就要开始学一些groovy的基本语法了.

我们可以像一般的强类型语言一样去定义方法,也可以选择像一些动态语言一样忽略参数和返回值类型:

```
int foo(int x, int y) {
    return x + y
}

def foo2(def x, def y) {
    return x + y
}
```

在调用方法的时候为了简洁,你可以选择省略括号,比如下面的两行代码是效果是一样的,而且我们可以看到,在定义变量的时候也可以选择忽略变量的类型:

```
def x = foo(1,2)
int y = foo 1,2
```

接下来看看groovy中list和map的定义方式:

```
def list = [1, 2, 3, 4]
def map = ['key1': 'val1', key2: 'val2', 3: 'val3', 1.23: 312]
```

可以看到,map很灵活,key/value都可以是任意的类型,然后在key是字符串的时候甚至可以直接省略引号.

甚至,在当作方法参数的时候,我们连map的中括号也是可以省略的,让我们来看看groovy代码是怎样一步步省略到极致的:

```
//下面的四行方法是完全等价的

//不做任何省略
func(['key1': 1, 'key2': 'val2'])

//省略key的双引号
func([key1: 1, key2: 'val2'])

//省略map中括号
func(key1: 1, key2: 'val2')

//省略方法调用的小括号
func key1: 1, key2: 'val2'
```

现在让我们回过头来看这行代码,是不是感觉突然好像有点理解了?

```
apply plugin: 'com.android.application'
```

首先它省略了调用apply的project对象,然后它省略了key的双引号,接着又省略了map里面的中括号,最后还省略了方法调用的小括号

如果不做任何省略的话,它的完整形式应该是:

```
project.apply(['plugin': 'com.android.application'])
```

其实我们也按住ctrl键然后用鼠标点击apply,查看方法的声明:

```
public interface PluginAware {
  ...
  void apply(Map<String, ?> options);
  ...
}
```

可以看到它跳转到了一个java接口里面,这个apply其实是PluginAware这个接口中的一个方法,参数为Map类型.

groovy其实是一种基于jvm的脚本,它可以直接使用java的代码.

所以我们可以选择直接用java编写插件,也可以选择使用groovy语言编写,不过最后groovy也是会被编译器编译成java字节码的.

# 编写自定义gradle代码

在gradle中编写代码有三种方式

最简单的一种是直接在build.gradle文件里面添加我们的代码

第二种是新建一个gradle文件,在里面编写我们的代码,然后用apply from在build.gradle里面导入我们的代码

第三中就是编写我们自己的插件了

第一种方法我们就不说了,直接讲第二种.

## apply from操作

首先我们需要创建一个gradle文件,然后在里面写我们的方法.

例如我在项目根目录下面新建了一个mycode.gradle文件,然后写好代码:

```
def add(def x, def y) {
    return x + y
}
println('=================')
println(add(1, 2))
println('=================')
```

然后在app目录下的build.gradle里面使用apply from操作导入这个文件:

```
apply plugin: 'com.android.application'
apply from: '../mycode.gradle'
```

然后点击build,就可以看到输出了:

```
Executing tasks: [build]

NDK is missing a "platforms" directory.
If you are using NDK, verify the ndk.dir is set to a valid NDK directory.  It is currently set to /home/linjw/android/sdk/ndk-bundle.
If you are not using NDK, unset the NDK variable from ANDROID_NDK_HOME or local.properties to remove this warning.

=================
3
=================
NDK is missing a "platforms" directory.
If you are using NDK, verify the ndk.dir is set to a valid NDK directory.  It is currently set to /home/linjw/android/sdk/ndk-bundle.
If you are not using NDK, unset the NDK variable from ANDROID_NDK_HOME or local.properties to remove this warning.
...
```

当然我们知道apply是一个接收Map的方法,我们不用调用两次apply方法,也可以直接这么写,直接在一次调用中com.android.application插件和mycode.gradle的导入

```
apply plugin: 'com.android.application', from: '../mycode.gradle'
```

## 自定义gradle插件

最高级的方法就是直接编写自定义插件了,编写好的插件可以发布到jcenter或者maven上给人使用.

### 创建Gradle Module

首先我们需要创建一个Gradle Module用于编写gradle插件的代码.但是Android Studio是没有办法直接创建Gradle Module的.

所以我们新建个普通的apk项目,或者新建个Android Library module然后再更改下配置将它改成Gradle Module就好

我这里就直接用新建出来的apk项目了.

第一步是进入app目录,将里面的东西全部都删掉.

#### 1.编写build.gradle

然后新建一个在app目录下新建一个build.gradle文件,写入代码:

```
apply plugin: 'groovy'

dependencies {
    compile gradleApi()
    compile localGroovy()
}
```

#### 2.编写代码

接着在app目录下面新建src目录,然后进入src目录新建main目录,然后再进入main继续新建groovy目录

最后在groovy目录中根据包名新建目录层级,并且新建MyPlugin.groovy文件用于编写我们的插件代码.

我的包名是me.linjw.plugin,所以目录结构如下:

{% img /一起入门gradle自定义插件编写一/1.png %}

插件都需要实现Plugin<Project>接口,然后编写自己代码.代码如下:

```
package me.linjw.plugin

import org.gradle.api.Plugin
import org.gradle.api.Project

public class MyPlugin implements Plugin<Project> {
    def add(def x, def y) {
        return x + y
    }

    void apply(Project project) {
        println("=======MyPlugin========")
        println(add(1, 2))
        println("===============")
    }
}
```

#### 3.注册插件

上面我们已经编写好了我们的插件了,接下来的事情就是告诉gradle哪个是我们的插件类.

main目录下新建resources目录,然后在resources目录里面再新建META-INF目录,再在META-INF里面新建gradle-plugins目录.最后在gradle-plugins目录里面新建properties文件.

这个properties文件的名字就是你插件的名字了,例如之前看到的com.android.application、com.android.library

我这边的名字为me.islinlw.plugin.demo.properties

接着在properties文件里面配置我们的插件类:

implementation-class=me.linjw.plugin.MyPlugin

{% img /一起入门gradle自定义插件编写一/5.png %}

#### 发布插件到本地maven

这个时候其实点击build已经可以在app/build/libs目录下看到我们的插件被编译成app.jar了

但是需要先发布出去别人才能使用,一般可以发布到公司内部或者公网的仓库如jcenter等.我们这边由于是demo,可以先选择发布到电脑的本地仓库.

我们修改下build.gradle:

```
apply plugin: 'groovy'
apply plugin: 'maven'

dependencies {
    compile gradleApi()
    compile localGroovy()
}


repositories {
    mavenCentral()
}

group='me.islinjw.plugin'
version='1.0.0'

uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: uri('/home/linjw/workspace/LocalMaven'))
        }
    }
}
```

然后点击uploadArchives,就可以将插件发布到/home/linjw/workspace/LocalMaven

{% img /一起入门gradle自定义插件编写一/2.png %}

#### 使用插件

让我们打开一个项目来验证下.

首先在项目根目录的build.gradle的buildscript.repositories里面配置本地仓库的路径,并且在buildscript.dependencies配置插件依赖:

{% img /一起入门gradle自定义插件编写一/3.png %}

最后在app目录下的build.gradle里面使用我们的插件:

{% img /一起入门gradle自定义插件编写一/4.png %}

就可以点击build看到输出了

```
16:57:12: Executing task 'build'...

Executing tasks: [build]

NDK is missing a "platforms" directory.
If you are using NDK, verify the ndk.dir is set to a valid NDK directory.  It is currently set to /home/linjw/android/sdk/ndk-bundle.
If you are not using NDK, unset the NDK variable from ANDROID_NDK_HOME or local.properties to remove this warning.

=======MyPlugin========
3
===============
NDK is missing a "platforms" directory.
If you are using NDK, verify the ndk.dir is set to a valid NDK directory.  It is currently set to /home/linjw/android/sdk/ndk-bundle.
If you are not using NDK, unset the NDK variable from ANDROID_NDK_HOME or local.properties to remove this warning.
```

#### 修改插件的ArtifactID

我们看到添加依赖的时候,插件的ArtifactID其实是app,这个又要怎么修改呢?

```
classpath 'me.islinjw.plugin:app:1.0.0'
```

回到我们的插件项目的根目录,修改settings.gradle,将模块名改成DemoPlugin:


```
//原来是include ':app'
include ':DemoPlugin'
```

然后将我们的app目录改名成DemoPlugin

最后再发布一次,就修改完成了

于是依赖就变成了

```
classpath 'me.islinjw.plugin:DemoPlugin:1.0.0'
```
