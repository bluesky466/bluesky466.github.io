title: 从源码看Activity生命周期
date: 2018-03-07 23:47:39
tags:
	- 技术相关
  - Android
---

# Activity是如何创建的

我们都知道在手机上点击应用的图标，系统会通过ActivityManagerService去启动它的主Activity，接下来我们就来一步步看看在它究竟是如何启动应用的。

首先手机开机的时候我们看到的界面其实是Launcher应用的主Activity,Launcher其实就是一个会被系统默认启动的安卓应用。在上面点击已装应用的图标，就会调用Actvity.startActivity去启动其他的应用。而Activity实际上是继承ContextWrapper的,所以调的是ContextWrapper.startActivity方法:

```
public class ContextWrapper extends Context {
     ...
     Context mBase;
     ...
     public ContextWrapper(Context base) {
         mBase = base;
     }
     ...
     @Override
     public void startActivity(Intent intent) {
         mBase.startActivity(intent);
     }
     ...
}
```

可以看到这里使用了委托的方式，实际上是调了mBase.startActivity。那这个mBase到底是什么呢？让我们来看看Context的继承关系图:

{% img /从源码看Activity生命周期/1.png %}


Context只有两个直接的子类,一个ContextImpl，一个ContextWrapper。ContextWrapper类如其名仅仅是一个包装的功能，它的成员变量mBase其实就是ContextImpl，所有实际的工作都是由ContextImpl去实现的。


于是我们就去看看ContextImpl.startActivity:


```
class ContextImpl extends Context {
    ...
    @Override
    public void startActivity(Intent intent) {
        ...
        startActivity(intent, null);
    }

    @Override
    public void startActivity(Intent intent, Bundle options) {
      ...
      mMainThread.getInstrumentation().execStartActivity(
                            getOuterContext(), mMainThread.getApplicationThread(), null,
                            (Activity)null, intent, -1, options);
    }
    ...
}

public final class ActivityThread {
    ...
    Instrumentation mInstrumentation;
    ...
    public Instrumentation getInstrumentation()  {
        return mInstrumentation;
    }
    ...
}
```

好吧，这里又将锅甩给了Instrumentation。Instrumentation其实也是一个比较偏但是很有作用的东西(通过它我们能做到很多神奇的事情，例如[Hook 应用入口 Activity](https://www.jianshu.com/p/53c9e8641e57))，主要用于监控Activity，这里我就不详细讲了，感兴趣的同学可以直接去搜索一下。我们直接看启动activity相关的代码:


```
public ActivityResult execStartActivity(
                Context who, IBinder contextThread, IBinder token, Activity target,
                Intent intent, int requestCode, Bundle options) {
   ...
   int result = ActivityManagerNative.getDefault()
                             .startActivity(whoThread, who.getBasePackageName(), intent,
                                          intent.resolveTypeIfNeeded(who.getContentResolver()),
                                          token, target != null ? target.mEmbeddedID : null,
                                          requestCode, 0, null, null, options);
   ...
}
```

这里的ActivityManagerNative最后拿到的就是ActivtyManagerService的IBinder,所以最后会使用Binder机制调用系统端的ActivityManagerService去创建Activity。

但是我们知道ActivityManagerService和我们的应用是在两个进程的，如果在ActivityManagerService中创建了Activity的话我们的应用也是获取不了的。

其实ActivityManagerService主要功能不是创建Activity，而是管理Activity栈。它在创建新的Activity的时候还是会通过Binder机制调回应用进程的ActivityThread去处理。最后ActivityManagerService只保存Activity的token。由于中间代码过于曲折，我这里就不贴出来了.这里直接看ActivityThread的代码吧：


```
public final class ActivityThread {
    final ArrayMap<IBinder, ActivityClientRecord> mActivities
                    = new ArrayMap<IBinder, ActivityClientRecord>();
    ...
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        Activity a = performLaunchActivity(r, customIntent);
        ...
    }
    ...
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        activity = mInstrumentation.newActivity(
                            cl, component.getClassName(), r.intent);
        ...
        r.activity = activity;
        ...
        mActivities.put(r.token, r);
    }
}
```


可以看到这里也是用Instrumentation去创建Activity的，创建完之后就将它丢到一个Map里面。而Instrumentation.newActivity则很简单，通过反射去创建Activity:

```
public class Instrumentation {
    ...
    public Activity newActivity(ClassLoader cl, String className, Intent intent)
                                throws InstantiationException, IllegalAccessException,
                                ClassNotFoundException {
        return (Activity)cl.loadClass(className).newInstance();
    }
    ...
}
```

总结下来，上面讲的的方法的调用时序图如下:

{% img /从源码看Activity生命周期/2.png %}


# Activity的生命周期是如何被调用的

这里涉及到几个类：ActivityManagerService、ActivityStackSupervisor、ActivityStack、ActivityThread。

ActivityManagerService负责通过binder机制接收启动应用的请求，它内部有各个ActivityStackSupervisor成员变量，用于管理Activity栈：


```
public final class ActivityManagerService extends ActivityManagerNative
implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
    ...
    /** Run all ActivityStacks through this */
    ActivityStackSupervisor mStackSupervisor;
    ...
}
```

ActivityStackSupervisor管理用于Activity栈列表，它负责将Activity压入对应的Activity栈中:

```
public final class ActivityStackSupervisor {
    ...
    /** All the non-launcher stacks */
    private ArrayList<ActivityStack> mStacks = new ArrayList<ActivityStack>();
    ...
}
```

ActivityStack用于管理Activity的生命周期，例如在新Activity被压入的时候调用旧栈顶Activity的onPasuse和onStop还有新activity的onStart和onResume。

```
final class ActivityStack {
    ...
    final void startPausingLocked(boolean userLeaving, boolean uiSleeping) {
        ...
        prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing, userLeaving, prev.configChangeFlags);
        ...
    }
    ...
}
```

ActivityStack并不会直接调用Activity的生命周期方法，而是通过ActivityThread间接调用。由于ActivityStack在系统进程中,而ActivityThread在应用进程中，所以通过Binder机制调用之后去到ActivityThread那里不是主线程，于是ActivityThread内部就使用了Handler机制同步到主线程中调用:


```
public final class ActivityThread {
    ...
    public final void schedulePauseActivity(IBinder token, boolean finished,  boolean userLeaving, int configChanges) {
        queueOrSendMessage(
            finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
            token,
            (userLeaving ? 1 : 0),
            configChanges);
    }
    ...
    private void queueOrSendMessage(int what, Object obj, int arg1, int arg2) {
        synchronized (this) {
            Message msg = Message.obtain();
            msg.what = what;
            msg.obj = obj;
            msg.arg1 = arg1;
            msg.arg2 = arg2;
            mH.sendMessage(msg);
        }
    }
    ...
    private class H extends Handler {
        ...
        public void handleMessage(Message msg) {
            ...
            switch (msg.what) {
                ...
                case PAUSE_ACTIVITY:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                    handlePauseActivity((IBinder)msg.obj, false, msg.arg1 != 0, msg.arg2);
                    maybeSnapshot();
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                ...
                case RESUME_ACTIVITY:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
                    handleResumeActivity((IBinder)msg.obj, true,
                    msg.arg1 != 0, true);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                ...
            }
            ...
        }
        ...
    }
    ...
    private void handlePauseActivity(IBinder token, boolean finished,boolean userLeaving, int configChanges) {
        ActivityClientRecord r = mActivities.get(token);
        ...
    }
}
```

这里需要提的一点是AcvitiyThread里面可能有不止一个activity。所以需要传一个token去指定调用哪个activity。handlePauseActivity方法最终会调用mInstrumentation.callActivityOnPause再调用到Activity.onPause。这里就不继续展示代码了。

总结一下，上面讲到的这些类之间的关系是这样的:

{% img /从源码看Activity生命周期/3.png %}

ActivityThread通过handler机制将activity的生命周期同步到主线程中调用:

{% img /从源码看Activity生命周期/4.png %}
