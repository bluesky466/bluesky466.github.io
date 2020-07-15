title: 深挖view绘制流程(一)
date: 2020-07-16 00:12:19
tags:
  - 技术相关
  - Android
---

view的绘制流程是安卓应用开发的基础,但是可能很多人对它的理解可能仅限于onMeasure、onLayout、onDraw这三个方法。

我们本着刨根问底的思想,从应用层到native层,对view的绘制流程是如何调用的一探到底。

大家都知道可以用View.requestLayout触发view的重新布局和绘制,我们就从这个方法开始挖。这个方法会一直往上调用父布局的requestLayout:

```java
public void requestLayout() {
    ...
    // 添加重新布局和绘制的标志位
    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    mPrivateFlags |= PFLAG_INVALIDATED;

    if (mParent != null && !mParent.isLayoutRequested()) {
        mParent.requestLayout();
    }
    ...
}
```

那到什么时候才是个头呢？答案是会一直调用到ViewRootImpl.requestLayout。

# View树的结构

大家可能没有太注意过，View.getParent的返回值其实不是View也不是ViewGroup而是ViewParent。这就是为了将ViewRootImpl挂到整棵view树的根:

```java
public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {
  ...
	protected ViewParent mParent;
	...
	public final ViewParent getParent() {
	    return mParent;
	}
	...
}

public abstract class ViewGroup extends View implements ViewParent, ViewManager {
	...
}

public final class ViewRootImpl implements ViewParent, View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
	...
}
```

这里就遇到了第一个问题,ViewRootImpl是如何成为View树的根节点的?让我们先跳出绘制流程看看这个问题。

一般设置activity布局都是调用Activity.setContentView方法,所以我们从这里开始看:

```java
public void setContentView(View view) {
    getWindow().setContentView(view);
    initWindowDecorActionBar();
}
```

这个getWindow拿到的实际是PhoneWindow,所以我们从它的setContentView继续往下追:

```java
@Override
public void setContentView(View view) {
    setContentView(view, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
}

@Override
public void setContentView(View view, ViewGroup.LayoutParams params) {
    if (mContentParent == null) {
        installDecor();
    }
    ...
    mContentParent.addView(view, params);
    ...
}

private void installDecor() {
    ...
    if (mDecor == null) {
        mDecor = generateDecor(-1);
        ...
    }
    ...
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor);
        ...
    }
}

protected DecorView generateDecor(int featureId) {
    ...
    return new DecorView(context, featureId, this, getAttributes());
}

protected ViewGroup generateLayout(DecorView decor) {
	...
	ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
	...
	return contentParent;
}

// Window.java
@Nullable
public <T extends View> T findViewById(@IdRes int id) {
    return getDecorView().findViewById(id);
}
```

经过上面的代码我们知道PhoneWindow里面有个DecorView,然后会用DecorView.findViewById(ID\_ANDROID\_CONTENT)得到一个mContentParent，而我们setContentView实际上就是将它addView成为这个mContentParent的子view。整个View树如下图:

{% img /view绘制流程一探到底一/1.png %}

然后在ActivityThread.handleResumeActivity里面会将这个DecorView add 到ViewManager

```java
final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume) {
  ...
  //performResumeActivity方法会调用Activity.onResume
  ActivityClientRecord r = performResumeActivity(token, clearHide);
  ...
  r.window = r.activity.getWindow();
  View decor = r.window.getDecorView();
  ...
  ViewManager wm = a.getWindowManager();
  ...
  wm.addView(decor, l);
  ...
}
```

在ViewManager里面就会创建ViewRootImpl并且将它设置成DecorView的Parent:

```java
public final class WindowManagerImpl implements WindowManager {
  private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
  ...
  public void addView(View view, ViewGroup.LayoutParams params) {
      mGlobal.addView(view, params, mDisplay, mParentWindow);
  }
  ...
}

public final class WindowManagerGlobal {
  public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
    ...
    ViewRootImpl root;
    ...
    root = new ViewRootImpl(view.getContext(), display);
    ...
    root.setView(view, wparams, panelParentView);
    ...
  }    
}

public final class ViewRootImpl implements ViewParent, View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
	public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    ...
    mView = view;
		...
		view.assignParent(this);
		...
	}
}

public class View implements Drawable.Callback, KeyEvent.Callback, AccessibilityEventSource {
	void assignParent(ViewParent parent) {
	    ...
	    mParent = parent;
	    ...
	}
}
```

现在View树长这样（当然DecorView下面可能不只有ID\_ANDROID\_CONTENT一个子view,还会有些view用来装ActionBar之类的,这里将它们省略了）:

{% img /view绘制流程一探到底一/2.png %}

所以到这里我们能确定View.requestLayout会一直调用mParent的requestLayout方法,最终调用到ViewRootImpl.requestLayout。

# ViewRootImpl布局流程

ViewRootImpl.requestLayout里面将mTraversalRunnable丢到了mChoreographer里面,Choreographer是编舞者、舞蹈编导的意思,它的作用是在接收到屏幕垂直同步信号(VSync)的时候使用handler机制将这个Runnable同步到主线程执行。这块我们之后再详细展开。

```java
@Override
public void requestLayout() {
    ...
    scheduleTraversals();
    ..
}

void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        ...
    }
}
```

而这个mTraversalRunnable在run方法里面会调用ViewRootImpl.doTraversal,最后调到performMeasure、performLayout、performDraw，这三个方法就会调用到DecorView的measure、layout、draw方法，然后最终调用到我们熟悉的onMeasure、onLayout、onDraw。

```java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}

final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        ...
        performTraversals();
        ...
    }
}

private void performTraversals() {
	  ...
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    ...
    performLayout(lp, mWidth, mHeight);
    ...
    performDraw();
    ...
}
```

我们还能看到这里用一个mTraversalScheduled变量保存绘制请求的状态,它在scheduleTraversals里面被设置成true，下一次再进入scheduleTraversals方法判断到为true的话就会跳过。它在下次实际的绘制调用doTraversal里面才会被还原成false。

这就保证了我们连续调用多次requestLayout，只会触发一次重新布局绘制。

# VSync

上面的源码中我们看到ViewRootImpl使用Choreographer.postCallback将绘制流程调用的Runnable丢给Choreographer。

而Choreographer会在接收到VSync信号的时候去调用这个Runnable执行实际的布局绘制,那VSync是什么东东呢？我们现在就来讲一讲。

我们也许都听说过,安卓的屏幕刷新率是60Hz，即屏幕每秒刷新60次。

但实际上每次屏幕刷新并不是整个屏幕的像素同时刷新的，它的刷新过程其实是从左到右一行行像素刷新的:

{% img /view绘制流程一探到底一/3.png %}

当整个屏幕刷新完毕一个刷新周期完成就完成了，此时屏幕就会发出VSync信号通知系统。然后之后会有一个短暂的空白期等待下一次刷新。

所以我们的Choreographer就是等待VSync信号利用这个短暂的空白去计算布局和渲染绘制。

# 双缓冲

双缓冲也是一个比较成熟的方案了，想象下如果我们只用一个buffer用于保存屏幕数据，如果计算量比较大，在下次屏幕刷新之前不能完成，那就可能出现屏幕边读取像素，GPU边写入数据的情况，导致上半部分屏幕显示的是前一帧画面下半部分是后一帧画面的尴尬场景。

而双缓冲指的就是使用两个buffer，一个用于GPU写入数据，另一个用于屏幕读取数据，当GPU写入完成之后交换两个buffer，屏幕就能读取到最新的画面了。

如果当屏幕刷新周期开始，而GPU还没有或者正在写入数据的话，屏幕读取的也是上一帧的画面不会有冲突:

{% img /view绘制流程一探到底一/4.png %}

这个Jank指的就是同一个画面在屏幕上多次出现。

当然现在还出现了三缓存机制来减少Jank的出现，有兴趣的同学可以自行搜索，这里就不展开了。



# SyncBarrier机制

由于这段空白的时间特别的短暂，所以我们需要尽快的完成布局和绘制来减少Jank的发生。这里有两个方向:一个是减少计算的时间，另外一个是将计算开始的时间提前。

减少计算时间好理解，但是将计算开始的时间提前又是怎么一回事呢？由于我们的view操作都是在主线程进行的，也就是往主线程的MessageQueue里面丢入message，而MessageQueue里面的message是一般情况按顺序执行的。

我们调用Choreographer.postCallback之后并不是立刻将消息丢到MessageQueue，而是要等VSync到来之后才会丢进去，中间的时间差就可能有消息插入了，于是就会导致执行布局绘制的message可能排在后面执行:

{% img /view绘制流程一探到底一/5.png %}

所以我们如果能将布局绘制的message优先级提高，就能在下次VSync到来之前完成绘制，这里使用的就是SyncBarrier机制:

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        // 开始前先往MessageQueue post 一个SyncBarrier
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        ...
    }
}

void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        // 执行布局绘制的时候才将SyncBarrier删除
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        ...
        performTraversals();
        ...
    }
}
```

postSyncBarrier其实是往MessageQueue里面丢了一个没有target的Message:

```java
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

然后在next方法取消息的时候如果拿到了SyncBarrier，则会跳过所有不是Asynchronous的消息:

```java
Message next() {
    ...
    for (;;) {
        ...
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            ..
            Message msg = mMessages;
            if (msg != null && msg.target == null) {  // target为null的是SyncBarrier
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            ...
        }
        ...
    }
}
```

所以在VSync信号到来的时候只要往MessageQueue里面丢一个Asynchronous的Message就能保证它会优先执行了。

# 整体流程

上面讲的整个流程可以用下面这张图来大概表示:

{% img /view绘制流程一探到底一/6.png %}
