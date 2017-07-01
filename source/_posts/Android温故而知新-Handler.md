title: Android温故而知新 - Handler
date: 2017-07-02 00:13:13
tags:
    - 技术相关
    - Android
---

我们都知道,安卓主线程(也就是ui线程)中不能做耗时操作,一旦主线程阻塞了超过5秒钟就会被系统强制关闭,甚至在主线程中访问网络都会直接抛异常。但是我们的ui操作又必须在主线程中进行。所以我们会在子线程中进行耗时的操作,完成之后将结果同步到主线程进行ui的刷新。

而Handler机制就是谷歌用来方便我们进行线程同步的,我们可以很方便的通过它,在子线程中将ui刷新的操作同步回主线程中进行。

# 使用Handler将ui刷新操作同步到主线程中进行

我们先来看一个例子直观感受下如何使用Handler将ui刷新操作从子线程同步到主线程中进行:

```
public class MainActivity extends AppCompatActivity {
    private static final int MSG_UPDATE_PROGRESS_BAR_ABOVE = 1;
    private static final int MSG_UPDATE_PROGRESS_BAR_BELOW = 2;
    private ProgressBar mProgressBarAbove;
    private ProgressBar mProgressBarBelow;

    private Handler mHandler;


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mProgressBarAbove = (ProgressBar) findViewById(R.id.progressAbove);

        mProgressBarBelow = (ProgressBar) findViewById(R.id.progressBelow);

        mHandler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);

                switch (msg.what) {
                    case MSG_UPDATE_PROGRESS_BAR_ABOVE:
                        mProgressBarAbove.setProgress(msg.arg1);
                        break;
                    case MSG_UPDATE_PROGRESS_BAR_BELOW:
                        Bundle data = msg.getData();
                        mProgressBarBelow.setProgress(data.getInt("progress"));
                        break;
                }
            }
        };

        new Thread(new Runnable() {
            @Override
            public void run() {
                int progressAbove = 0;
                int progressBelow = 0;

                while (progressAbove < 100 || progressBelow < 100) {
                    if (progressAbove < 100) {
                        progressAbove++;

                        Message above = new Message();
                        above.what = MSG_UPDATE_PROGRESS_BAR_ABOVE;
                        above.arg1 = progressAbove;
                        mHandler.sendMessage(above);
                    }

                    if (progressBelow < 100) {
                        progressBelow += 2;

                        Message below = mHandler.obtainMessage();
                        below.what = MSG_UPDATE_PROGRESS_BAR_BELOW;
                        Bundle data = new Bundle();
                        data.putInt("progress", progressBelow);
                        below.setData(data);
                        mHandler.sendMessage(below);
                    }

                    try {
                        Thread.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
    }
}
```

上面的例子很简单,界面有上下两条进度条,我们在子线程中使用Thread.sleep(100)模拟耗时操作,每隔100毫秒更新一下进度,上面的进度条进度每次加1,下面的进度条每次加2。

{% img /Android温故而知新-Handler/1.png %}

## 1.创建handler并重写handleMessage方法

首先我们会创建一个Handler并重写它的handleMessage,这个方法就是在主线程中被调用的,我们通过传给这个方法的Message去刷新ui。

Message的what成员变量用来标识消息的类型,我们这里用来区分更新哪一个进度条。同时我们可以从Message中取得从子线程中传过来的进度,然后直接在handleMessage里面刷新进度条的进度。

## 2.在子线程中发送Message给Handler

Message是在子线程中被创建的。如代码所示,我们可以直接将它new出来,也可以使用mHandler.obtainMessage()从mHandler的Message池中获取一个实例。

<font color=#FF0000>一般推荐使用obtainMessage的方式</font>,因为Message池中的Message是可以被重复利用的,避免了创建对象申请内存的开销。

在前面说过Message的what成员变量是用来标志消息的类型的,我们这里直接将MSG_UPDATE_PROGRESS_BAR_ABOVE或者MSG_UPDATE_PROGRESS_BAR_ABOVE赋值进去,在handleMessage的时候就能用它来区分到底更新哪个进度条了。

消息的值也有多种赋值方式。

第一种很简单,Message提供了arg1、arg2、obj、replyTo等public成员变量,可以直接将想保存的数据赋值給他们,在handleMessage方法中就能直接获取到他们了。

第二种就是创建一个Bundle对象,在Bundle对象中存入数据,然后再通过setData方法传给Message,在handleMessage方法中通过getData可以获得Message中保存的Bundle对象,从而获得保存的数据。

# 同步到主线程的各种姿势

使用Handler将操作同步到主线程中进行有两种方式,一种是上面例子中的发送Message的方式。另一种是直接将一个Runnable传给Handler,Handler就会在主线程中执行它:

- sendEmptyMessage(int what)
- sendEmptyMessageDelayed(int what, long delayMillis)
- sendEmptyMessageAtTime(int what, long uptimeMillis)
- sendMessage(Message msg)
- sendMessageDelayed(Message msg, long delayMillis)
- sendMessageAtTime(Message msg, long uptimeMillis)
- sendMessageAtFrontOfQueue(Message msg)
- post(Runnable r)
- postDelayed(Runnable r, long delayMillis)
- postAtTime(Runnable r, long uptimeMillis)
- postAtTime(Runnable r, Object token, long uptimeMillis)
- postAtFrontOfQueue(Runnable r)

上面就是可以使用的一些方法,send前缀的方法用于发送一个带数据的Message对象,post前缀的方法用于安排一个Runnable对象到主线程中执行。他们都有延迟发送,定时发送等姿势可以使用。

当然你也可以在Message或者Runnable未同步到主线程的时候使用下面的remove方法将他们取消:

- removeMessages(int what) 
- removeMessages(int what, Object object)
- removeCallbacks(Runnable r)
- removeCallbacks(Runnable r, Object token)
- removeCallbacksAndMessages(Object token)

实际上将Runnable post到Handler中的时候也是用Message去包装的:

```
	public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
    
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
```

在主线程分发消息的时候如果判断到Message有callback则会直接执行callback,否则就将消息传到handleMessage中进行处理:

```
	/**
     * Handle system messages here.
     */
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
```

# Handler机制的基本原理

Handler机制有四个重要的组件:

- Handler
- Message
- MessageQueue
- Looper

Handler和Message通过前面的例子应该已经很清楚了,但是MessageQueue和Looper又是什么鬼？

MessageQueue顾名思义,就是Message的队列,我们调用Handler的各种方法发送Message其实就是将Message放到MessageQueue中。

而Looper就将Message从MessageQueue中拿出来。Looper有一个loop方法,它里面有个死循环,不断从MessageQueue中拿Message出来并且将它传给Handler去处理。

我们在子线程中将Message放入MessageQueue,然后在主线程中运行Looper的loop方法,不断从MessageQueue中获取Message。这就是Message从子线程同步到主线程的原理。

我画了一幅图来更加形象的展示这个机制:

{% img /Android温故而知新-Handler/2.png %}

# 主线程中的Looper

有人会问了,我们也没有在主线程中中调用Looper的loop方法啊,而且再说了loop中不是一个死循环吗,如果在主线程中运行它的话不会被堵死吗？

其实安卓在启动主线程的时候就会自动创建一个Looper和执行Looper.loop()的了,不需要自己去手动操作。

至于第二个问题,我们可以直接开口安卓的源码,我们可以在[androidxref](http://androidxref.com/)这个网址中在线浏览多个版本的安卓源码。

一般来讲我们认为ActivityThread.main(String[] args)就是安卓程序运行的入口,也就是我们熟悉的main方法。它其实很短,我们在它的最后可以看到Looper.loop()这个方法的确是被调用了的。

```
	public static void main(String[] args) {
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        Security.addProvider(new AndroidKeyStoreProvider());

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        AsyncTask.init();

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

但是调用了loop方法之后,线程不就被堵住了吗？那主线程又是怎样接收到按键消息和调用各种生命周期方法的？我们可以看到代码里还有个sMainThreadHandler,这个sMainThreadHandler是个H类,它的定义如下:

```
private class H extends Handler {
	public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    ...
                } break;
                case RELAUNCH_ACTIVITY: {
                    ...
                } break;
                ...
            }
        }
    }
}
```

看Activity的各个生命周期，还有事件处理也是通过Handler机制实现的！

# 使用Handler将消息同步到其他线程

根据上面的原理,其实我们不仅可以使用Handler将消息同步到主线程中,也能用它来将消息从主线程同步到子线程中去执行。

只需要在子线程中运行Looper的loop方法,让它不断获取Message,然后在主线程中发送Message就能在子线程中被处理了。

代码如下

```
    private Handler mHandler;
    
    private Thread mThread = new Thread(new Runnable() {
        @Override
        public void run() {
            Looper.prepare();

            mHandler = new Handler() {
                public void handleMessage(Message msg) {
                    // process incoming messages here
                }
            };
            Looper.loop();
        }
    });
```

我们在需要在子线程中先调用Looper.prepare()。这个是个静态方法它用来创建一个Looper并绑定到当前的线程中。

然后创建Handler,Handler会自动绑定当前线程中的Looper。

最后调用Looper.loop()就大功告成了。

之后我们就能在主线程中使用mHandler将消息发送到子线程中处理了。

# HandlerThread

在上面一节中我们看到,在子线程中创建Handler还需要手动调用Looper.prepare()和Looper.loop()。为了简化操作,谷歌官方提供了HandlerThread给我们使用。

HandlerThread是Thread的子类,当HandlerThread启动的时候会自动调用Looper.prepare()和Looper.loop(),它的run方法源码如下:

```
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```

于是我们只需要在Handler构造的时候传入HandlerThread的Looper就行了:

```
HandlerThread handlerThread = new HandlerThread("HandlerThread");
handlerThread.start();

mHandler = new Handler(handlerThread.getLooper()) {
	@Override
	public void handleMessage(Message msg) {
		super.handleMessage(msg);
	}
};
```

# IntentService

我们知道Service的各个声明周期函数也是在主线程中执行的,它也不能直接执行耗时操作。需要将耗时操作放到子线程中进行。

为了方便在Service中进行耗时操作,谷歌提供了Service的子类IntentService。它有和Service相同的生命周期,同时也提供了在子线程处理耗时操作的机制。

IntentService是一个抽象类,使用的时候需要继承并实现它的onHandleIntent方法,这个方法是在子线程中执行的,可以直接在这里进行耗时操作。

其实IntentService内部也是通过HandlerThread实现的,而且代码十分简单:

```
public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper;
    private volatile ServiceHandler mServiceHandler;
    private String mName;
    private boolean mRedelivery;

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }

    public IntentService(String name) {
        super();
        mName = name;
    }

    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }

    @Override
    @Nullable
    public IBinder onBind(Intent intent) {
        return null;
    }

    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);
}

```