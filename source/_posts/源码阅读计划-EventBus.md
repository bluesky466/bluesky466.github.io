title: 源码阅读计划 - EventBus
date: 2021-08-12 22:26:26
tags:
    - 技术相关
    - Android
---
EventBus的api很简单:

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        findViewById(R.id.btn).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                EventBus.getDefault().post(new MyEvent(123));
            }
        });
    }

    @Override
    protected void onResume() {
        super.onResume();
        EventBus.getDefault().register(this);
    }

    @Override
    protected void onPause() {
        super.onPause();
        EventBus.getDefault().unregister(this);
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onMyEvent(MyEvent event) {
        Log.d("testtest", "onMyEvent " + event.data);
    }
}

class MyEvent {
    public int data;

    MyEvent(int data) {
        this.data = data;
    }
}
```

# 注册监听的原理

坦白讲内部的实现原理也挺简单的，我们从注册开始看:

```java
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    // 从subscriber的class从查找回调方法
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            // SubscriberMethod里面是方法的反射，所以需要绑定回调方法和观察者对象
            subscribe(subscriber, subscriberMethod);
        }
    }
}

private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    // 使用Event的class做key，查找这个Event的所有观察者
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    ...
    // 按照优先级插入观察者
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }
    ...
}
```

使用Finder从对象的Class中查找注册的回调方法的信息，使用Subscription将调用者与方法信息绑定起来，然后使用Event的Class作为key将它放到subscriptionsByEventType这个map中。由于同一种Event可能会有多个观察者，所以map的value是一个按优先级排序的List。

SubscriberMethod和Subscription的定义如下:

```java
public class SubscriberMethod {
    final Method method;         // 回调方法
    final ThreadMode threadMode; // 回调的线程策略
    final Class<?> eventType;    // 监听的事件类型
    final int priority;          // 优先级
    final boolean sticky;        // 是否监听粘性事件
    ...
}

final class Subscription {
    final Object subscriber; //观察者的对象引用，可以用来反射调用subscriberMethod.method
    final SubscriberMethod subscriberMethod;
    ...
}
```

从上面可以大概猜出来，事件的分发实际是反射调用的方法。

也就是说当我们使用post方法发送Event的时候就能用Event的Class查找到所有的Subscription，通过Subscription的subscriber找到注册的对象、subscriberMethod找到注册的方法，接着就能使用反射去进行分发。

整个注册的流程大体上是比较清晰的，但是findSubscriberMethods的查找注册信息流程中有些小的细节也比较值得学习。

## 1.缓存

第一点是扫描完一个class之后会将监听的信息放入METHOD\_CACHE缓存中，例如我们的demo，onResume的时候register，onPause的时候unregister，就能减少第二次onResume的耗时。

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    // METHOD_CACHE会保存查找过的subscriberClass的监听信息，防止重复查找耗时
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }
    ...
    subscriberMethods = findUsingInfo(subscriberClass);
    ...
    // 加入缓存，下次可以直接使用
    METHOD_CACHE.put(subscriberClass, subscriberMethods);
    return subscriberMethods;
    ...
}
```

## 2.APT生成索引

第二点就是由于反射遍历类方法去查找被@Subscribe修饰的方法比较耗时，所以可以使用apt编译时根据注解生成代码的方式直接生成索引:

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            // 优先从apt生成的索引查找，如果找不到就是用反射去遍历class，查找@Subscribe修饰的方法
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                ...
            } else {
                // 使用反射查找
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

apt创建索引的使用方法见[官方文档](https://greenrobot.org/eventbus/documentation/subscriber-index/)

## 3.对象池减少内存碎片

第三就是FindState是一个辅助查找的工具类，为了避免应用初始化的时候多个类都在注册到EventBus，导致这个FindState创建消耗多次，产生内存碎片，所以它使用类对象池技术:

```java
// 获取
private FindState prepareFindState() {
    synchronized (FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            FindState state = FIND_STATE_POOL[i];
            if (state != null) {
                FIND_STATE_POOL[i] = null;
                return state;
            }
        }
    }
    return new FindState();
}

// 回收
private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
    List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
    findState.recycle();
    synchronized (FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            if (FIND_STATE_POOL[i] == null) {
                FIND_STATE_POOL[i] = findState;
                break;
            }
        }
    }
    return subscriberMethods;
}
```

# 消息分发原理

上面我们看完了注册，实际上分发的流程大概也能猜出来了。不过这里面还是有蛮多小细节同样值得学习的。

## 1.Event Queue

EventBus的Event分发并不是直接循环遍历观察者进行分发，而是先将Event放到队列中，然后再去队列里面拿出来分发。

PostingThreadState使用了ThreadLocal，每条线程都有自己的Event队列，而且isPosting能够记录当前的线程是否正在分发Event。

也就是说如果在Event的回调中再去post一个Event，并不会立刻分发，而是会等之前的Event都分发完之后在去分发。

```java
public void post(Object event) {
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    //将Event丢到队列中
    eventQueue.add(event);

    if (!postingState.isPosting) { // 是否正在分发,例如在监听的回调中post,isPosting就是true
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            // 从队列中拿出Event进行分发，直至队列为空
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```

__这里有个小坑:threadMode是POSTING，只能代表回调是在post的线程调用的，并不能代表post方法里面会立即调用__

## 2.eventInheritance

当一个观察者监听的是父类，如果post了一个子类的Event，默认情况下观察者是可以接收到这个子类的Event。这是因为默认情况下EventBusBuilder.eventInheritance为true:

```java
boolean eventInheritance = true;
```

如果不想要这个功能我们可以设置成false。它的原理在postSingleEvent里面可以看到:

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    if (eventInheritance) {
        // 如果eventInheritance为true，查找eventClass的父类、实现的interface等去分发
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        // eventInheritance为false则只分发这个eventClass
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    ...
}
```

postSingleEventForEventType方法就是根据Class查找register的对应Subscription，调用postToSubscription进行实际的分发操作

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        for (Subscription subscription : subscriptions) {
            ...
            postToSubscription(subscription, event, postingState.isMainThread);
            ...
        }
        return true;
    }
    return false;
}
```

## 3.threadMode原理

postToSubscription内部就会根据不同的threadMode在不同线程使用反射调用注册的方法:

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            ..
            break;
        case MAIN:
            ...
            break;
        case MAIN_ORDERED:
            ...
            break;
        case BACKGROUND:
            ...
            break;
        case ASYNC:
            ...
            break;
        default:
            ...
    }
}
```

