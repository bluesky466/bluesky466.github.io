title: 源码阅读计划 - ARouter
date: 2021-03-18 23:05:32
tags:
    - 技术相关
    - Android
---

# 初始化

ARouter的源码相对来讲还是比较简单易懂的，我们先从初始化部分的逻辑开始看。它的初始化代码只有一行，一般放在Application.onCreate()中:

```java
ARouter.init(this)
```

我们最追进去可以看到它将实际的初始化逻辑委托给\_ARouter这个类去处理:

```java
public static void init(Application application) {
    if (!hasInit) {
        ...
        hasInit = _ARouter.init(application);

        if (hasInit) {
            _ARouter.afterInit();
        }
        ...
    }
}
```

继续看\_ARouter.init又会发现它又将初始化的逻辑放到LogisticsCenter.init:

```java
protected static synchronized boolean init(Application application) {
        mContext = application;
        LogisticsCenter.init(mContext, executor);
        logger.info(Consts.TAG, "ARouter init success!");
        hasInit = true;
        mHandler = new Handler(Looper.getMainLooper());

        return true;
}
```

最后来的LogisticsCenter.init就能看的真正的初始化逻辑:

```java
public synchronized static void init(Context context, ThreadPoolExecutor tpe) throws HandlerException {
    ...
    loadRouterMap();
    if (registerByPlugin) {
        logger.info(TAG, "Load router map by arouter-auto-register plugin.");
    } else {
        // 从dex中加载路由表,耗时操作
    }
    ...
}
```

一般情况下我们都会使用gradle插件去生成注册代码以加快启动速度，loadRouterMap()方法里面就是这些生成的代码，注册完成之后registerByPlugin被设置成true，不会跑到下面的耗时操作中。这里我想先讲讲这个耗时操作到底有多耗时:

```java
Set<String> routerMap;

if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
    routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
    if (!routerMap.isEmpty()) {
        context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE)
                .edit()
                .putStringSet(AROUTER_SP_KEY_MAP, routerMap)
                .apply();
    }

    PackageUtils.updateVersion(context);
} else {
    routerMap = new HashSet<>(context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE)
                        .getStringSet(AROUTER_SP_KEY_MAP, new HashSet<String>()));
}

for (String className : routerMap) {
    if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
        ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
    } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
        ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);
    } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
        ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
    }
}
```

上面是我删除多余注释之后的代码，可以看到其实它的流程并不复杂。先判断是否为debug模式或者版本是否有更新，如果是就使用ClassUtils.getFileNameByPackageName查找下面这些类:

- com.alibaba.android.arouter.routes.ARouter\$\$Root\$\$XXX
- com.alibaba.android.arouter.routes.ARouter\$\$Interceptors\$\$XXX
- com.alibaba.android.arouter.routes.ARouter\$\$Providers\$\$XXX

查找到的话将他们保存搭配sp中，避免每次启动都需要查找，下一次直接从sp中读取即可。

然后就会利用反射机制创建这些类的实例并且调用loadInto方法，将路由加载到Warehouse的对应map中。

ClassUtils.getFileNameByPackageName很暴力，直接开启线程去读取dex文件进行类的遍历:

```java
public static Set<String> getFileNameByPackageName(Context context, final String packageName) throws PackageManager.NameNotFoundException, IOException, InterruptedException {
    final Set<String> classNames = new HashSet<>();

    List<String> paths = getSourcePaths(context);
    final CountDownLatch parserCtl = new CountDownLatch(paths.size());

    for (final String path : paths) {
        DefaultPoolExecutor.getInstance().execute(new Runnable() {
            @Override
            public void run() {
                ...
                if (path.endsWith(EXTRACTED_SUFFIX)) {
                    dexfile = DexFile.loadDex(path, path + ".tmp", 0);
                } else {
                    dexfile = new DexFile(path);
                }

                Enumeration<String> dexEntries = dexfile.entries();
                while (dexEntries.hasMoreElements()) {
                    String className = dexEntries.nextElement();
                    if (className.startsWith(packageName)) {
                        classNames.add(className);
                    }
                }
                ...
                parserCtl.countDown();
                
            }
        });
    }

    parserCtl.await();
    return classNames;
}
```

getSourcePaths方法是查找这个app的所有dex文件的路径，我就不展开了。可以看出来这个流程需要遍历该app的所有dex的所有类去匹配类名，的确十分耗时。

## arouter-register

于是乎arouter提供了arouter-register用于优化初始化速度:

1. 在根目录的build.gradle加上插件的classpath

```groovy
buildscript {
    dependencies {
        classpath "com.alibaba:arouter-register:1.0.2"
    }
}
```

2. 在app目录下的build.gradle使用插件

```groovy
apply plugin: 'com.alibaba.arouter'
```

这个插件会在编译的时候使用用ASM向loadRouterMap方法插入注册代码:

```java
private static void loadRouterMap() {
  registerByPlugin = false;
  //auto generate register code by gradle plugin: arouter-auto-register
  // looks like below:
  // registerRouteRoot(new ARouter..Root..modulejava());
  // registerRouteRoot(new ARouter..Root..modulekotlin());
}
```

我们可以使用jadx反编译生成的apk看到插入的代码:

{% img /源码阅读计划_ARouter/1.png %}

register方法会根据类型调用不同的注册方法，但是它们都会使用markRegisteredByPlugin方法将registerByPlugin设置成true，以跳过dex的遍历:

```java
private static void register(String className) {
    ...
    Class<?> clazz = Class.forName(className);
    Object obj = clazz.getConstructor().newInstance();
    if (obj instanceof IRouteRoot) {
        registerRouteRoot((IRouteRoot) obj);
    } else if (obj instanceof IProviderGroup) {
        registerProvider((IProviderGroup) obj);
    } else if (obj instanceof IInterceptorGroup) {
        registerInterceptor((IInterceptorGroup) obj);
    } 
    ...
}
private static void registerRouteRoot(IRouteRoot routeRoot) {
    markRegisteredByPlugin();
    if (routeRoot != null) {
        routeRoot.loadInto(Warehouse.groupsIndex);
    }
}
private static void registerProvider(IProviderGroup providerGroup) {
    markRegisteredByPlugin();
    if (providerGroup != null) {
        providerGroup.loadInto(Warehouse.providersIndex);
    }
}
private static void registerInterceptor(IInterceptorGroup interceptorGroup) {
    markRegisteredByPlugin();
    if (interceptorGroup != null) {
        interceptorGroup.loadInto(Warehouse.interceptorsIndex);
    }
}
private static void markRegisteredByPlugin() {
    if (!registerByPlugin) {
        registerByPlugin = true;
    }
}
```

## 路由的注册

在上面的代码中我们知道ARouter在初始化的时候会调用IRouteRoot、IProviderGroup、IInterceptorGroup的loadInto方法将路由加载到Warehouse中。那这些类又是怎么知道我们代码里面用@Router注册的路径的?例如下面的MainActivity:

```kotlin
@Route(path = "/activity/home")
class MainActivity : BaseActivity() {
    ...
}
```

其实这些类都是由ARouter在编译的时候生成的:

{% img /源码阅读计划_ARouter/2.png %}

像ARouter\$\$Root\$\$app这个分组类就注册了我们的页面路由，而ARouter\$\$Root\$\$provider这个分组类就注册了我们的provider路由:

```java
public class ARouter$$Root$$app implements IRouteRoot {
  @Override
  public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
    routes.put("activity", ARouter$$Group$$activity.class);
    routes.put("provider", ARouter$$Group$$provider.class);
  }
}

public class ARouter$$Group$$activity implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/activity/date", RouteMeta.build(RouteType.ACTIVITY, DateActivity.class, "/activity/date", "activity", null, -1, -2147483648));
    atlas.put("/activity/edit", RouteMeta.build(RouteType.ACTIVITY, EditActivity.class, "/activity/edit", "activity", new java.util.HashMap<String, Integer>(){{put("taskGroupId", 4); }}, -1, -2147483648));
    atlas.put("/activity/home", RouteMeta.build(RouteType.ACTIVITY, MainActivity.class, "/activity/home", "activity", null, -1, -2147483648));
    atlas.put("/activity/login", RouteMeta.build(RouteType.ACTIVITY, LoginActivity.class, "/activity/login", "activity", null, -1, -2147483648));
    atlas.put("/activity/register", RouteMeta.build(RouteType.ACTIVITY, RegisterActivity.class, "/activity/register", "activity", null, -1, -2147483648));
    atlas.put("/activity/welcome", RouteMeta.build(RouteType.ACTIVITY, WelcomeActivity.class, "/activity/welcome", "activity", null, -1, -2147483648));
  }
}

public class ARouter$$Group$$provider implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/provider/demoProvider", RouteMeta.build(RouteType.PROVIDER, DemoProvider.class, "/provider/demoprovider", "provider", null, -1, -2147483648));
  }
}
```

ARouter\$\$Root\$\$app的后缀"app"字符串是由gradle里面配置的AROUTER\_MODULE\_NAME决定的,一般我们设置成module的名字，这样不同module生成的类就不会重名:

```groovy
javaCompileOptions {
    annotationProcessorOptions {
        arguments = [
                AROUTER_MODULE_NAME  : project.getName(),
        ]
    }
}
```

从上面可以看到IRouteRoot并不会直接将所有的路由信息直接加载进去，而是加载分组信息IRouteGroup。IRouteGroup会在navigation查找的时候再去加载对应分组的路由表。

## 路由分组

我们从上面的截图可以看到ARouter\$\$Group\$\$activity、ARouter\$\$Group\$\$provider这样的类，它就是activity和provider这两个分组的路由表注册逻辑。由于路由表可能会比较大，一次全部加载可能影响启动耗时，所以ARouter设计了路由分组的概念，在需要的时候才去加载。默认path的第一级就是分组，例如下面的activity:

```kotlin
@Route(path = "/activity/home")
class MainActivity : BaseActivity() {
    ...
}
```

当然你也可以主动指定分组:

```kotlin
@Route(path = Router.ActivityHome.PATH, group = "xxx")
class MainActivity : BaseActivity() {
    ...
}
```

于是@Router注解就会生成ARouter\$\$Group\$\$xxx这样的类去管理该分组下的路由表的加载。

## 拦截器初始化

由于拦截器不需要我们等主动去获取，在navigation的时候就会自动调用，所以ARouter在初始化的时候就会顺便将拦截器给初始化了。在ARouter.init的后面会调用\_ARouter.afterInit去初始化拦截器

```java
//ARouter.java
public static void init(Application application) {
    if (!hasInit) {
        ...
        hasInit = _ARouter.init(application);

        if (hasInit) {
            _ARouter.afterInit();
        }
        ...
    }
}

//_ARouter.java
static void afterInit() {
    interceptorService = (InterceptorService) ARouter.getInstance().build("/arouter/service/interceptor").navigation();
}
```

在afterInit里面会去创建InterceptorServiceImpl，它是一个IProvider，会在它的init方法里实例化拦截器并且添加到Warehouse.interceptors列表里面提供页面路由的时候调用:

```java
@Route(path = "/arouter/service/interceptor")
public class InterceptorServiceImpl implements InterceptorService {
    //...
    @Override
    public void init(final Context context) {
        LogisticsCenter.executor.execute(new Runnable() {
            @Override
            public void run() {
                //...
                for (Map.Entry<Integer, Class<? extends IInterceptor>> entry : Warehouse.interceptorsIndex.entrySet()) {
                    Class<? extends IInterceptor> interceptorClass = entry.getValue();
                    try {
                        IInterceptor iInterceptor = interceptorClass.getConstructor().newInstance();
                        iInterceptor.init(context);
                        Warehouse.interceptors.add(iInterceptor);
                    } catch (Exception ex) {
                        throw new HandlerException(TAG + "ARouter init interceptor error! name = [" + interceptorClass.getName() + "], reason = [" + ex.getMessage() + "]");
                    }
                }
                //...
            }
        });
    }
    //...
}
```

Warehouse.interceptorsIndex是一个UniqueKeyTreeMap，它在前面被扫描dex或者生成代码注册。key为拦截器的优先级，所以拦截器的优先级是不能有相同的:

```java
// Warehouse.java
static Map<Integer, Class<? extends IInterceptor>> interceptorsIndex = new UniqueKeyTreeMap<>("More than one interceptors use same priority [%s]");

//UniqueKeyTreeMap.java
public class UniqueKeyTreeMap<K, V> extends TreeMap<K, V> {
    private String tipText;

    public UniqueKeyTreeMap(String exceptionText) {
        super();

        tipText = exceptionText;
    }

    @Override
    public V put(K key, V value) {
        if (containsKey(key)) {
            throw new RuntimeException(String.format(tipText, key));
        } else {
            return super.put(key, value);
        }
    }
}
```

# navigation

使用ARouter进行页面跳转只需要下面的简单代码，build方法会分析路径，查找分组信息创建Postcard对象，然后使用Postcard对象的navigation进行跳转:

```java
ARouter.getInstance()
      .build("/activity/home")
      .navigation()
```

当然navigation不一定是跳转页面，也可能直接返回查找到的IProvider等组件，下面我们就来看看具体的实现逻辑。由于navigation最后面是跑到\_ARouter.navigation里面的，我们直接从这个方法开始分析:

```java
protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
    ...
    LogisticsCenter.completion(postcard);
    ...
    if (!postcard.isGreenChannel()) {
        interceptorService.doInterceptions(postcard, new InterceptorCallback() {
            @Override
            public void onContinue(Postcard postcard) {
                _navigation(context, postcard, requestCode, callback);
            }

            @Override
            public void onInterrupt(Throwable exception) {
                if (null != callback) {
                    callback.onInterrupt(postcard);
                }
            }
        });
    } else {
        return _navigation(context, postcard, requestCode, callback);
    }

    return null;
}
```

这里面的逻辑分三部分:

1. 使用LogisticsCenter.completion去查找路由表，将查找到的信息填充到postcard里面
2. 如果postcard不走绿色通道，就调用拦截器逻辑
3. 走绿色通道或者拦截器处理完成之后调用\_navigation进行实际的页面跳转或者返回查找到的组件

## 路由表查找

我们先看看completion查找路由表的逻辑:

```java
public synchronized static void completion(Postcard postcard) {
    ...
    RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
    if (null == routeMeta) {    // Maybe its does't exist, or didn't load.
        Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());  // Load route meta.
        if (null == groupMeta) {
            throw new NoRouteFoundException(TAG + "There is no route match the path [" + postcard.getPath() + "], in group [" + postcard.getGroup() + "]");
        } else {
            ...
            IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
            iGroupInstance.loadInto(Warehouse.routes);
            Warehouse.groupsIndex.remove(postcard.getGroup());
            ...
            completion(postcard);   // Reload
        }
    } else {
        postcard.setDestination(routeMeta.getDestination());
        postcard.setType(routeMeta.getType());
        postcard.setPriority(routeMeta.getPriority());
        postcard.setExtra(routeMeta.getExtra());

        Uri rawUri = postcard.getUri();
        if (null != rawUri) {   // Try to set params into bundle.
            ...
        }

        switch (routeMeta.getType()) {
            case PROVIDER:
                Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                IProvider instance = Warehouse.providers.get(providerMeta);
                if (null == instance) { // There's no instance of this provider
                    ...
                    provider = providerMeta.getConstructor().newInstance();
                    provider.init(mContext);
                    Warehouse.providers.put(providerMeta, provider);
                    instance = provider;
                    ...
                }
                postcard.setProvider(instance);
                postcard.greenChannel();    // Provider should skip all of interceptors
                break;
            case FRAGMENT:
                postcard.greenChannel();    // Fragment needn't interceptors
            default:
                break;
        }
    }
}
```

它会先查找Warehouse.routes这个缓存，如果找不到就先去查找分组并加载该分组的路由表到Warehouse.routes，然后递归调用自己重新查找Warehouse.routes。这次就能查找到路由信息了，于是将该path对应的组件的信息填充到postcard。

而且我们可以看到如果是Provider或者Fragment，就会走绿色通道，因为它们是不会被拦截器拦截的。

整体流程如下:

{% img /源码阅读计划_ARouter/3.png %}

## 拦截器调用原理

查找路由表成功填充信息之后如果不走绿色通道的话就会先去调用interceptorService.doInterceptions走拦截器的流程:

```java
@Override
public void doInterceptions(final Postcard postcard, final InterceptorCallback callback) {
    ...
    LogisticsCenter.executor.execute(new Runnable() {
        @Override
        public void run() {
            CancelableCountDownLatch interceptorCounter = new CancelableCountDownLatch(Warehouse.interceptors.size());
            try {
                _execute(0, interceptorCounter, postcard);
                interceptorCounter.await(postcard.getTimeout(), TimeUnit.SECONDS);
                if (interceptorCounter.getCount() > 0) {    // Cancel the navigation this time, if it hasn't return anythings.
                    callback.onInterrupt(new HandlerException("The interceptor processing timed out."));
                } else if (null != postcard.getTag()) {    // Maybe some exception in the tag.
                    callback.onInterrupt(new HandlerException(postcard.getTag().toString()));
                } else {
                    callback.onContinue(postcard);
                }
            } catch (Exception e) {
                callback.onInterrupt(e);
            }
        }
    });
    ...
}
```

首先看doInterceptions的逻辑我们可以知道它创建了一个子线程去使用\_execute方法调用拦截器，并且使用CancelableCountDownLatch等待所有拦截器执行完毕。也就是说\_execute里面可能也是存在子线程的:

```java
private static void _execute(final int index, final CancelableCountDownLatch counter, final Postcard postcard) {
    if (index < Warehouse.interceptors.size()) {
        IInterceptor iInterceptor = Warehouse.interceptors.get(index);
        iInterceptor.process(postcard, new InterceptorCallback() {
            @Override
            public void onContinue(Postcard postcard) {
                counter.countDown();
                _execute(index + 1, counter, postcard);  
            }

            @Override
            public void onInterrupt(Throwable exception) {
                postcard.setTag(null == exception ? new HandlerException("No message.") : exception.getMessage());    
                counter.cancel();
                ...
            }
        });
    }
}
```

\_execute里面的逻辑也比较容易看懂，根据传入的index参数从拦截器数组获取拦截器，并且调用拦截器的process方法，这个方法就是我们使用拦截器需要实现的接口，在内部可能会开子线程或者去到主线程弹出对话框让用户选择。当拦截逻辑处理完之后就必现要回调callback的onContinue方法或者onInterrupt，让它去到下一个拦截器或者取消整个路由流程。__所以我们这自定义拦截器的时候不要忘了调用onContinue或者onInterrupt。__

整个拦截器的流程如下:

{% img /源码阅读计划_ARouter/4.png %}

总结下就是doInterceptions里面开启子线程，设置CancelableCountDownLatch为拦截器数组大小，然后调用第一个拦截器，接着线程等待CountDown取消或者为0。拦截器调用onContinue让CountDown减一和继续调用下一个拦截器直到遍历完所有拦截器或者有拦截器调用onInterrupt。

所以我们在navigation的时候如果不是绿色通道，需要走到拦截器的话并不会立马跳转，而是会开启子线程等待拦截器处理，也就是说__拦截器是运行在子线程里面的。__

## \_navigation

走完拦截器就去到了\_navigation方法。这个方法比较简单，就是判断类型进行页面跳转或者创建返回:

```java
    private Object _navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        final Context currentContext = null == context ? mContext : context;

        switch (postcard.getType()) {
            case ACTIVITY:
                final Intent intent = new Intent(currentContext, postcard.getDestination());
                intent.putExtras(postcard.getExtras());
                int flags = postcard.getFlags();
                if (-1 != flags) {
                    intent.setFlags(flags);
                } else if (!(currentContext instanceof Activity)) {    // Non activity, need less one flag.
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                }
                String action = postcard.getAction();
                if (!TextUtils.isEmpty(action)) {
                    intent.setAction(action);
                }
                runInMainThread(new Runnable() {
                    @Override
                    public void run() {
                        startActivity(requestCode, currentContext, intent, postcard, callback);
                    }
                });
                break;
            case PROVIDER:
                return postcard.getProvider();
            case BOARDCAST:
            case CONTENT_PROVIDER:
            case FRAGMENT:
                Class fragmentMeta = postcard.getDestination();
                try {
                    Object instance = fragmentMeta.getConstructor().newInstance();
                    if (instance instanceof Fragment) {
                        ((Fragment) instance).setArguments(postcard.getExtras());
                    } else if (instance instanceof android.support.v4.app.Fragment) {
                        ((android.support.v4.app.Fragment) instance).setArguments(postcard.getExtras());
                    }

                    return instance;
                } catch (Exception ex) {
                    logger.error(Consts.TAG, "Fetch fragment instance error, " + TextUtils.formatStackTrace(ex.getStackTrace()));
                }
            case METHOD:
            case SERVICE:
            default:
                return null;
        }

        return null;
    }
```

## Provider类型查找

除了使用路径去查找Provider之外，我们还可以用Provider的类型去查找:

```kotlin
val provider = ARouter.getInstance().navigation(DemoProvider::class.java)
```

这里是通过类的Name或者SimpleName从providersIndex去查找的:

```java
protected <T> T navigation(Class<? extends T> service) {
    try {
        Postcard postcard = LogisticsCenter.buildProvider(service.getName());
        if (null == postcard) {
            postcard = LogisticsCenter.buildProvider(service.getSimpleName());
        }
        if (null == postcard) {
            return null;
        }
        LogisticsCenter.completion(postcard);
        return (T) postcard.getProvider();
    } catch (NoRouteFoundException ex) {
        return null;
    }
}

public static Postcard buildProvider(String serviceName) {
    RouteMeta meta = Warehouse.providersIndex.get(serviceName);

    if (null == meta) {
        return null;
    } else {
        return new Postcard(meta.getPath(), meta.getGroup());
    }
}
```

是我们在ARouter\$\$Provider\$\$xxx里面注册的:

{% img /源码阅读计划_ARouter/5.png %}

```java
public class ARouter$$Providers$$app implements IProviderGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> providers) {
    providers.put("me.linjw.checklist.DemoProvider", RouteMeta.build(RouteType.PROVIDER, DemoProvider.class, "/provider/demoProvider", "provider", null, -1, -2147483648));
  }
}
```

# @Autowired自动注入原理

最后我们来讲讲@Autowired自动注入。它的使用也很方便，使用@Autowired修饰想要注入的数据，然后再使用ARouter.getInstance().inject(this)进行自动注入将intent携带的数据注入:

```kotlin
@Route(path = Router.ActivityHome.PATH)
class MainActivity : BaseActivity() {
    @Autowired
    lateinit var data : String
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        ARouter.getInstance().inject(this)
        ...
    }
}
```

inject最后会根据类名查找到对应的ISyringe，并且调用它的inject实现注入:

```java
// _Arouter.java
static void inject(Object thiz) {
    AutowiredService autowiredService = ((AutowiredService) ARouter.getInstance().build("/arouter/service/autowired").navigation());
    if (null != autowiredService) {
        autowiredService.autowire(thiz);
    }
}


// AutowiredServiceImpl.java
@Override
public void autowire(Object instance) {
    doInject(instance, null);
}

private void doInject(Object instance, Class<?> parent) {
    Class<?> clazz = null == parent ? instance.getClass() : parent;

    ISyringe syringe = getSyringe(clazz);
    if (null != syringe) {
        syringe.inject(instance);
    }
    ...
}
```

这个ISyringe也是ARouter自动生成的，它不需要使用反射而是直接设置，所以__@Autowired修饰的数据不能是private的__:

```java
public class MainActivity$$ARouter$$Autowired implements ISyringe {
  private SerializationService serializationService;

  @Override
  public void inject(Object target) {
    serializationService = ARouter.getInstance().navigation(SerializationService.class);
    MainActivity substitute = (MainActivity)target;
    substitute.data = substitute.getIntent().getStringExtra("data");
  }
}
```

