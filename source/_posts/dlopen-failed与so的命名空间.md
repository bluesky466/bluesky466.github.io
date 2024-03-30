title: dlopen failed与so的命名空间
date: 2024-03-30 13:56:50
tags:
    - 技术相关
    - Android
---

之前的[Android热更新实践](https://blog.islinjw.cn/2023/12/19/Android%E7%83%AD%E6%9B%B4%E6%96%B0%E5%AE%9E%E8%B7%B5/)里面使用替换默认ClassLoader的方式实现了热修复,但偶然发现这种方式在加载某些系统so库的时候出现了问题。


背景是我们的某个功能依赖了我司开发的XXX.so这个系统so,这个XXX.so依赖了libandroid\_runtime.so,而libandroid\_runtime.so又依赖了libandroidicu.so:

{% plantuml %}
rectangle XXX.so 
rectangle libandroid_runtime.so
rectangle libandroidicu.so

XXX.so -down-> libandroid_runtime.so
libandroid_runtime.so -down-> libandroidicu.so
{% endplantuml %}

然后在应用层调用`System.loadLibrary`去加载XXX.so的时候报了下面的异常:

```
03-26 02:27:07.385  3671  3695 E Demo: err : java.lang.UnsatisfiedLinkError: dlopen failed: library "libandroidicu.so" not found: needed by /system/lib64/libandroid_runtime.so in namespace classloader-namespace
03-26 02:27:07.385  3671  3695 E Demo:       at java.lang.Runtime.loadLibrary0(Runtime.java:1077)
03-26 02:27:07.385  3671  3695 E Demo:       at java.lang.Runtime.loadLibrary0(Runtime.java:998)
03-26 02:27:07.385  3671  3695 E Demo:       at java.lang.System.loadLibrary(System.java:1661)
```

# so加载

从报错来看是libandroidicu.so在classloader-namespace这个命名空间里面不可访问。[原生库的命名空间](https://source.android.com/docs/core/permissions/namespaces_libraries?hl=zh-cn)是安卓7.0引入的东西,目的在于限制native层私有api的访问。

从文档上并不能直接定位到我们的问题,但是从触发异常的条件"默认的ClassLoader去加载这个so是没有问题的,用我们热修复的ClassLoader去加载就会出现异常"来看,

问题原因应该就是出在我们热修复的ClassLoader的对应的命名空间没有权限去加载libandroidicu.so了。

我们先用find命令看看libandroidicu.so在系统的什么目录:

```
console:/ # find . -name libandroidicu.so 2> /dev/null
./apex/com.android.i18n/lib/libandroidicu.so
./apex/com.android.i18n/lib64/libandroidicu.so
./system/apex/com.android.i18n/lib/libandroidicu.so
./system/apex/com.android.i18n/lib64/libandroidicu.so
```

的确不在/system/lib64下面,应用加载不到它是合理的。但我好奇的是默认的ClassLoader又是怎么加载到的？

从错误堆栈的Runtime.loadLibrary0一路往jni追踪可以发现so实际并不是由ClassLoader去加载的,而是通过ClassLoader找到了对应的NativeLoaderNamespace,然后用NativeLoaderNamespace::Load去加载的:

{% plantuml %}
component "Runtime.loadLibrary(String libname)" as loadLibrary1
component "Runtime.loadLibrary0(Class<?> fromClass, String libname)" as loadLibrary2
component "loadLibrary0(ClassLoader loader, Class<?> callerClass, String libname)" as loadLibrary3
component JVM_NativeLoad
component "JavaVMExt::LoadNativeLibrary" as LoadNativeLibrary
component "android::OpenNativeLibrary" as OpenNativeLibrary
component "NativeLoaderNamespace::Load" as Load
loadLibrary1 -down-> loadLibrary2 : 查找是哪个类调用的Runtime.loadLibrary
loadLibrary2 -down-> loadLibrary3 : 查找类对应的ClassLoader
loadLibrary3 -down-> JVM_NativeLoad : jni调用到native层
JVM_NativeLoad -down-> LoadNativeLibrary
LoadNativeLibrary -down-> OpenNativeLibrary
OpenNativeLibrary -down-> Load : 从g_namespaces里面查找ClassLoader对应的NativeLoaderNamespace\n并使用NativeLoaderNamespace去加载so
{% endplantuml %}

具体的代码调用如下:

```java

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r39:libcore/ojluni/src/main/java/java/lang/Runtime.java
public class Runtime {
    ...
    public void loadLibrary(String libname) {
        loadLibrary0(Reflection.getCallerClass(), libname);
    }
    
    void loadLibrary0(Class<?> fromClass, String libname) {
        ClassLoader classLoader = ClassLoader.getClassLoader(fromClass);
        loadLibrary0(classLoader, fromClass, libname);
    }
    
    private synchronized void loadLibrary0(ClassLoader loader, Class<?> callerClass, String libname) {
        ...
        nativeLoad(filename, loader);
        ...
    }

    private static String nativeLoad(String filename, ClassLoader loader) {
        return nativeLoad(filename, loader, null);
    }

    private static native String nativeLoad(String filename, ClassLoader loader, Class<?> caller);
    ...
}
```
```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r39:libcore/ojluni/src/main/native/Runtime.c
JNIEXPORT jstring JNICALL
Runtime_nativeLoad(JNIEnv* env, jclass ignored, jstring javaFilename,
                   jobject javaLoader, jclass caller)
{
    return JVM_NativeLoad(env, javaFilename, javaLoader, caller);
}

static JNINativeMethod gMethods[] = {
  FAST_NATIVE_METHOD(Runtime, freeMemory, "()J"),
  FAST_NATIVE_METHOD(Runtime, totalMemory, "()J"),
  FAST_NATIVE_METHOD(Runtime, maxMemory, "()J"),
  NATIVE_METHOD(Runtime, nativeGc, "()V"),
  NATIVE_METHOD(Runtime, nativeExit, "(I)V"),
  NATIVE_METHOD(Runtime, nativeLoad,
                "(Ljava/lang/String;Ljava/lang/ClassLoader;Ljava/lang/Class;)"
                    "Ljava/lang/String;"),
};

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r39:art/openjdkjvm/OpenjdkJvm.cc
JNIEXPORT jstring JVM_NativeLoad(JNIEnv* env,
                                 jstring javaFilename,
                                 jobject javaLoader,
                                 jclass caller) {
  	...
    bool success = vm->LoadNativeLibrary(env,
                                         filename.c_str(),
                                         javaLoader,
                                         caller,
                                         &error_msg);
    ...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r39:art/runtime/jni/java_vm_ext.cc
bool JavaVMExt::LoadNativeLibrary(JNIEnv* env,
                                  const std::string& path,
                                  jobject class_loader,
                                  jclass caller_class,
                                  std::string* error_msg) {
	...
	void* handle = android::OpenNativeLibrary(
      env,
      runtime_->GetTargetSdkVersion(),
      path_str,
      class_loader,
      (caller_location.empty() ? nullptr : caller_location.c_str()),
      library_path.get(),
      &needs_native_bridge,
      &nativeloader_error_msg);
	...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r39:art/libnativeloader/native_loader.cpp
LibraryNamespaces* g_namespaces = new LibraryNamespaces;

void* OpenNativeLibrary(JNIEnv* env, int32_t target_sdk_version, const char* path,
                        jobject class_loader, const char* caller_location, jstring library_path,
                        bool* needs_native_bridge, char** error_msg) {
  ...
  std::lock_guard<std::mutex> guard(g_namespaces_mutex);
  NativeLoaderNamespace* ns;

  if ((ns = g_namespaces->FindNamespaceByClassLoader(env, class_loader)) == nullptr) {
    // This is the case where the classloader was not created by ApplicationLoaders
    // In this case we create an isolated not-shared namespace for it.
    Result<NativeLoaderNamespace*> isolated_ns =
        CreateClassLoaderNamespaceLocked(env,
                                         target_sdk_version,
                                         class_loader,
                                         /*is_shared=*/false,
                                         /*dex_path=*/nullptr,
                                         library_path,
                                         /*permitted_path=*/nullptr,
                                         /*uses_library_list=*/nullptr);
    if (!isolated_ns.ok()) {
      *error_msg = strdup(isolated_ns.error().message().c_str());
      return nullptr;
    } else {
      ns = *isolated_ns;
    }
  }

  return OpenNativeLibraryInNamespace(ns, path, needs_native_bridge, error_msg);
  ...
}

void* OpenNativeLibraryInNamespace(NativeLoaderNamespace* ns, const char* path,
                                   bool* needs_native_bridge, char** error_msg) {
  auto handle = ns->Load(path);
  ...
  return handle.ok() ? *handle : nullptr;
}
```

从`ld.config.txt`可以看到不同的namespace有不同的search paths:

```
additional.namespaces = com_android_adbd,com_android_art,com_android_conscrypt,com_android_cronet,com_android_i18n,com_android_media,com_android_neuralnetworks,com_android_os_statsd,com_android_resolv,com_android_runtime,com_product_service1,product,rs,sphal,vndk,vndk_product
...
namespace.default.search.paths = /system/${LIB}
namespace.default.search.paths += /system_ext/${LIB}
...
namespace.com_android_i18n.search.paths = /apex/com.android.i18n/${LIB}
```

# public libs

g\_namespaces负责NativeLoaderNamespace的创建和缓存,NativeLoaderNamespace也是通过g\_namespaces去Create出来的:

```c++
Result<NativeLoaderNamespace*> CreateClassLoaderNamespaceLocked(JNIEnv* env,
                                                                int32_t target_sdk_version,
                                                                jobject class_loader,
                                                                bool is_shared,
                                                                jstring dex_path,
                                                                jstring library_path,
                                                                jstring permitted_path,
                                                                jstring uses_library_list)
    REQUIRES(g_namespaces_mutex) {
  Result<NativeLoaderNamespace*> ns = g_namespaces->Create(env,
                                                           target_sdk_version,
                                                           class_loader,
                                                           is_shared,
                                                           dex_path,
                                                           library_path,
                                                           permitted_path,
                                                           uses_library_list);
  ...
  return ns;
}
```

我们看看g\_namespaces的Create方法创建NativeLoaderNamespace干了些啥:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r39:art/libnativeloader/library_namespaces.cpp
Result<NativeLoaderNamespace*> LibraryNamespaces::Create(JNIEnv* env, uint32_t target_sdk_version,
                                                         jobject class_loader, bool is_shared,
                                                         jstring dex_path_j,
                                                         jstring java_library_path,
                                                         jstring java_permitted_path,
                                                         jstring uses_library_list) {
  ...
  auto app_ns = NativeLoaderNamespace::Create(
      namespace_name, library_path, permitted_path, parent_ns, is_shared,
      target_sdk_version < 24 /* is_exempt_list_enabled */, also_used_as_anonymous);
  ...
  for (const auto&[apex_ns_name, public_libs] : apex_public_libraries()) {
    auto ns = NativeLoaderNamespace::GetExportedNamespace(apex_ns_name, is_bridged);
    // Even if APEX namespace is visible, it may not be available to bridged.
    if (ns.ok()) {
      linked = app_ns->Link(&ns.value(), public_libs);
      if (!linked.ok()) {
        return linked.error();
      }
    }
  }
  ...
  // 缓存并返回app_ns
}
```

可以看到它在创建出app\_ns之后会遍历apex\_public\_libraries去链接apex里面的公共库:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r39:art/libnativeloader/public_libraries.cpp
constexpr const char* kApexLibrariesConfigFile = "/linkerconfig/apex.libraries.config.txt";

const std::map<std::string, std::string>& apex_public_libraries() {
  static std::map<std::string, std::string> public_libraries = InitApexLibraries("public");
  return public_libraries;
}

static std::map<std::string, std::string> InitApexLibraries(const std::string& tag) {
  std::string file_content;
  if (!base::ReadFileToString(kApexLibrariesConfigFile, &file_content)) {
    // config is optional
    return {};
  }
  auto config = ParseApexLibrariesConfig(file_content, tag);
  if (!config.ok()) {
    LOG_ALWAYS_FATAL("%s: %s", kApexLibrariesConfigFile, config.error().message().c_str());
    return {};
  }
  return *config;
}
```

在实机的`/linkerconfig/apex.libraries.config.txt`下可以看到:

```
jni com_android_appsearch libicing.so
public com_android_art libnativehelper.so
jni com_android_btservices libbluetooth_jni.so
jni com_android_conscrypt libjavacrypto.so
public com_android_i18n libicui18n.so:libicuuc.so:libicu.so
public com_android_neuralnetworks libneuralnetworks.so
jni com_android_os_statsd libstats_jni.so
jni com_android_tethering libframework-connectivity-jni.so:libframework-connectivity-tiramisu-jni.so:libandroid_net_connectivity_com_android_net_module_util_jni.so:libservice-connectivity.so
jni com_android_uwb libuwb_uci_jni_rust.so
```

com\_android\_i18n的`libicui18n.so`、`libicuuc.so`、`libicu.so`是公共的可以直接访问会被链接到app\_ns允许访问。而报错的libandroidicu.so的确没有在public里面,所以不会被链接,于是无法访问。

# shared libs

实际上除了这个public libs配置之外,还有个shared libs的配置可以用于配置NativeLoaderNamespace对哪些NativeLoaderNamespace暴露哪些so。

详细的规则可以参考[链接器命名空间](https://source.android.com/docs/core/architecture/vndk/linker-namespace?hl=zh-cn)的文档

具体的原理我们可以继续往下追一层看看NativeLoaderNamespace::Create是怎么创建app\_ns的:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r39:art/libnativeloader/native_loader_namespace.cpp
Result<NativeLoaderNamespace> NativeLoaderNamespace::Create(
    const std::string& name, const std::string& search_paths, const std::string& permitted_paths,
    const NativeLoaderNamespace* parent, bool is_shared, bool is_exempt_list_enabled,
    bool also_used_as_anonymous) {
  ...
  // All namespaces for apps are isolated
  uint64_t type = ANDROID_NAMESPACE_TYPE_ISOLATED;

  ...
  if (is_shared) {
    type |= ANDROID_NAMESPACE_TYPE_SHARED;
  }
  ...
  android_namespace_t* raw =
    android_create_namespace(name.c_str(), nullptr, search_paths.c_str(), type,
                              permitted_paths.c_str(), effective_parent.ToRawAndroidNamespace());
  if (raw != nullptr) {
    return NativeLoaderNamespace(name, raw);
  }
  ...
}
```

可以看到is\_shared为true的时候会给type添加一个ANDROID\_NAMESPACE\_TYPE\_SHARED的flag,这个flag最终在create\_namespace会判断:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r39:bionic/linker/linker.cpp
static android_namespace_t* g_anonymous_namespace = &g_default_namespace;
...
android_namespace_t* create_namespace(const void* caller_addr,
                                      const char* name,
                                      const char* ld_library_path,
                                      const char* default_library_path,
                                      uint64_t type,
                                      const char* permitted_when_isolated_path,
                                      android_namespace_t* parent_namespace) {
  if (parent_namespace == nullptr) {
    // if parent_namespace is nullptr -> set it to the caller namespace
    soinfo* caller_soinfo = find_containing_library(caller_addr);

    parent_namespace = caller_soinfo != nullptr ?
                       caller_soinfo->get_primary_namespace() :
                       g_anonymous_namespace;
  }

  ProtectedDataGuard guard;
  std::vector<std::string> ld_library_paths;
  std::vector<std::string> default_library_paths;
  std::vector<std::string> permitted_paths;
  ...
  android_namespace_t* ns = new (g_namespace_allocator.alloc()) android_namespace_t();
  ...
  if ((type & ANDROID_NAMESPACE_TYPE_SHARED) != 0) {
    // append parent namespace paths.
    std::copy(parent_namespace->get_ld_library_paths().begin(),
              parent_namespace->get_ld_library_paths().end(),
              back_inserter(ld_library_paths));

    std::copy(parent_namespace->get_default_library_paths().begin(),
              parent_namespace->get_default_library_paths().end(),
              back_inserter(default_library_paths));

    std::copy(parent_namespace->get_permitted_paths().begin(),
              parent_namespace->get_permitted_paths().end(),
              back_inserter(permitted_paths));

    // If shared - clone the parent namespace
    add_soinfos_to_namespace(parent_namespace->soinfo_list(), ns);
    // and copy parent namespace links
    for (auto& link : parent_namespace->linked_namespaces()) {
      ns->add_linked_namespace(link.linked_namespace(), link.shared_lib_sonames(),
                               link.allow_all_shared_libs());
    }
  } else {
    // If not shared - copy only the shared group
    add_soinfos_to_namespace(parent_namespace->get_shared_group(), ns);
  }
  
  ns->set_ld_library_paths(std::move(ld_library_paths));
  ns->set_default_library_paths(std::move(default_library_paths));
  ns->set_permitted_paths(std::move(permitted_paths));
  ...
}
```

可以看到添加了ANDROID\_NAMESPACE\_TYPE\_SHARED这个flag的话会添加父namespace的所有so和链接父namespace连接过的namespace。而如果为false则只会添加父namespace的shared so。

系统默认创建的Classloader对应的namespace的parent\_namespace为null,于是会把parent\_namespace设置成默认的"default"这个namespace。

而从`/linkerconfig/ld.config.txt`里面可以看到"default"这个namespace是链接了com\_android\_i18n这个namespace,可以从里面访问`libandroidicu.so`:

```
...
namespace.default.links = com_android_adbd,com_android_i18n,default,com_android_art,com_android_resolv,com_android_tethering,com_android_neural
namespace.default.link.com_android_adbd.shared_libs = libadb_pairing_auth.so
namespace.default.link.com_android_adbd.shared_libs += libadb_pairing_connection.so
namespace.default.link.com_android_adbd.shared_libs += libadb_pairing_server.so
namespace.default.link.com_android_i18n.shared_libs = libandroidicu.so
namespace.default.link.com_android_i18n.shared_libs += libicu.so
namespace.default.link.com_android_i18n.shared_libs += libicui18n.so
namespace.default.link.com_android_i18n.shared_libs += libicuuc.so
...
```

所以默认的ClassLoader创建NativeLoaderNamespace的时候is\_shared是true可以加载到`libandroidicu.so`

而我们自定义的ClassLoader创建NativeLoaderNamespace的时候is\_shared是false没有继承"default"这个parent\_namepsace的links配置无法加载`libandroidicu.so`

另外也可以看到默认的Classloader对应的namespace会连接com\_android\_i18n这个命名空间两次

第一次在`create\_namespace`函数里由于`ANDROID_NAMESPACE_TYPE_SHARED`继承了`defalut`这个parent\_namespace的links配置能访问`/linkerconfig/ld.config.txt`里的`namespace.default.link.com_android_i18n.shared_libs`配置的shared so,

第二次则是在LibraryNamespaces::Create里面赌钱`/linkerconfig/apex.libraries.config.txt`里`public com_android_i18n`配置链接`com_android_i18n`的public so。

[链接器命名空间](https://source.android.com/docs/core/architecture/vndk/linker-namespace?hl=zh-cn)这个文档里也有提到:

> 此属性与 public.libraries.txt 文件在底层实现上是相同的。这两种机制都通过使用库名称过滤器指定链接的方式来控制导入的共享库。

# is shared

从前面CreateClassLoaderNamespaceLocked的传参来看我们的自定义Classloader创建namespace的时候is\_shared的确为false:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r39:art/libnativeloader/native_loader.cpp
LibraryNamespaces* g_namespaces = new LibraryNamespaces;

void* OpenNativeLibrary(JNIEnv* env, int32_t target_sdk_version, const char* path,
                        jobject class_loader, const char* caller_location, jstring library_path,
                        bool* needs_native_bridge, char** error_msg) {
  ...
  std::lock_guard<std::mutex> guard(g_namespaces_mutex);
  NativeLoaderNamespace* ns;

  if ((ns = g_namespaces->FindNamespaceByClassLoader(env, class_loader)) == nullptr) {
    // This is the case where the classloader was not created by ApplicationLoaders
    // In this case we create an isolated not-shared namespace for it.
    Result<NativeLoaderNamespace*> isolated_ns =
        CreateClassLoaderNamespaceLocked(env,
                                         target_sdk_version,
                                         class_loader,
                                         /*is_shared=*/false,
                                         /*dex_path=*/nullptr,
                                         library_path,
                                         /*permitted_path=*/nullptr,
                                         /*uses_library_list=*/nullptr);
    if (!isolated_ns.ok()) {
      *error_msg = strdup(isolated_ns.error().message().c_str());
      return nullptr;
    } else {
      ns = *isolated_ns;
    }
  }

  return OpenNativeLibraryInNamespace(ns, path, needs_native_bridge, error_msg);
  ...
}
```

那默认的ClassLoader的is\_shared是怎么设置成true的呢?从LoadedApk代码里面可以看到系统app最终就会调用ClassLoaderFactory.createClassLoader在里面创建Classloader的同时调用createClassloaderNamespace创建namespace,传入的is\_shared为isBundledApp即true:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r39:frameworks/base/core/java/android/app/LoadedApk.java
public ClassLoader getClassLoader() {
    synchronized (mLock) {
        if (mClassLoader == null) {
            createOrUpdateClassLoaderLocked(null /*addedPaths*/);
        }
        return mClassLoader;
    }
}

private void createOrUpdateClassLoaderLocked(List<String> addedPaths) {
    ...
    boolean isBundledApp = mApplicationInfo.isSystemApp()
            && !mApplicationInfo.isUpdatedSystemApp();

    ...
    mDefaultClassLoader = ApplicationLoaders.getDefault().getClassLoaderWithSharedLibraries(
        zip, mApplicationInfo.targetSdkVersion, isBundledApp, librarySearchPath,
        libraryPermittedPath, mBaseClassLoader,
        mApplicationInfo.classLoaderName, sharedLibraries.first, nativeSharedLibraries,
        sharedLibraries.second);

    mAppComponentFactory = createAppFactory(mApplicationInfo, mDefaultClassLoader);
    ...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r39:frameworks/base/core/java/android/app/ApplicationLoaders.java
ClassLoader getClassLoaderWithSharedLibraries(
        String zip, int targetSdkVersion, boolean isBundled,
        String librarySearchPath, String libraryPermittedPath,
        ClassLoader parent, String classLoaderName,
        List<ClassLoader> sharedLibraries, List<String> nativeSharedLibraries,
        List<ClassLoader> sharedLibrariesLoadedAfterApp) {
    // For normal usage the cache key used is the same as the zip path.
    return getClassLoader(zip, targetSdkVersion, isBundled, librarySearchPath,
                          libraryPermittedPath, parent, zip, classLoaderName, sharedLibraries,
                          nativeSharedLibraries, sharedLibrariesLoadedAfterApp);
}


ClassLoader getClassLoaderWithSharedLibraries(
        String zip, int targetSdkVersion, boolean isBundled,
        String librarySearchPath, String libraryPermittedPath,
        ClassLoader parent, String classLoaderName,
        List<ClassLoader> sharedLibraries, List<String> nativeSharedLibraries,
        List<ClassLoader> sharedLibrariesLoadedAfterApp) {
    // For normal usage the cache key used is the same as the zip path.
    return getClassLoader(zip, targetSdkVersion, isBundled, librarySearchPath,
                          libraryPermittedPath, parent, zip, classLoaderName, sharedLibraries,
                          nativeSharedLibraries, sharedLibrariesLoadedAfterApp);
}

rivate ClassLoader getClassLoader(String zip, int targetSdkVersion, boolean isBundled,
                                   String librarySearchPath, String libraryPermittedPath,
                                   ClassLoader parent, String cacheKey,
                                   String classLoaderName, List<ClassLoader> sharedLibraries,
                                   List<String> nativeSharedLibraries,
                                   List<ClassLoader> sharedLibrariesLoadedAfterApp) {
    ...
    ClassLoader classloader = ClassLoaderFactory.createClassLoader(
                        zip,  librarySearchPath, libraryPermittedPath, parent,
                        targetSdkVersion, isBundled, classLoaderName, sharedLibraries,
                        nativeSharedLibraries, sharedLibrariesLoadedAfterApp);
    ...
    return loader;
    ...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r39:frameworks/base/core/java/com/android/internal/os/ClassLoaderFactory.java
public static ClassLoader createClassLoader(String dexPath,
    String librarySearchPath, String libraryPermittedPath, ClassLoader parent,
    int targetSdkVersion, boolean isNamespaceShared, String classLoaderName,
    List<ClassLoader> sharedLibraries, List<String> nativeSharedLibraries,
    List<ClassLoader> sharedLibrariesAfter) {

final ClassLoader classLoader = createClassLoader(dexPath, librarySearchPath, parent,
        classLoaderName, sharedLibraries, sharedLibrariesAfter);
  ...
  String errorMessage = createClassloaderNamespace(classLoader,
                                                   targetSdkVersion,
                                                   librarySearchPath,
                                                   libraryPermittedPath,
                                                   isNamespaceShared,
                                                   dexPath,
                                                   sonameList);
  ...
  return classLoader;
}
```

# 解决方案

## 方法1

由于ClassLoaderFactory.createClassloaderNamespace是private的不能在外部调用,所以解决自定义classloader找不到libandroidicu.so的方法就是不要自己直接new ClassLoader,而是调用ClassLoaderFactory.createClassLoader去创建,传入isNamespaceShared为true。

## 方法2

由于系统默认的classloader对应的namespace已经加载了libandroid\_runtime.so,如果将我们自定义的classloader的父classloader设置成系统默认的classloder,则自定义classloader对应的namespace的parent\_namespace也会指向默认classloader的namespace。

然后这个namespace已经加载了libandroid\_runtime.so,于是在后面的`add_soinfos_to_namespace(parent_namespace->get_shared_group(), ns);`里面就能直接使用已经加载好的shared so libandroid\_runtime.so：

{% plantuml %}
component defalut
component "classloader-namespace-shared" as NameSpaceShared
component "classloader-namespace" as NameSpace
component 系统默认ClassLoader
component 自定义ClassLoader
component libandroid_runtime.so
component libandroidicu.so

defalut <-- NameSpaceShared : parent namespace
NameSpaceShared <-- NameSpace : parent namespace

系统默认ClassLoader ..right..> NameSpaceShared
自定义ClassLoader ..right..> NameSpace

系统默认ClassLoader <-- 自定义ClassLoader : parent ClassLoader

NameSpaceShared ..right..> libandroid_runtime.so : 已经load

libandroidicu.so <.. libandroid_runtime.so : 依赖

defalut ..right..> libandroidicu.so : ld.config.txt的links配置允许Load

NameSpaceShared ..up..> libandroidicu.so : 由于is_shared true\n继承default的links配置
NameSpace .up.> libandroid_runtime.so : is_shared false也会调用add_soinfos_to_namespace(parent_namespace->get_shared_group(), ns)\n可以直接使用parent namespace已经Load的shared so
{% endplantuml %}

使用`setprop debug.ld.all dlopen,dlerror`命令打开全部linker打印也可以看到libandroid\_runtime.so Already loaded的日志:

``
03-29 04:36:10.024  6046  6069 D linker  : find_library_internal(ns=classloader-namespace, task=libandroid_runtime.so): Already loaded (by sona
me): /system/lib64/libandroid_runtime.so`
``

但是当父classloader设置成系统默认的classloader之后由于双亲委托的机制,会先从父classloader去加载class,达不到热修复的需求。

于是我们需要修改自定义classloader的loadClass打破双亲委托机制,先自己去加载class,加载不到再让父classloder去加载:

```java
@Override
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    //先从自己查找,找不到再从父classloader查找,实现热修复
    Class<?> c = null;
    try {
        c = findClass(name);
    } catch (ClassNotFoundException e) {
        // ignore
    }
    if (c == null) {
        c = getParent().loadClass(name);
    }
    return c;
}
```