title: Android温故而知新 - ClassLoader
date: 2017-09-28 02:22:09
tags:
    - 技术相关
    - Android
---

安卓插件化越来越流行,其中用到的技术不外乎加载外部的资源和加载外部的代码,关于加载外部资源我之前写过一篇文章[《安卓皮肤包机制的原理》](http://blog.islinjw.cn/2017/08/01/%E5%AE%89%E5%8D%93%E7%9A%AE%E8%82%A4%E5%8C%85%E6%9C%BA%E5%88%B6%E7%9A%84%E5%8E%9F%E7%90%86/),感兴趣的同学可以去看一下。

加载外部代码的作用在于热更新。程序主体定义接口,具体实现放在外部。只需要替换外部代码,就能修复bug甚至是更新功能。相比传统的ota手段更加省流量,用户体验也更加的好,毕竟有很多的用户是不喜欢更新的。

这篇文章我想复习一下ClassLoader的相关知识,它是加载外部代码的核心原理。

虽然android自己实现了一个特殊的虚拟机,它的类加载机制和普通的java程序有点区别。但是我还是想从普通的java程序讲起,一方面多知道点东西总是好的,另一方面它们的基本原理是一样的,对我们理解安卓的类加载机制也有很大的帮助。

# 普通java程序的类加载机制

我们都知道java代码需要先编译成class文件才能被jvm加载运行。那jvm又是如何加载class文件的呢?

其实class文件是通过ClassLoader加载到jvm的。java自带了三个ClassLoader,分别是:

- BootstrapClassLoader 用于加载核心类库
- ExtClassLoader 用于加载拓展库
- AppClassLoader 用于加载当前应用的类

然后需要说明的是java类不是一次性全部加载的,而是只有在用到的时候才会去加载。

因为全部加载的话会加载一些没有用到的类,造成资源的浪费。所以当程序需要用到某个类时,才会通过ClassLoader在系统的特定路径搜索这个类的class文件并将它加载到jvm去执行。

ExtClassLoader和AppClassLoader都是URLClassLoader的子类,他们内部保存了URL列表用于指定搜索路径。我们可以通过URLClassLoader.getURLs()方法获取到这个URL列表。

BootstrapClassLoader虽然不是URLClassLoader的子类,但我们也可以从sun.misc.Launcher.getBootstrapClassPath().getURLs()方法获取到BootstrapClassLoader的搜索路径。

下面的代码打印了各个ClassLoader的搜索路径:

```
import java.net.URL;
import java.net.URLClassLoader;

public class ClassLoaderURLs {
    public static void main(String[] args) {
        System.out.println("BootstrapClassLoader urls :");
        URL[] urls = sun.misc.Launcher.getBootstrapClassPath().getURLs();
        for (URL url : urls) {
            System.out.println(url);
        }

        URLClassLoader extClassLoader = (URLClassLoader) ClassLoader.getSystemClassLoader().getParent();
        System.out.println("\n" + extClassLoader + " urls :");
        urls = extClassLoader.getURLs();
        for (URL url : urls) {
            System.out.println(url);
        }

        URLClassLoader appClassLoader = (URLClassLoader) ClassLoader.getSystemClassLoader();
        System.out.println("\n" + appClassLoader + " urls :");
        urls = appClassLoader.getURLs();
        for (URL url : urls) {
            System.out.println(url);
        }
    }
}
```

打印如下:

```
BootstrapClassLoader urls :
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/resources.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/rt.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/sunrsasign.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/jsse.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/jce.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/charsets.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/jfr.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/classes

sun.misc.Launcher$ExtClassLoader@74a14482 urls :
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/ext/cldrdata.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/ext/dnsns.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/ext/jfxrt.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/ext/localedata.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/ext/nashorn.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/ext/sunec.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/ext/sunjce_provider.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/ext/sunpkcs11.jar
file:/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/jre/lib/ext/zipfs.jar
file:/System/Library/Java/Extensions/AppleScriptEngine.jar
file:/System/Library/Java/Extensions/dns_sd.jar
file:/System/Library/Java/Extensions/j3daudio.jar
file:/System/Library/Java/Extensions/j3dcore.jar
file:/System/Library/Java/Extensions/j3dutils.jar
file:/System/Library/Java/Extensions/jai_codec.jar
file:/System/Library/Java/Extensions/jai_core.jar
file:/System/Library/Java/Extensions/libAppleScriptEngine.jnilib
file:/System/Library/Java/Extensions/libJ3D.jnilib
file:/System/Library/Java/Extensions/libJ3DAudio.jnilib
file:/System/Library/Java/Extensions/libJ3DUtils.jnilib
file:/System/Library/Java/Extensions/libmlib_jai.jnilib
file:/System/Library/Java/Extensions/mlibwrapper_jai.jar
file:/System/Library/Java/Extensions/MRJToolkit.jar
file:/System/Library/Java/Extensions/vecmath.jar
file:/usr/lib/java/libjdns_sd.jnilib

sun.misc.Launcher$AppClassLoader@28d93b30 urls :
file:/Users/linjw/workspace/class_loader_demo/
```

我们可以看到这些url有指向jar包的,也有指向一个目录的(还有指向.jnilib文件的,这个我们可以不用管)。

ClassLoader从指定的路径下搜索class文件。而jar包其实是一个压缩包,将class文件打包在一起,所以ClassLoader也可以从jar包中搜索需要用到的class。

## Java类的加载流程

### ClassLoader的创建

我们先从ClassLoader的创建开始说起。我们可以直接看[sun.misc.Launcher](http://www.grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/sun/misc/Launcher.java)的源码,它在构造函数中创建了ExtClassLoader和AppClassLoader:

```
public Launcher() {
    // Create the extension class loader
    ClassLoader extcl;
    try {
        extcl = ExtClassLoader.getExtClassLoader();
    } catch (IOException e) {
        throw new InternalError(
            "Could not create extension class loader", e);
    }

    // Now create the class loader to use to launch the application
    try {
        loader = AppClassLoader.getAppClassLoader(extcl);
    } catch (IOException e) {
        throw new InternalError(
            "Could not create application class loader", e);
    }

    // Also set the context class loader for the primordial thread.
    Thread.currentThread().setContextClassLoader(loader);

	...
}
```

ExtClassLoader.getExtClassLoader()是一个工厂方法:

```
public static ExtClassLoader getExtClassLoader() throws IOException
{
    final File[] dirs = getExtDirs();

    try {
        // Prior implementations of this doPrivileged() block supplied
        // aa synthesized ACC via a call to the private method
        // ExtClassLoader.getContext().

        return AccessController.doPrivileged(
            new PrivilegedExceptionAction<ExtClassLoader>() {
                public ExtClassLoader run() throws IOException {
                    int len = dirs.length;
                    for (int i = 0; i < len; i++) {
                        MetaIndex.registerDirectory(dirs[i]);
                    }
                    return new ExtClassLoader(dirs);
                }
            });
    } catch (java.security.PrivilegedActionException e) {
        throw (IOException) e.getException();
    }
}
```

AppClassLoader.getAppClassLoader(final ClassLoader extcl)也是一个工厂方法,它需要传入一个ClassLoader作为AppClassLoader的父ClassLoader。而我们将ExtClassLoader传了进去,也就是说ExtClassLoader是AppClassLoader的父ClassLoader:

```
public static ClassLoader getAppClassLoader(final ClassLoader extcl)
    throws IOException
{
    final String s = System.getProperty("java.class.path");
    final File[] path = (s == null) ? new File[0] : getClassPath(s);

    // Note: on bugid 4256530
    // Prior implementations of this doPrivileged() block supplied
    // a rather restrictive ACC via a call to the private method
    // AppClassLoader.getContext(). This proved overly restrictive
    // when loading  classes. Specifically it prevent
    // accessClassInPackage.sun.* grants from being honored.
    //
    return AccessController.doPrivileged(
        new PrivilegedAction<AppClassLoader>() {
            public AppClassLoader run() {
            URL[] urls =
                (s == null) ? new URL[0] : pathToURLs(path);
            return new AppClassLoader(urls, extcl);
        }
    });
}
```

每一个ClassLoader都有一个父ClassLoader,我们可以通过ClassLoader.getParent()方法获取。同时我们也能使用Class.getClassLoader()获取加载这个类的ClassLoader。所以让我们来看看下面的代码:

```
public class GetClassLoader {
    public static void main(String[] args) {
        ClassLoader loader = GetClassLoader.class.getClassLoader();
        do {
            System.out.println(loader);
        } while ((loader = loader.getParent()) != null);
    }
}
```

查看打印我们可以知道, GetClassLoader是AppClassLoader加载的,而AppClassLoader的父ClassLoader是ExtClassLoader:

```
sun.misc.Launcher$AppClassLoader@28d93b30
sun.misc.Launcher$ExtClassLoader@74a14482
```

但是如果我们查看String的ClassLoader又会发现它是null的:

```
public class GetClassLoader {
    public static void main(String[] args) {
        ClassLoader loader = String.class.getClassLoader();
        System.out.println("loader : " + loader);
    }
}
```

```
loader : null
```

那是不是说String不是由ClassLoader加载的?当然不是!其实String是BootstrapClassLoader加载的。BootstrapClassLoader负责加载java的核心类。

但是为什么String.class.getClassLoader()拿到的是null呢？

原因是BootstrapClassLoader实际上不是一个java类,它是由C/C++编写的,它本身是虚拟机的一部分。所以在java中当然没有办法获取到它的引用。

### 双亲委托

相信大家如果知道ClassLoader的话应该有听说过双亲委托,那下面我们就来讲一下双亲委托究竟是怎么一回事。

我们知道ClassLoader.loadClass()的方法可以加载一个类,所以研究一个类的加载流程,最好的方法当然还是去看源码啦:

```
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // 首先,从缓存中查询该类是不是被加载过,如果加载过就可以直接返回
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
            		//判断它的父ClassLoader是否为空,如果不为空就调用父ClassLoader的loadClass方法去加载该类
                    c = parent.loadClass(name, false);
                } else {
                	//如果它的父ClassLoader为空,则调用BootstrapClassLoader去加载该类,所以此时从逻辑上来讲BootstrapClassLoader是父ClassLoader
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                
            }

            if (c == null) {
                long t1 = System.nanoTime();
                
                //如果父ClassLoader不能加载该类才由自己去加载,这个方法从本ClassLoader的搜索路径中查找该类
                c = findClass(name);
                
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

从代码中我们可以看到,加载一个类的时候,ClassLoader先会让父类去加载,如果父类加载失败,才会由它自己去加载,这就是我们说的双亲委托。

为什么类加载需要设计成双亲委托的方式呢？原因就在于双亲委托可以防止类被重复加载。如果父ClassLoader已经加载过一个类了,子ClassLoader就不会再次加载,可以防止同一个类被两个ClassLoader重复加载的问题。

这里还需要说的是,当我们自定义一个ClassLoader的时候,最好将AppClassLoader设为父ClassLoader。这样的话可以保证我们自定义的ClassLoader找加载类失败的时候还能从父ClassLoader中加载这个类。

双亲委托模式的流程如下图所示:


{% img /Android温故而知新-ClassLoader/1.png %}


## 自定义ClassLoader

有时候我们可以继承ClassLoader实现自己的类加载器。自定义ClassLoader有两种方式:

1. 重写loadClass方法 
2. 重写findClass方法

他们有什么区别呢,还记得上一级ClassLoader.loadClass()的源码吗？loadClass方法内会先调用父ClassLoader的loadClass方法,如果父ClassLoader没有加载过该类才会调用本ClassLoader的findClass方法去加载类。

所以如果想要打破双亲委托机制的话就可以loadClass(),而如果还想继续沿用双亲委托机制的话就只需要重写findClass就好了。

我们写个小例子:

```
public class MyClassLoader extends ClassLoader {
    public String mClassDir;

    public MyClassLoader(String classDir) {
        this.mClassDir = classDir;
    }


    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        File file = new File(mClassDir, getClassFileName(name));
        if (file.exists()) {
            try {
                FileInputStream is = new FileInputStream(file);

                ByteArrayOutputStream buf = new ByteArrayOutputStream();
                int len;
                while ((len = is.read()) != -1) {
                    buf.write(len);
                }

                byte[] data = buf.toByteArray();
                is.close();
                buf.close();

                return defineClass(name, data, 0, data.length);
            } catch (FileNotFoundException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        return super.findClass(name);
    }

    private String getClassFileName(String fullName) {
        int index = fullName.lastIndexOf(".");
        if (index == -1) {
            return fullName + ".class";
        } else {
            return fullName.substring(index + 1) + ".class";
        }
    }
}
```

因为我们不需要打破双亲委托机制所以只需要重写findClass方法就可以了。我们自定义的ClassLoader会从指定的路径中搜索class文件,将它读入内存,然后通过调用ClassLoader.defineClass()方法去加载这个类。

我们在/Users/linjw/workspace/class\_loader\_demo目录下创建了一个Test.java:

```
package linjw.demo.classloader;
public class Test {
    public String getData() {
        return "Hello World";
    }
}
```

然后通过javac命令编译出Test.class文件,同样放在/Users/linjw/workspace/class\_loader\_demo目录下。

然后用我们的MyClassLoader去加载它:

```
MyClassLoader loader = new MyClassLoader("/Users/linjw/workspace/class_loader_demo");
Class clazz = loader.loadClass("linjw.demo.classloader.Test");
if (clazz != null) {
    Object obj = clazz.newInstance();
    Method method = clazz.getDeclaredMethod("getData");
    String result = (String) method.invoke(obj);
    System.out.println(result);
    System.out.println("ClassLoader : " + clazz.getClassLoader());
} else {
    System.out.println("can't load class");
}
```

可以看到下面的打印,说明我们已经成功用MyClassLoader加载了Test这个类:

```
Hello World
ClassLoader : linjw.demo.classloader.MyClassLoader@66cd51c3
```

这里还有一个小的知识点,如果一个类是由某个ClassLoader加载的,那么它import的类也会由这个ClassLoader去加载。这里我们可以做一个实验:

```
package linjw.demo.classloader;

import linjw.demo.classloader.Test;

public class Test2 {
    public String getData(){
        return "Test ClassLoader : " + Test.class.getClassLoader();
    }
}
```

我们写一个Test2类,它会import Test并返回Test的ClassLoader。让我们写个demo看看这个Test的ClassLoader:

```
MyClassLoader loader = new MyClassLoader("/Users/linjw/workspace/class_loader_demo");
Class clazz = loader.loadClass("linjw.demo.classloader.Test2");
if (clazz != null) {
    Object obj = clazz.newInstance();
    Method method = clazz.getDeclaredMethod("getData");
    String result = (String) method.invoke(obj);
    System.out.println(result);
} else {
    System.out.println("can't load class");
}
```

通过打印可以知道Test也是由MyClassLoader加载的:

```
linjw.demo.classloader.MyClassLoader@66cd51c3
```

## Context ClassLoader

Context ClassLoader并不是一个实际的类,它只是Thread的一个成员变量:

```
public class Thread implements Runnable {
	private ClassLoader contextClassLoader;

	private void init2(Thread parent) {
        this.contextClassLoader = parent.getContextClassLoader();
        ...
    }

    public ClassLoader getContextClassLoader() {
        return contextClassLoader;
    }

    public void setContextClassLoader(ClassLoader cl) {
        contextClassLoader = cl;
    }
    
    ...
}
```

每个Thread都有一个相关联的ClassLoader,子线程默认使用父线程的ClassLoader。

而线程的默认ClassLoader是AppClassLoader:

```
public Launcher() {
    ...
    
    try {
        loader = AppClassLoader.getAppClassLoader(extcl);
    } catch (IOException e) {
        throw new InternalError(
            "Could not create application class loader", e);
    }
    
    //设置AppClassLoader为当前线程的Context ClassLoader
    Thread.currentThread().setContextClassLoader(loader);

	...
}
```

Context ClassLoader的存在是为了解决使用双亲委托机制下父ClassLoader无法找到子ClassLoader的问题。假如有下面的委托链:

ClassLoaderA -> AppClassLoader -> ExtClassLoader -> BootstrapClassLoader

那么委派链左边的ClassLoader就可以很自然的使用右边的ClassLoader所加载的类。 

但如果是右边的ClassLoader想要反过来使用左边的ClassLoader所加载的类就无能为力了。

这个时候如果使用Context ClassLoader就能从线程中获得左边的ClassLoader了。

那什么时候会出现右边的ClassLoader想要反过来使用左边的ClassLoader所加载的类的情况呢？

我们上一节刚刚说过:“如果一个类是由某个ClassLoader加载的,那么它import的类也会由这个ClassLoader去加载”。

举个例子,Java 提供了很多服务提供者接口（Service Provider Interface，SPI）,允许第三方为这些接口提供实现。如JAXP(XML处理的Java API)的SPI__接口__定义包含在 javax.xml.parsers包中，它是由BootstrapClassLoader加载的。

但是它的实现代码很可能是作为Java应用所依赖的jar包被包含进来,如实现了JAXP SPI的Apache Xerces所包含的jar包,它由AppClassLoader加载。

我们用javax.xml.parsers.DocumentBuilderFactory类中的newInstance()方法用来生成一个新的DocumentBuilderFactory的实例, DocumentBuilderFactory是一个抽象类,它定是java核心库的一部分,由BootstrapClassLoader去加载。因此,DocumentBuilderFactory里面import的类都由BootstrapClassLoader去加载。

但是DocumentBuilderFactory的实现类却是在org.apache.xerces.jaxp.DocumentBuilderFactoryImpl中定义的, BootstrapClassLoader无法加载它。这个时候就需要在DocumentBuilderFactory. newInstance()的代码中使用Context ClassLoader，找到AppClassLoader去加载DocumentBuilderFactoryImpl这个实现类。


# 安卓中的ClassLoader

安卓的的类也是通过ClassLoader加载的,但是并不是java中的BootstrapClassLoader、 ExtClassLoader或者AppClassLoader。写个小demo看看安卓中加载类的是哪些ClassLoader:

```
Log.d("DxClassLoader", "BootClassLoader :" + String.class.getClassLoader());

ClassLoader loader = MainActivity.class.getClassLoader();
do {
	Log.d("DxClassLoader", "loader :" + loader);
} while ((loader = loader.getParent()) != null);
```

打印如下:

```
09-27 23:11:03.432 21151-21151/? D/DxClassLoader: BootClassLoader :java.lang.BootClassLoader@ad96016
09-27 23:11:03.432 21151-21151/? D/DxClassLoader: loader :dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/linjw.demo.classloader-2/base.apk"],nativeLibraryDirectories=[/data/app/linjw.demo.classloader-2/lib/arm64, /vendor/lib64, /system/lib64]]]
09-27 23:11:03.433 21151-21151/? D/DxClassLoader: loader :java.lang.BootClassLoader@ad96016
```

我们可以看到安卓中用的了PathClassLoader和BootClassLoader两个ClassLoader,其中BootClassLoader是PathClassLoader的parent。

而和在java程序不同的是String是由BootClassLoader加载的。安卓的BootClassLoader其实就相当于java的BootstrapClassLoader,只不过它是由java实现的而不是由c/c++实现的。

## PathClassLoader

我们在上一节中将PathClassLoader打印出来的时候可以看到一个apk路径:

```
[zip file "/data/app/linjw.demo.classloader-2/base.apk"]
```

apk其实是一个也是一个zip压缩包,我们可以将一个apk文件后缀改成.zip然后就可以直接解压了。PathClassLoader的作用其实就是在这个zip包中加载dex文件,我们通过它甚至可以加载其他应用的代码,但它只能加载已安装的应用。

例如我们可以新建一个ext工程,它的包名为linjw.demo.classloader.ext,然后在里面创建Test类:

```
package linjw.demo.classloader.ext;

public class Test {
    public String getData() {
        return "Hello World";
    }
}
```

然后编译出apk来,并且安装。之后就能从这个apk中加载出Test类了:

```
String path = null;
PackageManager pm = getPackageManager();
try {
    path = pm.getApplicationInfo("linjw.demo.classloader.ext", 0).sourceDir;
} catch (PackageManager.NameNotFoundException e) {
    e.printStackTrace();
}

PathClassLoader loader = new PathClassLoader(path, ClassLoader.getSystemClassLoader());

try {
    Class clazz = loader.loadClass("linjw.demo.classloader.ext.Test");

    if (clazz != null) {
        Object obj = clazz.newInstance();
        Method method = clazz.getDeclaredMethod("getData");
        String result = (String) method.invoke(obj);
        Log.d("DxClassLoader", result);
    } else {
        Log.d("DxClassLoader", "can't load class");
    }
} catch (ClassNotFoundException e) {
    e.printStackTrace();
} catch (NoSuchMethodException e) {
    e.printStackTrace();
} catch (InstantiationException e) {
    e.printStackTrace();
} catch (IllegalAccessException e) {
    e.printStackTrace();
} catch (InvocationTargetException e) {
    e.printStackTrace();
}
```

可以得到打印:

```
09-27 23:39:16.571 24077-24077/? D/DxClassLoader: Hello World
```

## DexClassLoader

PathClassLoader只能加载已经安装的应用里面的类,但是DexClassLoader却能加载未安装的应用里面的类。例如我们将apk放到存储卡目录下而不去安装它:

```
String dir = Environment.getExternalStorageDirectory().getAbsolutePath();
File apk = new File(dir, "Ext.apk");
File dexOutputDir = this.getDir("dex", 0);
DexClassLoader loader = new DexClassLoader(
        apk.getAbsolutePath(),
        dexOutputDir.getAbsolutePath(),
        null, getClassLoader());

try {
    Class clazz = loader.loadClass("linjw.demo.classloader.ext.Test");

    if (clazz != null) {
        Object obj = clazz.newInstance();
        Method method = clazz.getDeclaredMethod("getData");
        String result = (String) method.invoke(obj);
        Log.d("DxClassLoader", result);
    } else {
        Log.d("DxClassLoader", "can't load class");
    }
} catch (ClassNotFoundException e) {
    e.printStackTrace();
} catch (NoSuchMethodException e) {
    e.printStackTrace();
} catch (InstantiationException e) {
    e.printStackTrace();
} catch (IllegalAccessException e) {
    e.printStackTrace();
} catch (InvocationTargetException e) {
    e.printStackTrace();
}
```

同样可以得到打印:

```
09-27 23:54:29.206 25472-25472/? D/DxClassLoader: Hello World
```

我们可以看到, DexClassLoader的构造函数的参数比PathClassLoader的要多出一个optimizedDirectory:

```
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory, String librarySearchPath, ClassLoader parent) {
        super((String)null, (File)null, (String)null, (ClassLoader)null);
        throw new RuntimeException("Stub!");
    }
}
```

```
public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super((String)null, (File)null, (String)null, (ClassLoader)null);
        throw new RuntimeException("Stub!");
    }

    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
        super((String)null, (File)null, (String)null, (ClassLoader)null);
        throw new RuntimeException("Stub!");
    }
}
```

那这个optimizedDirectory到底有什么作用呢?其实optimizedDirectory是用来存放从apk中解压出来的dex文件的。

DexClassLoader和PathClassLoader其实归根结底都是通过DexFile这个类去加载的dex文件,并不是直接读取的apk。因为如果每次都需要解压才能加载代码的话效率实在太低了。

DexClassLoader可以主动解压apk,所以可以加载未安装的应用中的代码。但PathClassLoader不会主动解压apk,它是读取的已经安装的apk在cache中存在缓存的dex文件,所以它只能加载已安装应用中的代码。

## 生成dex文件

DexClassLoader和PathClassLoader最后都是加载的dex文件。所以我们可以直接将dex文件的路径传给他们去加载。但dex文件又是个什么东西呢？

普通的java程序中,JVM虚拟机可以通过ClassLoader去加载jar到的加载类的目的。但是android使用的Dalvik/ART虚拟机不能直接加载jar包,需要把.jar文件优化成.dex文件才能加载。所以实际上dex文件是优化过的jar包。

我们可以用Android SDK提供的DX工具把.jar文件优化成.dex文件。我们用之前的Test.java做例子,具体步骤如下:

1.使用javac命令编译Test.java得到Test.class文件(我这边的java环境是1.8的,如果不指定用1.7的话生成dex也会失败,报__com.android.dx.cf.iface.ParseException: bad class file magic (cafebabe) or version (0034.0000)__)

```
javac -source 1.7 -target 1.7 Test.java
```

2.将创建目录子目录linjw/demo/classloader/ext并将Test.class移动到子目录中(因为Test的package是linjw.demo.classloader.ext,所以要根据它生成同样的目录,要不然生成dex会失败)

```
mkdir -p linjw/demo/classloader/ext
mv Test.class linjw/demo/classloader/ext
```

3.使用jar命令将linjw目录打包成jar包

```
jar -cf Test.jar linjw
```

4.用dx工具将jar包优化成dex包

```
/Users/linjw/androidsdk/android-sdk-macosx/build-tools/19.1.0/dx --dex --output=Test.dex Test.jar
```

## 动态加载dex文件

然后我们就能将它放到存储卡中用DexClassLoader或者PathClassLoader去加载了。

### 使用反射的反射加载

```

String dir = Environment.getExternalStorageDirectory().getAbsolutePath();
File dex = new File(dir, "Test.dex");
File dexOutputDir = this.getDir("dex", 0);

//使用PathClassLoader加载dex
//PathClassLoader loader = new PathClassLoader(dex.getAbsolutePath(), getClassLoader());

//使用DexClassLoader加载dex
DexClassLoader loader = new DexClassLoader(
        dex.getAbsolutePath(),
        dexOutputDir.getAbsolutePath(),
        null,
        getClassLoader());

try {
    Class clazz = loader.loadClass("linjw.demo.classloader.ext.Test");

    if (clazz != null) {
        Object obj = clazz.newInstance();
        Method method = clazz.getDeclaredMethod("getData");
        String result = (String) method.invoke(obj);
        Log.d("DxClassLoader", result);
    } else {
        Log.d("DxClassLoader", "can't load class");
    }
} catch (ClassNotFoundException e) {
    e.printStackTrace();
} catch (NoSuchMethodException e) {
    e.printStackTrace();
} catch (InstantiationException e) {
    e.printStackTrace();
} catch (IllegalAccessException e) {
    e.printStackTrace();
} catch (InvocationTargetException e) {
    e.printStackTrace();
}
```


### 使用接口的方式加载

或者我们也可以使用接口的方式:

1.添加ITest接口:

```
package linjw.demo.classloader.ext;

public interface ITest {
    String getData();
}
```

2.Test类实现ITest接口:

```
package linjw.demo.classloader.ext;

public class Test implements ITest {
    @Override
    public String getData() {
        return "Hello World";
    }
}
```

3.将它们一起打包到Test.dex

```
javac -source 1.7 -target 1.7 Test.java ITest.java

mkdir -p linjw/demo/classloader/ext

mv Test.class linjw/demo/classloader/ext

mv Test.class linjw/demo/classloader

jar -cf Test.jar linjw

/Users/linjw/androidsdk/android-sdk-macosx/build-tools/19.1.0/dx --dex --output=Test.dex Test.jar
```

4.在安卓项目中导入ITest接口并调整代码:

```
String dir = Environment.getExternalStorageDirectory().getAbsolutePath();
File dex = new File(dir, "Test.dex");
File dexOutputDir = this.getDir("dex", 0);

//使用PathClassLoader加载dex
//PathClassLoader loader = new PathClassLoader(dex.getAbsolutePath(), getClassLoader());

//使用DexClassLoader加载dex
DexClassLoader loader = new DexClassLoader(
    dex.getAbsolutePath(),
    dexOutputDir.getAbsolutePath(),
    null,
    getClassLoader());

try {
Class clazz = loader.loadClass("linjw.demo.classloader.ext.Test");

if (clazz != null) {
    //注意这里,使用的是ITest
    ITest obj = (ITest) clazz.newInstance();
    String result = obj.getData();
    Log.d("DxClassLoader", result);
} else {
    Log.d("DxClassLoader", "can't load class");
}
} catch (ClassNotFoundException e) {
e.printStackTrace();
} catch (InstantiationException e) {
e.printStackTrace();
} catch (IllegalAccessException e) {
e.printStackTrace();
}
```

其实我比较推荐使用在程序主体中定义接口,加载外部实现代码的这种方法。一方面它比反射的效率高,另一方面也比较容易阅读。
