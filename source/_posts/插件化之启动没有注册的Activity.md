title: 插件化之启动没有注册的Activity
date: 2018-10-25 23:56:55
tags:
	- 技术相关
  - Android
---

启动没有在AndroidManifest中注册的Activity是安卓插件化中一个很重要的知识点,只有这样你才能把Activity中分离出来,放到插件中．

启动没有在AndroidManifest中注册的Activity，会涉及到Activity启动流程、反射、动态代理的知识,我觉得就算不学插件化,掌握这些知识也是很有用的．


# Activity的启动流程

为了达到启动没有在AndroidManifest中注册的Activity的目的,我们先来分析下Activity的启动流程,看看有没有什么突破口.

这部分的知识我在[《从源码看Activity生命周期》](http://blog.islinjw.cn/2018/03/07/%E4%BB%8E%E6%BA%90%E7%A0%81%E7%9C%8BActivity%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F/)这篇博客里面其实也有讲过,这里只做大概的讲解,然后做一些补充,感兴趣的同学可以将两篇博客结合起来看看．

## 抛出ActivityNotFoundException的原因

如果使用startActivity去启动一个没有在AndroidManifest中注册的Activity,正常情况下是会抛出ActivityNotFoundException的,那这个异常是怎么抛出来的呢?

我们知道调用Activity.startActivity方法,实际上最后是调用了Instrumentation.execStartActivity:

```
public class Instrumentation {
  ...

  public ActivityResult execStartActivity(
                  Context who, IBinder contextThread, IBinder token, Activity target,
                  Intent intent, int requestCode, Bundle options) {
      ...
      int result = ActivityManagerNative.getDefault()
                         .startActivity(whoThread, who.getBasePackageName(), intent,
                                      intent.resolveTypeIfNeeded(who.getContentResolver()),
                                      token, target != null ? target.mEmbeddedID : null,
                                      requestCode, 0, null, null, options);
      checkStartActivityResult(result, intent);
      ...
  }

  ...

  public static void checkStartActivityResult(int res, Object intent) {
        ...
        switch (res) {
              case ActivityManager.START_INTENT_NOT_RESOLVED:
              case ActivityManager.START_CLASS_NOT_FOUND:
                  if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                      throw new ActivityNotFoundException(
                              "Unable to find explicit activity class "
                              + ((Intent)intent).getComponent().toShortString()
                              + "; have you declared this activity in your AndroidManifest.xml?");
                  throw new ActivityNotFoundException(
                          "No Activity found to handle " + intent);
              ...
        }
        ...
    }

    ...
}

```

可以看到Instrumentation又是通过ActivityManagerNative.getDefault()拿到一个IActivityManager去调用其startActivity来启动Activity的．

这个IActivityManager内部实际是通过Binder机制将处理转发给ActivityManagerService:

```
public abstract class ActivityManagerNative extends Binder implements IActivityManager
    ...

    static public IActivityManager getDefault() {
       return gDefault.get();
    }

    ...

    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            //实际上是用Binder机制与AMS进行交互
            IBinder b = ServiceManager.getService("activity");
            IActivityManager am = asInterface(b);
            return am;
        }
    };

    ...
}
```

所以可以看到通过ActivityManagerService去startActivity之后会有个返回值.

ActivityManagerService内部会使用PackageManagerService查询这个Activity是否在AndroidManifest中注册.如果没有,就会返回START\_CLASS\_NOT\_FOUND或者START\_INTENT\_NOT\_RESOLVED,这个时候Instrumentation就会抛出ActivityNotFoundException.

所以ActivityNotFoundException就是这样被抛出的．

## Activity是怎样被创建的

我们都知道两个不同的进程直接是不能直接访问内存的,所以处于应用进程的Activity肯定还是应用进程去创建,而不是被AMS创建的.

这块的代码在ActivityThread中实现:

```
public final class ActivityThread {
    ...
    final H mH = new H();

    ...
    @Override
    public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
            CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
            int procState, Bundle state, PersistableBundle persistentState,
            List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
            boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
        ...
        sendMessage(H.LAUNCH_ACTIVITY, r);
    }

    ...
    private class H extends Handler {
        public static final int LAUNCH_ACTIVITY         = 100;
        ...

        public void handleMessage(Message msg) {
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;
                ...
            }
            ...
    }
    ...
}
```

AMS会调用ActivityThread的scheduleLaunchActivity,在这个方法中会使用一个Hander同步到主线程中再去创建Activity.

## Activity启动的原理图

{% img /插件化之启动没有注册的Activity/1.png %}


# 怎样欺骗ActivityManagerService

从上面的Activity启动的原理图可以看到大概的流程是:

应用将要启动的Activity告诉AMS->AMS检查Activity是否注册->AMS让ActivityThread去创建Activity．

那是不是可以这样呢?

1. 新建一个StubActivity并且在AndroidManifest中注册
2. 将想要启动的Activity换成StubActivity,而将真正想要启动的Activity保存到Extra中
3. 骗过AMS
4. 在ActivityThread中拿出真正想要创建的Activity换回来去创建


修改后的原理如下:

{% img /插件化之启动没有注册的Activity/2.png %}

## 将要启动的Activity替换成StubActivity

第一步是将要启动的Activity替换成StubActivity,我们回顾下上一节看到的ActivityManagerNative代码:

```
public abstract class ActivityManagerNative extends Binder implements IActivityManager
    ...

    static public IActivityManager getDefault() {
       return gDefault.get();
    }

    ...

    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            //实际上是用Binder机制与AMS进行交互
            IBinder b = ServiceManager.getService("activity");
            IActivityManager am = asInterface(b);
            return am;
        }
    };

    ...
}
```

可以看到这个gDefault其实是个静态的私有成员变量.

那我们是不是可以通过反射,将它替换成我们写的Singleton<IActivityManager>,然后保存好原来的gDefault,在替换的代码里面先将要启动的Activity替换成StubActivity,然后再将Intent传给原来的gDefault?


大概的做法如下:

```

class MyActivityManager implements IActivityManager {
    private IActivityManager mOrigin;

    public MyActivityManager(IActivityManager origin) {
        mOrigin = origin;
    }
    public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode, int flags,
            ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        // TODO 将要启动的activity替换成StubActivity

        return mOrigin. startActivity(caller, callingPackage, intent,
            resolvedType, resultTo, resultWho, requestCode, flags,
            profilerInfo, options);
    }
    ...
}

Class c = Class.forName("android.app.ActivityManagerNative");
final Field field =  c.getDeclaredField("gDefault");
field.setAccessible(true);

Singleton<IActivityManager> proxy = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        return new MyActivityManager(field.get(null));
    }
};

field.set(null, proxy);

```

但是这个做法问题很大,首先我们要将IActivityManager的所有方法都实现一遍转发给mOrigin。而且最大的问题是IActivityManager和Singleton被隐藏了,我们在应用层是找不到定义的!

那怎么办呢？别急,我们先来看看Singleton的实现:

```
public abstract class Singleton<T> {
    private T mInstance;

    protected abstract T create();

    public final T get() {
        synchronized (this) {
            if (mInstance == null) {
                mInstance = create();
            }
            return mInstance;
        }
    }
}
```

其实最终的IActivityManager是保存在mInstance这个变量里面的,我们只需要替换这个变量就好,于是就绕过了Singleton没有定义的问题。但是还有这个IActivityManager的定义问题摆在我们面前。

怎么办呢？答案就是我们可以用动态代理的方法去创建IActivityManager。关于动态代理我之前写过一篇博客 [《Java自定义注解和动态代理》](http://blog.islinjw.cn/2016/05/27/Java%E8%87%AA%E5%AE%9A%E4%B9%89%E6%B3%A8%E8%A7%A3%E5%92%8C%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86/) ,大家感兴趣的话可以去看看。这里就直接把代码贴上了:

```
// 获取gDefault
Class activityManagerClass = Class.forName("android.app.ActivityManagerNative");
Field gDefaultField = activityManagerClass.getDeclaredField("gDefault");
gDefaultField.setAccessible(true);
Object gDefault = gDefaultField.get(null);

//　获取mIntance
Class singletonClass = Class.forName("android.util.Singleton");
Field mInstanceField = singletonClass.getDeclaredField("mInstance");
mInstanceField.setAccessible(true);
Object mInstance = mInstanceField.get(gDefault);

// 替换mIntance
Object proxy = Proxy.newProxyInstance(
        mInstance.getClass().getClassLoader(),
        new Class[]{Class.forName("android.app.IActivityManager")},
        new IActivityManagerHandler(mInstance));
mInstanceField.set(gDefault, proxy);


public static class IActivityManagerHandler implements InvocationHandler {
    private Object mOrigin;

    IActivityManagerHandler(Object origin) {
        mOrigin = origin;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if ("startActivity".equals(method.getName())) {
            int index = 0;
            for (int i = 0; i < args.length; i++) {
                if (args[i] instanceof Intent) {
                    index = i;
                    break;
                }
            }
            Intent raw = (Intent) args[index];

            Intent intent = new Intent();
            intent.setClassName(raw.getComponent().getPackageName(), StubActivity.class.getName());
            intent.putExtra("RawIntent", raw);
            args[index] = intent;
        }
        return method.invoke(mOrigin, args);
    }
}
```

上面的代码的功能就是创建一个IActivityManager的代理,代理startActivity方法,将启动的Activity的Intent换成启动StubActivity的Intent,并且将原来的Intent保存起来放到RawIntent这个Extra里。

然后用它去替换ActivityManagerNative.gDefault的mInstance成员变量。


## 将StubActivity替换会要启动的Activity


在上面我们已经将要启动的Activity替换成了已经注册了的StubActivity,这样在AMS检查的时候就能在AndroidManifest查到,不会报ActivityNotFoundException了.

然后AMS会让ActivityThread去创建Activity,这个时候就要将StubActivity替换会真正要启动的Activity了.

再回顾下这部分的代码:


```
public final class ActivityThread {
    ...
    final H mH = new H();

    ...
    @Override
    public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
            CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
            int procState, Bundle state, PersistableBundle persistentState,
            List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
            boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
        ...
        sendMessage(H.LAUNCH_ACTIVITY, r);
    }

    ...
    private class H extends Handler {
        public static final int LAUNCH_ACTIVITY         = 100;
        ...

        public void handleMessage(Message msg) {
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;
                ...
            }
            ...
    }
    ...
}
```

ActivityThread的scheduleLaunchActivity方法会被调到,然后会向mH发送LAUNCH_ACTIVITY消息.

所以关键点就是将这个mH变量替换成我们的代理对象,将Intent替换回之前保存的RawIntent.

但是这里有个问题,H是个内部类,我们是没有办法用动态代理的方式创建内部类的,也就是说我们没有办法替换掉mH这个对象.

于是只好继续挖一挖Handler内部有没有机会了,其实在Handler.dispatchMessage里面是会先判断mCallback是不是有赋值的,如果有就会将消息交给它去处理.

```
public class Handler {
    ...
    final Callback mCallback;
    ...
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
    ...
}
```

所以我们可以从这个mCallback入手,将mH的mCallback设置成我们的代理对象:

```
//　获取ActivityThread实例
Class activityThreadClass = Class.forName("android.app.ActivityThread");
Field threadField = activityThreadClass.getDeclaredField("sCurrentActivityThread");
threadField.setAccessible(true);
Object sCurrentActivityThread = threadField.get(null);

//　获取mH变量
Field mHField = activityThreadClass.getDeclaredField("mH");
mHField.setAccessible(true);
Object mH = mHField.get(sCurrentActivityThread);

//　设置mCallback变量
Field mCallbackField = Handler.class.getDeclaredField("mCallback");
mCallbackField.setAccessible(true);
Handler.Callback callback = new Handler.Callback() {
   @Override
   public boolean handleMessage(Message msg) {
       if (msg.what == 100) {
           try {
               Field intentField = msg.obj.getClass().getDeclaredField("intent");
               intentField.setAccessible(true);
               Intent intent = (Intent) intentField.get(msg.obj);
               Intent raw = intent.getParcelableExtra("RawIntent");
               intent.setComponent(raw.getComponent());
           } catch (Exception e) {
               Log.e("hook", "get intent err", e);
           }

       }
       return false;
   }
};
mCallbackField.set(mH, callback);
```

ActivityThread的实例保存在sCurrentActivityThread这个静态成员变量里,代码我就不贴了,然后我们在mCallback这里将要启动的Activity设置回来.


# 处理Android 8.0的情况

上面的代码运行在8.0的系统上会崩溃,原因是8.0对Activity的启动这块做了些改动,不再使用ActivityManagerNative.getDefault()了,改成了ActivityManager.getService():


```
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
    ...
    int result = ActivityManager.getService()
        .startActivity(whoThread, who.getBasePackageName(), intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()),
                token, target != null ? target.mEmbeddedID : null,
                requestCode, 0, null, options);
    checkStartActivityResult(result, intent);
    ...
}
```


ActivityManager其实和ActivityManagerNative很像:

```
public class ActivityManager {
    ...
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }
    ...
    private static final Singleton<IActivityManager> IActivityManagerSingleton =
          new Singleton<IActivityManager>() {
              @Override
              protected IActivityManager create() {
                  final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                  final IActivityManager am = IActivityManager.Stub.asInterface(b);
                  return am;
              }
          };
    ...
}
```

所以我们类似的去替换IActivityManagerSingleton就好了:

```
// 获取IActivityManagerSingleton
Class activityManagerClass = Class.forName("android.app.ActivityManager");
Field singletonField = activityManagerClass.getDeclaredField("IActivityManagerSingleton");
singletonField.setAccessible(true);
Object gDefault = singletonField.get(null);

//　获取mIntance
Class singletonClass = Class.forName("android.util.Singleton");
Field mInstanceField = singletonClass.getDeclaredField("mInstance");
mInstanceField.setAccessible(true);
Object mInstance = mInstanceField.get(gDefault);

// 替换mIntance
Object proxy = Proxy.newProxyInstance(
        mInstance.getClass().getClassLoader(),
        new Class[]{Class.forName("android.app.IActivityManager")},
        new IActivityManagerHandler(mInstance));
mInstanceField.set(gDefault, proxy);
```

# 处理AppCompatActivity的情况


到目前为止,我们已经可以正常启动没有注册的Activity了,但是其实还有一个BUG:如果启动的是没有注册的AppCompatActivity就会崩溃。

```
10-25 19:32:30.867  8754  8754 E AndroidRuntime: Caused by: java.lang.IllegalArgumentException: android.content.pm.PackageManager$NameNotFoundException: ComponentInfo{me.linjw.plugindemo/me.linjw.plugindemo.HideActivity}
10-25 19:32:30.867  8754  8754 E AndroidRuntime:        at android.support.v4.app.NavUtils.getParentActivityName(NavUtils.java:285)
10-25 19:32:30.867  8754  8754 E AndroidRuntime:        at android.support.v7.app.AppCompatDelegateImplV9.onCreate(AppCompatDelegateImplV9.java:158)
10-25 19:32:30.867  8754  8754 E AndroidRuntime:        at android.support.v7.app.AppCompatDelegateImplV14.onCreate(AppCompatDelegateImplV14.java:58)
10-25 19:32:30.867  8754  8754 E AndroidRuntime:        at android.support.v7.app.AppCompatActivity.onCreate(AppCompatActivity.java:72)
10-25 19:32:30.867  8754  8754 E AndroidRuntime:        at com.cvte.tv.speech.TestActivity.onCreate(TestActivity.java:14)
10-25 19:32:30.867  8754  8754 E AndroidRuntime:        at android.app.Activity.performCreate(Activity.java:6664)
10-25 19:32:30.867  8754  8754 E AndroidRuntime:        at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1118)
10-25 19:32:30.867  8754  8754 E AndroidRuntime:        at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2599)
```

网上很多讲启动未注册的Activity的文章要不就没有讲这个,要不就没有详细讲如何处理,直接一笔带过了.这里我手把手带大家解BUG.

遇到问题先不要慌,先看看打印找到崩溃的代码在哪:

```
@Nullable
public static String getParentActivityName(Activity sourceActivity) {
    try {
        return getParentActivityName(sourceActivity, sourceActivity.getComponentName());
    } catch (NameNotFoundException e) {
        // Component name of supplied activity does not exist...?
        throw new IllegalArgumentException(e);
    }
}

@Nullable
public static String getParentActivityName(Context context, ComponentName componentName)
        throws NameNotFoundException {
    PackageManager pm = context.getPackageManager();
    ActivityInfo info = pm.getActivityInfo(componentName, PackageManager.GET_META_DATA);
    String parentActivity = IMPL.getParentActivityName(context, info);
    return parentActivity;
}
```

很明显是PackageManager.getActivityInfo在AndroidManifest里面找不到Activity抛出了NameNotFoundException.

所以我们看看有没有办法替换一下这个Context.getPackageManager()拿到的PackageManager:


```
class ContextImpl extends Context {
	...
	@Override
	public PackageManager getPackageManager() {
	    if (mPackageManager != null) {
	        return mPackageManager;
	    }

	    IPackageManager pm = ActivityThread.getPackageManager();
	    if (pm != null) {
	        // Doesn't matter if we make more than one instance.
	        return (mPackageManager = new ApplicationPackageManager(this, pm));
	    }

	    return null;
	}
	...
}
```

ContextImpl会从ActivityThread.getPackageManager获取IPackageManager,让我们继续挖:


```
public final class ActivityThread {
	...
	static volatile IPackageManager sPackageManager;
	...
	public static IPackageManager getPackageManager() {
	    if (sPackageManager != null) {
	        //Slog.v("PackageManager", "returning cur default = " + sPackageManager);
	        return sPackageManager;
	    }
	    IBinder b = ServiceManager.getService("package");
	    //Slog.v("PackageManager", "default service binder = " + b);
	    sPackageManager = IPackageManager.Stub.asInterface(b);
	    //Slog.v("PackageManager", "default service = " + sPackageManager);
	    return sPackageManager;
	}
	...
}
```

所以sPackageManager就是我们的突破点,让我们来把它换掉:

```
try {
    //要先获取一下,保证它初始化
    context.getPackageManager();

    Class activityThread = Class.forName("android.app.ActivityThread");
    Field pmField = activityThread.getDeclaredField("sPackageManager");
    pmField.setAccessible(true);
    final Object origin = pmField.get(null);
    Object handler = Proxy.newProxyInstance(activityThread.getClassLoader(),
            new Class[]{Class.forName("android.content.pm.IPackageManager")},
            new PackageManagerHandler(context, origin));
    pmField.set(null, handler);
} catch (Exception e) {
    Log.e("hook", "hook IPackageManager err", e);
}

static class PackageManagerHandler implements InvocationHandler {
        private Context mContext;
        private Object mOrigin;

        PackageManagerHandler(Context context, Object origin) {
            mContext = context;
            mOrigin = origin;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (!method.getName().equals("getActivityInfo")) {
                return method.invoke(mOrigin, args);
            }

            //如果没有注册,并不会抛出异常,而是会直接返回null
            Object ret = method.invoke(mOrigin, args);
            if (ret == null) {
                for (int i = 0; i < args.length; i++) {
                    if (args[i] instanceof ComponentName) {
                        ComponentName componentName = (ComponentName) args[i];
                        componentName.getClassName();
                        args[i] = new ComponentName(
                        	mContext.getPackageName(),
                        	StubActivity.class.getName()
                        );
                        return method.invoke(mOrigin, args);
                    }
                }
            }
            return ret;

        }
    }
```

在IPackageManager.getActivityInfo方法抛出异常的时候invoke会返回null,就代表这个Activity没有注册,我们直接将他换成StubActivity就好。

大功告成!

# 完整Demo

完整Demo见我的[Github](https://github.com/bluesky466/PluginDemo)
