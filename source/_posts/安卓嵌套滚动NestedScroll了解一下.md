title: 安卓嵌套滚动NestedScroll了解一下
date: 2018-06-04 08:07:50
tags:
    - 技术相关
    - Android
---


其实嵌套滚动已经算一个比较常见的特效了,下面这个动图就是嵌套滚动的一个例子:

{% img /安卓嵌套滚动NestedScroll了解一下/1.gif %}

看到这个动效,大家可能都知道可以用CoordinatorLayout去实现.其实CoordinatorLayout是基于NestedScroll机制去实现的,而我们直接通过NestedScroll机制也能很方便的实现这个动效.


# 原理

NestedScroll的其实很简单.

一般的触摸消息的分发都是从外向内的,由外层的ViewGroup的dispatchTouchEvent方法调用到内层的View的dispatchTouchEvent方法.

而NestedScroll提供了一个反向的机制,内层的view在接收到ACTION_MOVE的时候,将滚动消息先传回给外层的ViewGroup,看外层的ViewGroup是不是需要消耗一部分的移动,然后内层的View再去消耗剩下的移动.内层view可以消耗剩下的滚动的一部分,如果还没有消耗完,外层的view可以再选择把最后剩下的滚动消耗掉.

上面的描述可能有点绕,可以看下面的图来帮助理解:

{% img /安卓嵌套滚动NestedScroll了解一下/2.png %}

# 具体实现

NestedScroll机制会涉及到四个类:

NestedScrollingChild， NestedScrollingChildHelper 和 NestedScrollingParent ， NestedScrollingParentHelper

NestedScrollingChild和NestedScrollingParent是两个接口,我们先看看他们的声明:

```
public interface NestedScrollingChild {
    public void setNestedScrollingEnabled(boolean enabled);

    public boolean isNestedScrollingEnabled();

    public boolean startNestedScroll(int axes);

    public void stopNestedScroll();

    public boolean hasNestedScrollingParent();

    public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow);

    public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow);

    public boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed);

    public boolean dispatchNestedPreFling(float velocityX, float velocityY);
}

public interface NestedScrollingParent {
    public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes);

    public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes);

    public void onStopNestedScroll(View target);

    public void onNestedScroll(View target, int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed);

    public void onNestedPreScroll(View target, int dx, int dy, int[] consumed);

    public boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed);

    public boolean onNestedPreFling(View target, float velocityX, float velocityY);

    public int getNestedScrollAxes();
}
```

这里真正重要的其实是NestedScrollingParent的几个方法,因为其他方法都能直接让NestedScrollingChildHelper或者NestedScrollingParentHelper去代理:

- onStartNestedScroll 是否接受嵌套滚动,只有它返回true,后面的其他方法才会被调用
- onNestedPreScroll 在内层view处理滚动事件前先被调用,可以让外层view先消耗部分滚动
- onNestedScroll 在内层view将剩下的滚动消耗完之后调用,可以在这里处理最后剩下的滚动
- onNestedPreFling 在内层view的Fling事件处理之前被调用
- onNestedFling 在内层view的Fling事件处理完之后调用

我们只要让子view和父view分别实现NestedScrollingChild和NestedScrollingParent接口,然后分别调用NestedScrollingChildHelper和NestedScrollingParentHelper的对应方法去代理一些具体功能,然后在NestedScrollingChild的onTouchEvent那里根据需求调用startNestedScroll/dispatchNestedPreScroll/stopNestedScroll就能实现嵌套滚动了:

```
//NestedScrollingChild
private NestedScrollingChildHelper mHelper = new NestedScrollingChildHelper(this);

public boolean startNestedScroll(int axes) {
  return mHelper.startNestedScroll(axes);
}
public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow) {
  return mHelper.dispatchNestedScroll(dxConsumed,  dyConsumed,
             dxUnconsumed,  dyUnconsumed, offsetInWindow);
}
...
```

```
//NestedScrollingParent
private NestedScrollingParentHelper mHelper = new NestedScrollingParentHelper(this);

public void onNestedScrollAccepted(View child, View target, int axes) {
  mHelper.onNestedScrollAccepted(child, target, axes);
}

public int getNestedScrollAxes() {
  return mHelper.getNestedScrollAxes();
}
...
```

但是如果你使用sdk21及以上的版本,NestedScroll机制已经直接集成到了View中了,你只需要直接重写View的对应方法就好

## 布局

我们先看布局文件

```
<me.linjw.nestedscrolldemo.NestedScrollParentView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <FrameLayout
        android:id="@+id/header"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <ImageView
            android:layout_width="match_parent"
            android:layout_height="200dp"
            android:src="@mipmap/ic_launcher" />
    </FrameLayout>

    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="@color/colorAccent"
        android:text="Title"
        android:textAlignment="center"
        android:textSize="20dp" />

    <android.support.v7.widget.RecyclerView
        android:id="@+id/list"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />
</me.linjw.nestedscrolldemo.NestedScrollParentView>
```

最外层是我们自定义的NestedScrollParentView,其实它是一个LinearLayout,内部竖直排列了三个子view:

- 一个由FrameLayout包裹的ImageView
- 一个TextView
- 一个RecyclerView

## 代码

为了简便起见,我们先直接用sdk22的版本用重写View方法的方式去实现它.

NestedScrollParentView中有两个方法比较重要,嵌套滚动基本上就是由这两个方法实现的:

```
  @Override
    public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
        return true;
    }

    @Override
    public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
        super.onNestedPreScroll(target, dx, dy, consumed);

        boolean headerScrollUp = dy > 0 && getScrollY() < mHeaderHeight;
        boolean headerScrollDown = dy < 0 && getScrollY() > 0 && !target.canScrollVertically(-1);
        if (headerScrollUp || headerScrollDown) {
            scrollBy(0, dy);
            consumed[1] = dy;
        }
    }
```

- onStartNestedScroll 这个方法如果返回true的话代表接受由内层传来的滚动消息,我们直接返回true就好,否则后面的消息都接受不到

- onNestedPreScroll 这个方法用于消耗内层view的一部分滚动.我们需要将消耗掉的滚动存到counsumed中让consumed知道.例如我们这里在顶部的FrameLayout需要移动的情况下会消耗掉所有的dy,这样内层的view(即RecyclerView)就不会滚动了.

这里的mHeaderHeight保存的是顶部的FrameLayout的高度:

```
 @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
        mHeaderHeight = mHeader.getMeasuredHeight();
    }
```

到这里基本上就实现了动图的效果,是不是很简单?

完整代码可以参考 https://github.com/bluesky466/NestedScrollDemo/tree/sdk22
