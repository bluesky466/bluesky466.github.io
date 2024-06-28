title: Android so链接的一些坑
date: 2024-06-26 19:19:51
tags:
    - 技术相关
    - Android
    - C/C++
---

# SONAME缺失

前几天遇到了个比较诡异的链接问题,分析下来感觉挺有意思的。

背景是我们导入了供应商给的几个so,编译成功之后在机器上运行出现链接报错:

```
06-26 08:10:01.940 25976 25976 E AndroidRuntime: java.lang.UnsatisfiedLinkError: dlopen failed: library "/Users/linjw/workspace/Demo/app/src/main/cpp/../../../libs/arm64-v8a/libcjson.so" not found: needed by /data/app/~~Y8XCESOaI01yUY_5GwBPeg==/me.linjw.demo-HHKb0jSb43YhjfSIfuxutw==/lib/arm64/libDemo.so in namespace classloader-namespace
```

`libcjson.so`的确是其中一个so，但可以看到它的运行报错居然是去找我的开发电脑上的这个路径:`/Users/linjw/workspace/Demo/app/src/main/cpp/../../../libs/arm64-v8a/libcjson.so`。

这样的问题首先我们可以在`adb shell`里面用`readelf`命令或者在开发电脑里的ndk目录下找到对应abi的`readelf`工具看看`libDemo.so`的信息:

```
# readelf -d /data/app/~~Y8XCESOaI01yUY_5GwBPeg==/me.linjw.demo-HHKb0jSb43YhjfSIfuxutw==/lib/arm64/libDemo.so

Dynamic section at offset 0x3f6c8 contains 38 entries:
  Tag                Type                 Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [/Users/linjw/workspace/Demo/app/src/main/cpp/../../../libs/arm64-v8a/libcjson.so]
 0x0000000000000001 (NEEDED)             Shared library: [libcurl.so.4]
 0x0000000000000001 (NEEDED)             Shared library: [libcrypto.so.1.1]
 ...
```

可以看到的确有一个`NEEDED`配置的是`/Users/linjw/workspace/Demo/app/src/main/cpp/../../../libs/arm64-v8a/libcjson.so`,但是可以看到其他的像`libcurl.so.4`和`libcrypto.so.1.1`也是供应商提供的,他们就没有带开发电脑的路径。从CMake配置上看他们的配置方式是一样的:

```
set(lib_path ${CMAKE_SOURCE_DIR}/../../../libs)

add_library(cjson SHARED IMPORTED)
set_target_properties(cjson PROPERTIES IMPORTED_LOCATION ${lib_path}/${ANDROID_ABI}/libcjson.so)

add_library(curl SHARED IMPORTED)
set_target_properties(curl PROPERTIES IMPORTED_LOCATION ${lib_path}/${ANDROID_ABI}/libcurl.so)

add_library(crypto SHARED IMPORTED)
set_target_properties(crypto PROPERTIES IMPORTED_LOCATION ${lib_path}/${ANDROID_ABI}/libcrypto.so)
```

那么问题就只能出现在他们的so本身,我们继续用`readelf`去对比看看这几个so的区别:

{% img /AndroidSo链接的一些坑/1.png %}

可以看到`libcrypto.so`和`libcurl.so`都是带有`SONAME`的,但是`libcjson.so`没有携带。我之前在[其他的问题](https://blog.islinjw.cn/2020/11/24/JNI%E8%BF%90%E8%A1%8C%E9%94%99%E8%AF%AF-%E7%AC%A6%E5%8F%B7%E6%9C%AA%E5%AE%9A%E4%B9%89/)里面遇到过`SONAME`配错了导致找不到符号的问题。看起链接器在链接的时候是使用so的`SONAME`字段而不是文件名去写入target的NEEDED字段所以造成了这个问题。

## so的几个名字

这里我们再回顾下so几个name的作用:

### realname

realname实际上就是so的文件名，一般格式为`lib${name}.so.${major}.${minor}.${revision}`例如libcurl.so.4.5.0，我们可以在编译的时候用-o参数指定:

```
gcc -shared -o $(realname) …
```

### linkname

linkname是在链接时使用的,用-l参数指定例如下面的foo就是linkname。我们在这里不需要填so文件的名字，gcc会自动为linkname补上lib和.so，去链接lib$(name).so

```
gcc main.c -L. -lfoo
```

另外我们在java里面加载so填的也是linkname:

```
System.loadLibrary("Demo");
```

### soname

soname顾名思义就是so的名字，它可以在编译的时候用`−Wl,−soname,${soname}`指定，-Wl,表示后面的参数将传给link程序ld:

```
gcc -shared -fPIC -Wl,-soname,libfoo.so.0 -o libfoo.so.0.0.0 foo.c
```

如前面所见,soname会被记录在so的二进制数据中。在链接目标程序的时候也会将soname填入目标程序的NEEDED字段记录依赖,**如果so里面没有SONAME字段则将文件路径打入目标程序的NEEDED字段**。在加载目标程序的时候则是根据这个NEEDED去相应目录加载`${NEEDED}`这个文件。

## patchelf

如果我们有源码,当然可以修改编译配置把SONAME加入到`libcjson.so`,但是这个so是供应商提供的。我们可以先用[patchelf](https://github.com/NixOS/patchelf/releases)工具尝试给它加上SONAME验证看看。下载[patchelf-0.18.0-aarch64.tar.gz](https://github.com/NixOS/patchelf/releases/download/0.18.0/patchelf-0.18.0-aarch64.tar.gz)解压出patchelf直接adb push到安卓机器上去运行:

```
patchelf --set-soname libcjson.so libcjson.so
```

然后再把修改后的libcjson.so用adb pull回来重新编译app。运行之后可以发现前面的报错的确没有了,证明的确是SONAME缺失导致的。


# so的版本号问题

但是却出现了其他的报错:

```
06-26 08:46:47.737 30092 30092 E AndroidRuntime: java.lang.UnsatisfiedLinkError: dlopen failed: library "libcrypto.so.1.1" not found: needed by /data/app/~~Y8XCESOaI01yUY_5GwBPeg==/me.linjw.demo-HHKb0jSb43YhjfSIfuxutw==/lib/arm64/libDemo.so in namespace classloader-namespace
```

这是由于`libcrypto.so`的SONAME字段是`libcrypto.so.1.1`,所以libDemo.so在链接它之后NEEDED字段填入的也是`libcrypto.so.1.1`:

```
# readelf -d /data/app/~~Y8XCESOaI01yUY_5GwBPeg==/me.linjw.demo-HHKb0jSb43YhjfSIfuxutw==/lib/arm64/libDemo.so

Dynamic section at offset 0x3f6c8 contains 38 entries:
  Tag                Type                 Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [/Users/linjw/workspace/Demo/app/src/main/cpp/../../../libs/arm64-v8a/libcjson.so]
 0x0000000000000001 (NEEDED)             Shared library: [libcurl.so.4]
 0x0000000000000001 (NEEDED)             Shared library: [libcrypto.so.1.1]
 ...
```

但我们导入apk的so名字是`libcrypto.so`,在安装目录只有`libcrypto.so`找不到`libcrypto.so.1.1`这个名字的so:

```
# ls /data/app/~~Y8XCESOaI01yUY_5GwBPeg==/me.linjw.demo-HHKb0jSb43YhjfSIfuxutw==/lib/arm64/ | grep libcrypto
libcrypto.so
```

所以比较容易想到的是把`libcrypto.so`文件名改成`libcrypto.so.1.1`,在adb shell里面用`mv`命令修改名字运行时可以的。但代码工程里面修改so名字再去编译,实际编译出来之后仍然报错。这个时候在安装目录甚至都找不到`libcrypto.so.1.1`。

原因就是虽然安卓系统是支持这种加载带版本后缀的so,但是gradle在编译apk的时候确是只会将`.so`后缀的文件打包到apk,所以安装之后就缺失了这个so。

在Android上库不是在系统范围内安装的它们总是应用程序包的一部分,所以so的版本标记是不必要的,谷歌就把这块在打包的时候去掉了,但这样的差异造成了在安卓上使用c/c++库方面需要对so的版本号进行额外的处理。例如在编译ffmpeg的时候编译参数添加`--target-os=android`最终链接的时候就会添加`-shared -Wl,-soname,$(SLIBNAME)`参数指定`soname`为不带版本后缀的`SLIBNAME`:

```
# ffmpeg-4.4.2 configure
...
SLIBPREF="lib"
SLIBSUF=".so"
SLIBNAME='$(SLIBPREF)$(FULLNAME)$(SLIBSUF)'
...
# OS specific
case $target_os in
    ...
    android)
        disable symver
        enable section_data_rel_ro
        add_cflags -fPIE
        add_ldexeflags -fPIE -pie
        SLIB_INSTALL_NAME='$(SLIBNAME)'
        SLIB_INSTALL_LINKS=
        SHFLAGS='-shared -Wl,-soname,$(SLIBNAME)'
        ;;
    ...
```


解决这个问题除了修改编译配置重新编译之外,如果没有源代码同样可以用`patchelf`把`libcrypto.so`的SONAME改成`libcrypto.so`,不过由于蛮多第三方库交叉编译之后都会出现带版本后缀so文件名和soname的情况,这里我再提供两个思路。


## so的搜索路径

一个是可以用`rpath`或者`runpath`去解决。

安卓默认会按照优先级搜索下面的路径:

- so文件的RPATH字段指的的目录
- LD\_LIBRARY\_PATH环境变量指定的目录
- so文件的RUNPATH字段指的的目录
- 应用的安装目录如上面的(/data/app/~~Y8XCESOaI01yUY_5GwBPeg==/me.linjw.demo-HHKb0jSb43YhjfSIfuxutw==/lib/arm64/)
- 系统目录如/system/lib64/、/vendor/lib64/、/system/apex/com.android.i18n/lib64/等

所以我们可以在CMakeLists.txt对libDemo.so添加如下link参数指定`rpath`到应用的内部私有目录:

```
project("Demo")
...
set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES LINK_FLAGS "-Wl,-rpath,/data/data/me.linjw.demo/cache")
```

然后在第一次运行的时候将带有版本后缀的so拷贝到这个目录下。然后加载libDemo.so的时候就会先去到这个`rpath`指的的目录去搜索NEEDED so。

如果有多个目录需要指定`rpath`可以用冒号分割,例如`"-Wl,-rpath,/data/data/me.linjw.demo/cache:/data/data/me.linjw.demo/files"`

另外从前面的搜索目录来看,Linux并不会在可执行程序的当前目录下去搜索so。而`rpath`还有个`$ORIGIN`变量它指定的是可执行程序的位置,例如我们写的一个可执行程序依赖了某个so,可以将`rpath`指定为`$ORIGIN`,那么只要so和可执行程序在同一个目录就能搜索到。

## so缓存

另外一个是我们在load `libDemo.so`之前手动调用`System.loadLibrary("crypto")`去load `libcrypto.so`,然后load的时候读取到SONAME是`libcrypto.so.1.1`放到缓存里，然后再load `libDemo.so`查找依赖的时候在缓存里面就能找`libcrypto.so.1.1`