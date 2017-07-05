title: Android温故而知新 - Java的四种引用
date: 2017-07-06 02:59:46
tags:
    - 技术相关
    - Android
---
java对象的引用包括强引用，软引用，弱引用，虚引用四种
  
# 强引用

强引用是最常用的引用,我们在代码中处处可见:

```
String str = "hello world";
Map<String, String> map = new HashMap<>();
int[] arr = new int[10];
```

上面的str、map、arr都是强引用。一个对象,只要有强引用与它关联,那么JVM就不会回收它。即使是在内存不足的情况下,JVM宁愿抛出OutOfMemory的异常也不会回收它。

```
public void function() {
	Object object = new Object();
	Object[] array = new Object[9999];
}
```

比如上面的方法,当运行到Object[] array = new Object[9999];的时候,如果内存不够了,JVM就好抛出OutOfMemory的异常,而不会回收object的内存。所以一个强引用的内存肯定是有效的,所以java并不像c++,需要担心野指针的问题。

当然,当退出function之后,object和array就都已经不存在了,所以它们所指向的内存就可以被回收了。

当一个对象使用完,不会再被用到的时候,我们可以将所有指向它的强引用都赋为null。这样JVM会在合适的时机去回收它。

# 软引用

软引用所管理的对象在内存不足的时候,如果没有其他强引用与之管理,就会被回收。用SoftReference来表示软引用,使用方法如下:

```
public class DemoActivity extends AppCompatActivity {
    SoftReference<String> mStr = new SoftReference<>(new String("hello world"));

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_layout);
    }

    public void onClick(View view) {
        Log.d("SoftReference", "mStr = " + mStr.get());
    }
}
```

注意的是这里不能直接new SoftReference<>("hello world");因为JVM内建字符串池机制的存在会导致字符串池强引用的存在，因此不会被垃圾回收。


str.get()就可以获取到管理的对象,如果对象已经被回收就会返回null。

我们可以用Android Studio的Monitors强制gc,释放内存,然后这个时候就能看到它返回的是null了。

{% img /Android温故而知新-Java的四种引用/1.png %}

值得注意的是“SoftReference所管理的对象被回收”并不代表SoftReference的内存被回收, SoftReference此时依然是一个可以使用的对象,但它已经没有使用价值了。我们需要在合适的时候将SoftReference赋值为null,释放掉它所占用的内存,避免大量无用的SoftReference存在导致内存泄漏。

SoftReference也可以和ReferenceQueue一起使用。构造SoftReference的时候将ReferenceQueue传入SoftReference的构造方法。当SoftReference所管理的对象被回收的时候SoftReference就会被放到ReferenceQueue中。


```
public class DemoActivity extends AppCompatActivity {

    ReferenceQueue<String> mReferenceQueue = new ReferenceQueue<>();
    SoftReference<String> mStr = new SoftReference<>(new String("hello world"), mReferenceQueue);

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_layout);
    }

    public void onClick(View view) {
        if (mStr.get() == null) {
            Log.d("SoftReference", "mStr = " + mStr);

            mStr = null;
            SoftReference<String> ref;
            do {
                ref = (SoftReference<String>) mReferenceQueue.poll();
                Log.d("SoftReference", "ref = " + ref);

            } while (ref != null);
        }
    }
}
```

软引用的特性使得它很适合用来实现数据缓存,如图片缓存,网页缓存等。在内存紧张的时候如果没有其他的强引用关联,即该资源仅仅是放在缓存中而没有被使用,就会被释放。而当内存不紧张的时候,即使没有其他强引用与之关联,JVM的垃圾回收机制也是不会回收软引用所管理的资源的。

当需要使用的时候判断获取的是不是null,如果是的话证明之前内存被回收了,直接重新加载就好了。

# 弱引用

弱引用所管理的对象在JVM进行垃圾回收的时候,只要没有其他强引用与之关联。不管内存是否充足,都会被回收。它用WeakReference来表示。

弱引用可以用在回调函数中防止内存泄漏。我们来看个典型的例子,也是一个很多人都会犯的错误:

```
public class DemoActivity extends AppCompatActivity {
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_layout);
    }

    // 可能会引入内存泄漏
    private final Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
    };
}
```

不知道大家有没有看过我之前写的一篇[关于Handler的博客](http://www.jianshu.com/p/66a39db072d8),如果没有没有看过,而对Handler又不太熟悉的同学可以去看一下。

我们知道Handler是和Looper还有MessageQueue一起工作的。当安卓应用启动之后,系统会在主线程创建一个Looper和MessageQueue,它们的生命周期贯穿整个应用的生命周期。

Handler在发送Message的时候会将Message传到MessageQueue里面去,而Message里面保存着Handler的引用。这样的话如果Message还没有被处理,Handler也不会被回收。

而这里的Handler是DemoActivity的一个内部类。在java中,非晶体内部匿名类会持有外部类的一个隐式引用,这样就有可能导致外部类无法被垃圾回收。

如果我们代码中这样写:

```
public class DemoActivity extends AppCompatActivity {
    private static final int MSG_DO_SOMETHING = 1;
    
    // 可能会引入内存泄漏
    private final Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
        	...
        }
    };

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_layout);

        // 延迟十分钟发送消息
        mHandler.sendEmptyMessageDelayed(MSG_DO_SOMETHING, 1000 * 60 * 10);
    }
}
```

即使退出了DemoActivity,在消息没有被处理之前, DemoActivity的内存也是不会被回收的。

那要怎样解决它呢？

我们可以使用静态内部类加虚引用的方式:

```
public class DemoActivity extends AppCompatActivity {
    private static final int MSG_DO_SOMETHING = 1;
    // 可能会引入内存泄漏
    private static class InnerHandler extends Handler {
        private final WeakReference<DemoActivity> mActivity;

        InnerHandler(DemoActivity activity) {
            mActivity = new WeakReference<>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            DemoActivity activity = mActivity.get();
            if (activity != null) {
                ...
            }
        }
    }

    private final Handler mHandler = new InnerHandler(this);

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_layout);

        // 延迟十分钟发送消息
        mHandler.sendEmptyMessageDelayed(MSG_DO_SOMETHING, 1000 * 60 * 10);
    }
}
```

由于静态内部类不持有外部类的引用,所以现在只有虚引用关联了DemoActivity,在垃圾回收的时候,不管是否内存不足,DemoActivity都会被回收。

十分钟之后当handleMessage方法被调用的时候,用WeakReference的get方法获取DemoActivity,如果返回null的话证明DemoActivity已经被回收,就不应该再做什么处理了。

WeakReference同样可以在构造方法中传入ReferenceQueue。如果它所管理的对象被JVM回收，这个WeakReference就会被加入到ReferenceQueue。

# 虚引用

虚引用或者叫做幽灵引用在java中用PhantomReference表示。它和前面的应用都不一样,它不影响对象的生命周期,当一个对象只有虚引用与之关联的时候,就和没有任何引用一样。

而且它必须与ReferenceQueue一起使用,它只有一个构造函数:

```
public PhantomReference(T referent, ReferenceQueue<? super T> q) {
	super(referent, q);
}
```

而且它的get方法永远返回null:

```
public T get() {
	return null;
}
```

如果PhantomReference管理的对象只有PhantomReference与之关联,系统就会在这个时候或者一段时间之后将PhantomReference放到ReferenceQueue中。而不用等到垃圾回收的时候(参考[PhantomReference的文档](https://docs.oracle.com/javase/7/docs/api/java/lang/ref/PhantomReference.html)):

> If the garbage collector determines at a certain point in time that the referent of a phantom reference is phantom reachable, then at that time or at some later time it will enqueue the reference.

[这篇文章](http://www.milletblog.com/2016/09/J2SE%E8%BF%99%E6%89%8D%E6%98%AFjava%E8%99%9A%E5%BC%95%E7%94%A8PhantomReference-4/)对虚引用做了一个详细的介绍,其中对这一点他是这样解释的:

> 当一个虚引用被认为是一定会被垃圾回收器回收的时候，这个虚引用才会被注册到引用队列，而不会像软引用和弱引用必须等到垃圾回收器回收之后才会被注册到引用队列

对这个虚引用我其实理解的还不是很深入,查了很多的资料对它的讲解也是很模糊的。按我的理解,它应该就是单纯的用来判断一个对象是否被回收了。如果是,就进行一些清理操作。