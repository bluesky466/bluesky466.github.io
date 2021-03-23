title: 源码阅读计划 - LiveData
date: 2021-03-23 20:09:40
tags:
    - 技术相关
    - Android
---

LiveData是Jetpack里一个十分常用的组件，它是一个可以被观察的数据源。能够感知 Activity等的生命周期，在onStart或者onResume的时候才会回调监听。

这里举个简单的例子，我们在Activity中可以使用DataSource的observe方法去监听内部数据的改变，直接修改TextView，因为这个方法是回调在主线程的。而且可以看到DataSource里面是没有解注册的方法的，原因是LiveData会帮我们自动在LifecycleOwner的onDestory的时候解注册:

```kotlin
class DataSource {
    private val liveData = MutableLiveData<Int>()

    fun observe(owner: LifecycleOwner, observer: Observer<Int>) {
        liveData.observe(owner, observer)
    }

    fun loadData() {
        Thread(Runnable {
            // 加载数据
            //liveData.value = 123 // 主线程中
            liveData.postValue(123) // 子线程中
        }).start()
    }
}

// Activity中监听
dataSource.observe(this, Observer {data->
    textView.text = "data = ${data}"
})
```

知道用法之后我们来看看原理，瞧一瞧它是如何实现的。

# 注册监听

```java
private SafeIterableMap<Observer<? super T>, ObserverWrapper> mObservers =
            new SafeIterableMap<>();

@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    // 判断主线程  
    assertMainThread("observe");

    // 判断是否已经DESTROYED
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        return;
    }

    // 创建LifecycleBoundObserver
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
  

    // 判断是否已经添加过监听,如果没有就丢入mObservers这个map中保存
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }

    // 向Lifecycle注册监听
    owner.getLifecycle().addObserver(wrapper);
}   
```

首先是注册监听，这个代码十分简单，具体可以看我写的注释。它最核心的功能就是最后的向Lifecycle注册监听。

# 数据更新与通知

注册完监听之后我们看看数据的更新部分，如果在主线程的话直接调用LiveData的setValue方法即可更新数据:

```java
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}
```

这个方法逻辑也很清晰，总共四行就做了四件事情: 

1. 判断主线程
2. 更新数据版本
3. 更新数据
4. 分发更新事件

dispatchingValue里面判断传入的initiator是否为空，如果不为空则只通知该observer，否则通知所有的observer:

```java
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    ...

    if (initiator != null) {
        considerNotify(initiator);
        initiator = null;
    } else {
        for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
            considerNotify(iterator.next().getValue());
            ...
        }
    }
    ...
}
```

considerNotify方法也比较易懂，就是判断Observer是否已经激活，如果没有激活就返回。如果激活了再判断observer是否已经接收到最后一次修改数据的事件(反正Activity反复切换的时候重复通知)，如果没有就调用onChanged方法通知。

```java
private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    observer.mObserver.onChanged((T) mData);
}
```

这里的Active判断其实判断的observer的owner的是否至少是onStart的:

```java
@Override
boolean shouldBeActive() {
    return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
}
```

## 生命周期感知

也就是说如果Activity是在后台的，那么setValue的时候就不会去通知observer，这就是LiveData的生命周期感知机制。那么这个消息是否就丢失了呢？当然不是。

还记得observe方法里面向Lifecycle注册了监听吗:

```java
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    ...
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ...
    owner.getLifecycle().addObserver(wrapper);
}   
```

在Activity走到各个生命周期的时候LifeCycle就会回调LifecycleBoundObserver的onStateChanged方法:

```java
public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event) {
    if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
        removeObserver(mObserver);
        return;
    }
    activeStateChanged(shouldBeActive());
}
```

在这里面做两件事情:

1. 判断owner是否已经destroy了，如果是就自动解除监听
2. 如果没有destroy就修改激活状态

如果是start或者resume的activeStateChanged里面就会进行事件方法:

```java
void activeStateChanged(boolean newActive) {
    ...
    mActive = newActive;
    ...
    if (mActive) {
        dispatchingValue(this);
    }
}
```

dispatchingValue方法上面讲过，在这里会调用considerNotify去单独分发给initiator。:

```java
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    ...

    if (initiator != null) {
        considerNotify(initiator);
        initiator = null;
    } else {
        for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
            considerNotify(iterator.next().getValue());
            ...
        }
    }
    ...
}
```

于是observer就能监听到onStart之前发送的消息了。

# 粘性事件

LiveData的事件都是粘性事件，什么是粘性事件呢？简单来讲就是注册的监听可以接收到注册之前的事件:

```kotlin
class DataSource {
    private val liveData = MutableLiveData<Int>()

    fun observe(owner: LifecycleOwner, observer: Observer<Int>) {
        liveData.observe(owner, observer)
    }

    fun loadData() {
        liveData.value = 123
    }
}

// Activity
override fun onResume() {
    super.onResume()
        // 先发送
    dataSource.loadData()
        // 再注册
    dataSource.observe(this, Observer { data ->
        Log.d("testtest", "data = ${data}")
    })
}
```

这个其实结合上面的更新流程，去到considerNotify里面只要mLastVersion小于LiverData.mVersion就会被通知

```java
private void considerNotify(ObserverWrapper observer) {
    ...
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    observer.mObserver.onChanged((T) mData);
}
```

而在创建ObserverWrapper的时候mLastVersion初始化是-1:

```java
public abstract class LiveData<T> {
    ...
    static final int START_VERSION = -1;
    ...
    private abstract class ObserverWrapper {
        ...
        int mLastVersion = START_VERSION;
        ...
    }
    ...
}
```

于是注册的时候就会直接通知到LiveData之前设置的数据了。

## 实现非粘性事件

如果我们不想要这个粘性事件要怎么做呢？从根本原因上来讲是因为observe创建ObserverWrapper的时候mLastVersion直接设置成了-1，所以在分发的时候肯定会被分发一次。也就是说只要我们在创建的时候将mLastVersion设置成当前LiveData的mVersion就能避免了。

但是无论mLastVersion还是mVersion都是没有对外公开的，所以很容易想到继承LiveData在observe的时候反射去修改mLastVersion。但是反射的代码比较恶心，我就不贴出来了。我这里再提供另外一种思路:

```kotlin
class MyLiveData<T> : LiveData<T>() {
    var version = -1

    public override fun setValue(value: T) {
        version++
        super.setValue(value)
    }

    override fun observe(owner: LifecycleOwner, observer: Observer<in T>) {
        super.observe(owner, ObserverWrapper(observer, version))
    }

    private inner class ObserverWrapper<T>(
        private val observer: Observer<in T>,
        private var version: Int
    ) :
        Observer<T> {
        override fun onChanged(t: T) {
            if (this.version < this@MyLiveData.version) {
                observer.onChanged(t)
            }

            this.version = this@MyLiveData.version
        }
    }
}
```

代码很简单，我们自己使用装饰器模式再包装了一层维护了LiveData的version和Observer的version进行判断。

# postValue

在上面我们看到setValue是会判断主线程的:

```java
protected void setValue(T value) {
    assertMainThread("setValue");
    ...
}
```

如果我们在子线程里面想要修改数据需要使用postValue:

```java
protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}

private final Runnable mPostValueRunnable = new Runnable() {
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        setValue((T) newValue);
    }
};
```

它的原理其实很简单，将value保存到mPendingData，然后用postToMainThread(最底层其实是Handler.post)将mPostValueRunnable丢到主线程执行setValue(mPendingData)方法.

而且在postValue会判断之前是否有设置过mPendingData，如果有代表已经post过Runnable，于是就不再post第二个，也就是说最多只有一个Runnable会被post。

于是这就有个问题了，就是在子线程中多次postValue，只有最后一个value会在主线程中最终设置。

但其实这个也并不是什么问题，大多数情况下主线程其实并不关心数据是如何一步步更新的，只关心最终的结果。

# observeForever

observe注册的观察者都是生命周期感知的，如果不是start活着resume状态的话是不能立马接收到消息的。如果我们需要非生命周期感知的话可以使用observeForever方法，里面创建的是AlwaysActiveObserver而不是observe里看到的LifecycleBoundObserver:

```java
public void observeForever(@NonNull Observer<? super T> observer) {
    ...
    AlwaysActiveObserver wrapper = new AlwaysActiveObserver(observer);
    ...
}
```

它的特点在于shouldBeActive永远是true:

```java
private class AlwaysActiveObserver extends ObserverWrapper {
    ...
    @Override
    boolean shouldBeActive() {
        return true;
    }
}
```

这就意味着considerNotify里面肯定会继续往下走到version判断那里，不会因为生命周期而提前结束:

```java
private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    if (!observer.shouldBeActive()) { // AlwaysActiveObserver永远为true
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    observer.mObserver.onChanged((T) mData);
}
```

