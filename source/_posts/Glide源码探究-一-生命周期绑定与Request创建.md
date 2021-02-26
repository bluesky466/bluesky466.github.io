title: Glide源码探究(一) - 生命周期绑定与Request创建
date: 2021-02-05 21:01:23
tags:
    - 技术相关
    - Android
---

系列文章:

- [Glide源码探究(一) - 生命周期绑定与Request创建](https://blog.islinjw.cn/2021/02/05/Glide%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6-%E4%B8%80-%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E7%BB%91%E5%AE%9A%E4%B8%8ERequest%E5%88%9B%E5%BB%BA/)
- [Glide源码探究(二) - 内存缓存](https://blog.islinjw.cn/2021/02/08/Glide%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6-%E4%BA%8C-%E5%86%85%E5%AD%98%E7%BC%93%E5%AD%98/)
- [Glide源码探究(三) - 网络资源加载](https://blog.islinjw.cn/2021/02/10/Glide%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6-%E4%B8%89-%E7%BD%91%E7%BB%9C%E8%B5%84%E6%BA%90%E5%8A%A0%E8%BD%BD/)



蛮久之前囫囵吞枣地瞄过Glide部分源码，最近由于某个契机又心血来潮比较系统的过了一遍它的源码，发现它的蛮多设计还是比较有意思的(该系列笔记基于Glide 4.12.0这个版本进行分析)。

首先Glide的使用十分简单，只需要三行代码就能完成图片的下载、缓存和显示:

```kotlin
Glide.with(this)
    .load(url)
    .into(img)
```

但为了我们可以方便的使用，实际上每一行代码的背后都帮我们做了不少脏活累活。

# Glide.with

我们一行行来看，首先看Glide.with方法，它其实有一系列的重载:

```java
public static RequestManager with(@NonNull Context context) {
  return getRetriever(context).get(context);
}

public static RequestManager with(@NonNull Activity activity) {
  return getRetriever(activity).get(activity);
}

public static RequestManager with(@NonNull FragmentActivity activity) {
  return getRetriever(activity).get(activity);
}

public static RequestManager with(@NonNull Fragment fragment) {
  return getRetriever(fragment.getContext()).get(fragment);
}

public static RequestManager with(@NonNull android.app.Fragment fragment) {
  return getRetriever(fragment.getActivity()).get(fragment);
}

public static RequestManager with(@NonNull View view) {
  return getRetriever(view.getContext()).get(view);
}
```

内容大同小异，都是用getRetriever方法获取RequestManagerRetriever，然后从里面get出RequestManager。

## Glide 单例模式实现

追踪getRetriever方法，可以比较容易看出来实际上是从Glide单例里面获取RequestManagerRetriever，而且我们还能看到Glide使用的是单例模式的双重校验锁方式:

```java
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
        ...
    return Glide.get(context).getRequestManagerRetriever();
}

public static Glide get(@NonNull Context context) {
    if (glide == null) {
      ...
      synchronized (Glide.class) {
        if (glide == null) {
          checkAndInitializeGlide(context, annotationGeneratedModule);
        }
      }
    }

    return glide;
}
```

## Glide生命周期绑定

glide其实已经帮我们对activty生命周期进行了绑定：在onStop的时候停止加载，在onStart的时候继续加载，在onDestory清除任务和进行缓存的回收。它的原理是通过在Activity里面添加一个不可见的fragment，在里面监听生命周期。

我们可以看到在RequestManagerRetriever.get方法最终会add一个SupportRequestManagerFragment到Activity里面:

```java
public RequestManager get(@NonNull FragmentActivity activity) {
    ...
    FragmentManager fm = activity.getSupportFragmentManager();
    return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    ...
}

@NonNull
private RequestManager supportFragmentGet(Context context, FragmentManager fm, Fragment parentHint, boolean isParentVisible) {
    SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm, parentHint);
    RequestManager requestManager = current.getRequestManager();
    ...
    return requestManager;
}

@NonNull
private SupportRequestManagerFragment getSupportRequestManagerFragment(final FragmentManager fm, Fragment parentHint) {
    SupportRequestManagerFragment current = (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
        ...
        current = new SupportRequestManagerFragment();
        ...
        fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
        ...
    }
    return current;
}
```

SupportRequestManagerFragment里面有个lifecycle成员，我们可以向它注册监听，RequestManager也正是这么干的:

```java
public class SupportRequestManagerFragment extends Fragment {
    ...
  private final ActivityFragmentLifecycle lifecycle;
  ...
  public SupportRequestManagerFragment(@NonNull ActivityFragmentLifecycle lifecycle) {
    this.lifecycle = lifecycle;
  }
  ...
  @Override
  public void onStart() {
    super.onStart();
    lifecycle.onStart();
  }
  @Override
  public void onStop() {
    super.onStop();
    lifecycle.onStop();
  }
  @Override
  public void onDestroy() {
    super.onDestroy();
    lifecycle.onDestroy();
    unregisterFragmentWithRoot();
  }
  ...
}
```

```java
//RequestManager
RequestManager(
      Glide glide,
      Lifecycle lifecycle,
      RequestManagerTreeNode treeNode,
      RequestTracker requestTracker,
      ConnectivityMonitorFactory factory,
      Context context) {
    ...
    lifecycle.addListener(connectivityMonitor);
    ...
}

public synchronized void onStart() {
    resumeRequests();
    targetTracker.onStart();
}

@Override
public synchronized void onStop() {
    pauseRequests();
    targetTracker.onStop();
}

@Override
public synchronized void onDestroy() {
    targetTracker.onDestroy();
    for (Target<?> target : targetTracker.getAll()) {
      clear(target);
    }
    targetTracker.clear();
    requestTracker.clearRequests();
    ...
}
```

这种添加不可见Fragment监听生命周期的技巧还是挺实用的。RxPermissions里面也用了这种技巧去监听onActivityResult回调

# RequestManager.load

第二行的load方法，虽然只有一行但是实际上它做了两件事情:

```java
public RequestBuilder<Drawable> load(@Nullable String string) {
    return asDrawable().load(string);
}
```

1. new了一个请求Drawable的RequestBuilder:

```java
public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
}

public <ResourceType> RequestBuilder<ResourceType> as(
      @NonNull Class<ResourceType> resourceClass) {
    return new RequestBuilder<>(glide, this, resourceClass, context);
}
```

2. 往builder里设置了model:

```java
public RequestBuilder<TranscodeType> load(@Nullable String string) {
    return loadGeneric(string);
}
private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    ...
    this.model = model;
    ...
}
```

没错这里使用的是Builder模式。

# RequestManager.into

into实际上也是干了两件事情，build了一个Request出来，然后runRequest执行它:

```java
// RequestBuilder
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    ...
    return into(
        glideContext.buildImageViewTarget(view, transcodeClass),
        /*targetListener=*/ null,
        requestOptions,
        Executors.mainThreadExecutor());
}

private <Y extends Target<TranscodeType>> Y into(
  @NonNull Y target,
  @Nullable RequestListener<TranscodeType> targetListener,
  BaseRequestOptions<?> options,
  Executor callbackExecutor) {
    ...
    Request request = buildRequest(target, targetListener, options, callbackExecutor);
    ...
  target.setRequest(request);
    requestManager.track(target, request);
    return target;
}

// RequestManager
synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
    targetTracker.track(target);
    requestTracker.runRequest(request);
}
```

这里其实还有个细节，我们展开讲讲。如果我们中途想取消对该view的图片加载，可以用下面两行代码进行取消:

```java
Glide.with(this)
    .clear(img)
```

这样就要求我们要记录view和request的关联，可能有些同学会想到用map<View,Request>这样的形式去保存，但是这样的话需要我们额外做出一些手段去防止view的内存泄露(如弱引用等)。其实安卓里比较常用的手段是将需要关联的东西用setTag方法保存到view里面:

```java
// ViewTarget
public void setRequest(@Nullable Request request) {
    setTag(request);
}

private void setTag(@Nullable Object tag) {
    isTagUsedAtLeastOnce = true;
    view.setTag(tagId, tag);
}
```

于是clear的时候只需要用View.getTag把request再拿出来就好:

```java
public void clear(@NonNull View view) {
    clear(new ClearTarget(view));
}

public void clear(@Nullable final Target<?> target) {
    if (target == null) {
      return;
    }
    untrackOrDelegate(target);
}

private void untrackOrDelegate(@NonNull Target<?> target) {
    boolean isOwnedByUs = untrack(target);
  Request request = target.getRequest();
  if (!isOwnedByUs && !glide.removeFromManagers(target) && request != null) {
      target.setRequest(null);
      request.clear();
  }
}
```

# request的执行

runRequest的逻辑其实也比较简单，如果没有pause就begin request，如果不是就放到pendingRequests列表里面等待后面再begin:

```java
public void runRequest(@NonNull Request request) {
    requests.add(request);
    if (!isPaused) {
        request.begin();
    } else {
        request.clear();
        ...
        pendingRequests.add(request);
    }
}
```

而request的begin方法的核心流程就是先获取图片的尺寸，然后使用engine.load去启动真正的加载逻辑:

```java
public void begin() {
    ...
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        onSizeReady(overrideWidth, overrideHeight);
    } else {
        target.getSize(this);
    }
    ...
}

public void onSizeReady(int width, int height) {
    ...
    loadStatus =
      engine.load(
          glideContext,
          model,
          requestOptions.getSignature(),
          this.width,
          this.height,
          requestOptions.getResourceClass(),
          transcodeClass,
          priority,
          requestOptions.getDiskCacheStrategy(),
          requestOptions.getTransformations(),
          requestOptions.isTransformationRequired(),
          requestOptions.isScaleOnlyOrNoTransform(),
          requestOptions.getOptions(),
          requestOptions.isMemoryCacheable(),
          requestOptions.getUseUnlimitedSourceGeneratorsPool(),
          requestOptions.getUseAnimationPool(),
          requestOptions.getOnlyRetrieveFromCache(),
          this,
          callbackExecutor);
    ...
}
```

到这里其实一切都比较好理解，下一步就到了Engine这个Glide的核心模块，Glide的实际加载和多级缓存都是由它去调度的，由于该模块比较复杂，我们留到下一篇单独来聊。

