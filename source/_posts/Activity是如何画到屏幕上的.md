title: Activity是如何画到屏幕上的
date: 2018-02-28 23:38:14
tags:
	- 技术相关
  - Android
---

# Activity是如何管理布局的

一切从setContentView说起。安卓中最常用的代码可能就是setContentView了，但大家有没有想过这个方法的背后到底做了些什么？

```
public class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```

直接跳转到Activity的源码我们可以看到,Activity.setContentView实际上调用了PhoneWindow.setContentView:

```
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window) {
     ...
     mWindow = new PhoneWindow(this, window);
     ...
}

public Window getWindow() {
    return mWindow;
}

public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

我们继续跟踪PhoneWindow的源码,可以发现最终layoutResID被inflate出来之后是成为了mDecor这个DecorView的子view。而DecorView实际上是一个FrameLayout:

```
public void setContentView(int layoutResID) {
     if (mContentParent == null) {
          installDecor();
     } else {
          mContentParent.removeAllViews();
     }
     mLayoutInflater.inflate(layoutResID, mContentParent);
     final Callback cb = getCallback();
     if (cb != null && !isDestroyed()) {
          cb.onContentChanged();
     }
}

private void installDecor() {
     if (mDecor == null) {
          mDecor = generateDecor();
          ...
     }
     if (mContentParent == null) {
          //mContentParent 实际上是mDecor的一个子view
          mContentParent = generateLayout(mDecor);
          ...
     }
     ...
}

protected DecorView generateDecor() {
     return new DecorView(getContext(), -1);
}

private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {
     ...
}
```


这里的generateLayout比较重要，它实际上是根据window的各种属性inflate出不同的layout挂到DecorView下面,而mContentParent是这个layout中的一个子ViewGroup。如果我们没有对window的属性进行设置就会使用默认的com.android.internal.R.layout.screen_simple这个layout:

```
protected ViewGroup generateLayout(DecorView decor) {    
     ...
     if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
          ...
          layoutResource = com.android.internal.R.layout.screen_title_icons;
          ...
     } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
  && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
          layoutResource = com.android.internal.R.layout.screen_progress;
     } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
          ...
          layoutResource = com.android.internal.R.layout.screen_custom_title;
          ...
     } ... else{
          layoutResource = com.android.internal.R.layout.screen_simple;
     }
     ...
     View in = mLayoutInflater.inflate(layoutResource, null);
     decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
     ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
     ...
     return contentParent;
}
```

我们可以在AndroidSdk根目录/platforms/android-19/data/res/layout/下面找到这些layout xml,例如screen_simple,这是个竖直的LinearLayout,由上方的ActionBar和下方的content FrameLayout组成。它就是我们最常见的带ActionBar的activity样式:

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
  android:layout_width="match_parent"
  android:layout_height="match_parent"
  android:fitsSystemWindows="true"
  android:orientation="vertical">
  <ViewStub android:id="@+id/action_mode_bar_stub"
    android:inflatedId="@+id/action_mode_bar"
    android:layout="@layout/action_mode_bar"
    android:layout_width="match_parent"
    android:layout_height="wrap_content" />
  <FrameLayout
    android:id="@android:id/content"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:foregroundInsidePadding="false"
    android:foregroundGravity="fill_horizontal|top"
    android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

我们可以用一张图片来总结下Activity是如何管理布局的(这里假设DecorView里面添加了screen_simple这个布局):

{% img /Activity是如何画到屏幕上的/1.png %}

# Activity的布局是怎样被系统渲染的

在上一节中我们已经知道了Activity是怎样管理布局的。接着我们来看看Activity中的布局是如何渲染到系统的。

ActivityThread用于管理Activity的声明周期,之后我会专门写一篇文章来讲它。我们直接看ActivityThread.handleResumeActivity方法:

```
final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume) {
  ...
  //performResumeActivity方法会调用Activity.onResume
  ActivityClientRecord r = performResumeActivity(token, clearHide);
  ...
  r.window = r.activity.getWindow();
  View decor = r.window.getDecorView();
  decor.setVisibility(View.INVISIBLE);
  ViewManager wm = a.getWindowManager();
  WindowManager.LayoutParams l = r.window.getAttributes();
  a.mDecor = decor;
  l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
  l.softInputMode |= forwardBit;
  if (a.mVisibleFromClient) {
    a.mWindowAdded = true;
    wm.addView(decor, l);
  }
  ...
}
```

可以看到它在Activity.onResume之后从Activity中获取了Window,然后又从window中获取了DecorView。最后使用WindowManager.addView将DecorView添加到了WindowManager中。这样就将DecorView在手机上渲染了出来。

WindowManager.addView方法可以将一个view渲染到手机界面上。不知道大家有没有做过类似悬浮球的应用,就是用WindowManager.addView去实现的。这里就不再展开了，大家有兴趣的话可以自己去搜索一下。


# 为什么不能在子线程中操作view

我们都知道,在安卓中必须在ui线程中操作ui,不能在子线程中对view进行操作,否则或抛出CalledFromWrongThreadException异常。但是在子线程中操作view是不是真的就一定会出现异常呢?让我们运行下面的代码:

```
public class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        new Thread(new Runnable() {
            @Override
            public void run() {
                ((TextView)findViewById(R.id.textView)).setText("子线程中操作view");
            }
        }).start();
    }
}
```

我们可以看到实际上在onCreate的时候直接启动子线程去修改TextView的文字是可以正常运行的,且文字也是显示正常的:


{% img /Activity是如何画到屏幕上的/2.png %}


让我们加1秒的延迟再试一下:

```
public class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                ((TextView)findViewById(R.id.textView)).setText("子线程中操作view");
            }
        }).start();
    }
}
```

运行之后就能看到熟悉的崩溃日志了:

```
02-28 22:36:48.550  3780  3817 E AndroidRuntime: FATAL EXCEPTION: Thread-5
02-28 22:36:48.550  3780  3817 E AndroidRuntime: Process: com.example.linjw.myapplication, PID: 3780
02-28 22:36:48.550  3780  3817 E AndroidRuntime: android.view.ViewRootImpl$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.
02-28 22:36:48.550  3780  3817 E AndroidRuntime:        at android.view.ViewRootImpl.checkThread(ViewRootImpl.java:6987)
02-28 22:36:48.550  3780  3817 E AndroidRuntime:        at android.view.ViewRootImpl.requestLayout(ViewRootImpl.java:1104)
02-28 22:36:48.550  3780  3817 E AndroidRuntime:        at android.view.View.requestLayout(View.java:19807)
02-28 22:36:48.550  3780  3817 E AndroidRuntime:        at android.view.View.requestLayout(View.java:19807)
02-28 22:36:48.550  3780  3817 E AndroidRuntime:        at android.view.View.requestLayout(View.java:19807)
02-28 22:36:48.550  3780  3817 E AndroidRuntime:        at android.view.View.requestLayout(View.java:19807)
02-28 22:36:48.550  3780  3817 E AndroidRuntime:        at android.support.constraint.ConstraintLayout.requestLayout(ConstraintLayout.java:874)
02-28 22:36:48.550  3780  3817 E AndroidRuntime:        at android.view.View.requestLayout(View.java:19807)
02-28 22:36:48.550  3780  3817 E AndroidRuntime:        at android.widget.TextView.checkForRelayout(TextView.java:7375)
02-28 22:36:48.550  3780  3817 E AndroidRuntime:        at android.widget.TextView.setText(TextView.java:4487)
02-28 22:36:48.550  3780  3817 E AndroidRuntime:        at android.widget.TextView.setText(TextView.java:4344)
02-28 22:36:48.550  3780  3817 E AndroidRuntime:        at android.widget.TextView.setText(TextView.java:4319)
02-28 22:36:48.550  3780  3817 E AndroidRuntime:        at com.example.linjw.myapplication.MainActivity$1.run(MainActivity.java:20)
02-28 22:36:48.550  3780  3817 E AndroidRuntime:        at java.lang.Thread.run(Thread.java:760)
```

为什么延迟1秒之后就能看到异常被抛出了呢?本着寻根问底的精神,我们直接扣ViewRootImpl的源码看看CalledFromWrongThreadException异常是怎么被抛出的:

```
public ViewRootImpl(Context context, Display display) {
    ...
    mThread = Thread.currentThread();
    ...
}

void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
                "Only the original thread that created a view hierarchy can touch its views.");
    }
}

public void requestLayout() {
   if (!mHandlingLayoutInLayoutRequest) {
       checkThread();
       mLayoutRequested = true;
       scheduleTraversals();
   }
}
```

在View.requestLayout方法中会调用ViewRootImpl.requestLayout,然后在ViewRootImpl.requestLayout里面会调用ViewRootImpl.checkThread去判断当前线程和创建ViewRootImpl的线程是不是同一个线程。如果不是的话就抛出CalledFromWrongThreadException异常。

那ViewRootImpl又是在哪个线程中被创建的呢?还记得上一节中讲到的ActivityThread.handleResumeActivity方法中将DecorView添加到WindowManager中吗?WindowManager实际上是WindowManagerImpl实例:

```
public final class WindowManagerImpl implements WindowManager {
  private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
  ...
  public void addView(View view, ViewGroup.LayoutParams params) {
      mGlobal.addView(view, params, mDisplay, mParentWindow);
  }
  ...
}
```

我们可以看到WindowManagerImpl.addView实际上是调到了WindowManagerGlobal.addView:

```
public final class WindowManagerGlobal {
  public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
    ...
    ViewRootImpl root;
    ...
    root = new ViewRootImpl(view.getContext(), display);
    ...
  }    
}          
```

所以ViewRootImpl是在handleResumeActivity的线程中被创建的,我们都知道onResume是在主线程中被调用的,所以ViewRootImpl是在主线程中被调用的。所以只要在非主线程中调用ViewRootImpl.requestLayout就会抛出CalledFromWrongThreadException异常。


那回到最初的问题,为什么我们在onCreate的时候直接起子线程去修改TextView的文字,不会抛出CalledFromWrongThreadException异常?因为ViewRootImpl是在onResume中创建的,在onCreate的时候它就还没有被创建,所以就不会抛出CalledFromWrongThreadException异常。

等到onResume的时候ViewRootImpl被创建,会进行第一次layout,这个时候才会检查是否在主线程中操作ui。
