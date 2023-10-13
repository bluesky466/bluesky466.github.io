title: ClassLoader类加载流程补充
date: 2023-09-21 08:37:21
tags:
	- 技术相关
  - Android
---

之前写过一篇ClassLoader的[笔记](https://blog.islinjw.cn/2017/09/28/Android%E6%B8%A9%E6%95%85%E8%80%8C%E7%9F%A5%E6%96%B0-ClassLoader/)介绍了如何用ClassLoader去加载外部dex包,但是那个场景更多是插件化的场景,主要讲的是双亲委托的流程。

最近的项目里面涉及到了一点热修复的需求,如果用插件化的做法新增接口层或者改用反射调用代价比较大,更希望的是可以用外部dex的类直接替换apk内部的类。整个原理也比较简单,这里先把之前漏讲的findClass流程讲一下。

# findClass流程

安卓应用启动后的默认ClassLoader是PathClassLoader,而findClass方法实际是在父类BaseDexClassLoader里面定义的。

BaseDexClassLoader.findClass里面实际是调用DexPathList.findClass去加载的类:

```java
// https://cs.android.com/android/platform/superproject/+/android-platform-13.0.0_r6:libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java
...
private final DexPathList pathList;
...
protected Class<?> findClass(String name) throws ClassNotFoundException {
    ...
    List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
    Class c = pathList.findClass(name, suppressedExceptions);
    if (c != null) {
        return c;
    }
    ...
}
```

而DexPathList.findClass则是遍历dexElements去调用内部类Element的findClass最终调用DexFile.loadClassBinaryName:

```java
https://cs.android.com/android/platform/superproject/+/android-platform-13.0.0_r6:libcore/dalvik/src/main/java/dalvik/system/DexPathList.java
...
private Element[] dexElements;
...
public Class<?> findClass(String name, List<Throwable> suppressed) {
    for (Element element : dexElements) {
        Class<?> clazz = element.findClass(name, definingContext, suppressed);
        if (clazz != null) {
            return clazz;
        }
    }
    ...
}

static class Element {
	...
    private final DexFile dexFile;
	...
	public Class<?> findClass(String name, ClassLoader definingContext,
            List<Throwable> suppressed) {
        return dexFile != null ? dexFile.loadClassBinaryName(name, definingContext, suppressed)
                : null;
    }
	...
}
```

而DexFile.loadClassBinaryName最终会调用到DexFile.defineClassNative去到native层解析dex创建类:

```java
// https://cs.android.com/android/platform/superproject/+/android-platform-13.0.0_r6:libcore/dalvik/src/main/java/dalvik/system/DexFile.java
public Class loadClassBinaryName(String name, ClassLoader loader, List<Throwable> suppressed) {
    return defineClass(name, loader, mCookie, this, suppressed);
}

private static Class defineClass(String name, ClassLoader loader, Object cookie,
                                 DexFile dexFile, List<Throwable> suppressed) {
    Class result = null;
    try {
        result = defineClassNative(name, loader, cookie, dexFile);
    } catch (NoClassDefFoundError e) {
        if (suppressed != null) {
            suppressed.add(e);
        }
    } catch (ClassNotFoundException e) {
        if (suppressed != null) {
            suppressed.add(e);
        }
    }
    return result;
}
...
private static native Class defineClassNative(String name, ClassLoader loader, Object cookie,
                                                  DexFile dexFile)
            throws ClassNotFoundException, NoClassDefFoundError;
```

可以大概总结为BaseDexClassLoader委托DexPathList去加载类,而DexPathList内部有个Element数组,每个Element代表一个dex文件,DexPathList去加载类的原理则是遍历Element数组,看类在哪个dex可以加载出来。

# Tinker热修复原理

知道了类加载的流程之后,热修复的原理实际上也比较好理解: 用外部dex创建Element,插入到Element数组最前面。这样的话在findClass的时候就会优先加载外部dex的类,而不是apk内部的类了。

不过这里还有个小问题,如何用外部dex创建Element?

答案是我们可以用DexClassLoader加载dex让它帮我们生成Element,然后用反射获取。

获取到了之后也是比较顺理成章的用反射插入到默认的ClassLoader的pathList的Element数组最前面:

```java
// 用DexClassLoader加载外部dex，并获取Element数组
val dexClassLoader = DexClassLoader(dexFile.path, context.cacheDir.path, null, context.classLoader)
val newPathList = getDeclaredField(dexClassLoader, BaseDexClassLoader::class.java, "pathList")!!
val newDexElements = getDeclaredField(newPathList, "dalvik.system.DexPathList", "dexElements")!!

// 获取进程原本的Element数组
val oldPathList = getDeclaredField(context.classLoader, BaseDexClassLoader::class.java, "pathList")!!
val oldDexElements = getDeclaredField(oldPathList, "dalvik.system.DexPathList", "dexElements")!!

// 合并两个Element数组,把DexClassLoader的Element数组放在前面
val combineArray = combineDexArray(newDexElements, oldDexElements)

// 修改进程原本的Element数组为合并的新数组
setDeclaredField(oldPathList, "dalvik.system.DexPathList", "dexElements", combineArray)
```

完整的代码已经上传到[GitHub](https://github.com/bluesky466/HotfixDemo),demo里面DemoUtils.getString返回的是"this is a bug",而我修改成"bug fix"编译出jar之后用dx工具转换成hotfix.dex放到assets:

```kotlin
fun getString(): String {
    return "this is a bug"
    // return "bug fix"
}
```

在Application.onCreate里面加载这个dex:

```kotlin
val patch = File(cacheDir, HOTFIX_DEX)
assets.open(HOTFIX_DEX).use { src ->
    patch.outputStream().use { dest ->
        FileUtils.copy(src, dest)
    }
}
PatchLoader.loadPatch(this, patch)
```

最终在MainActivity里面读取出来的就是修复后的"bug fix":

```
findViewById<TextView>(R.id.label).text = DemoUtils.getString()
```

Tinker的核心原理就是这样的。不过这里还有个细节就是外部dex的加载是在Application里面执行的,单如果需要修复Application的bug怎么办?

它的解决方法是把Applcation的逻辑都挪到ApplicationLike里面,由Tinker加载完dex之后再在Application去调用ApplicationLike的生命周期回调。


# 其他热修复方案

除了修改Element数组方案之外还有其他的热修复方案可以参考下。

## 1.[Robust](https://tech.meituan.com/2017/03/17/android-autopatch.html):

使用插桩技术在每个类的每个方法最前面插入判断代码,如果有加载外部dex就反射执行外部dex对应的方法然后返回:

```java
public class DemoClass {
    // 插桩生成
    public static ChangeQuickRedirect changeQuickRedirect;

    public int foo() {
    	// 插桩生成
        if (changeQuickRedirect != null) {
            // 使用changeQuickRedirect去调用外部dex里面的DemoClass.foo方法
        }


        return 1;
    }
}
```

## 2.AndFix

从前面Tinker的原理我们可以看到类最终是由DexFile.defineClassNative在native层加载的,实际上java层的类和方法会对应native层的一堆指针,阿里的AndFix就是直接在native层把旧类的指针直接替换成外部dex新类的指针。


