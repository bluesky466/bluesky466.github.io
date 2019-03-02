title: Handler.postDelayed的原理
date: 2019-03-02 16:29:52
tags:
    - 技术相关
    - Android
---

大部分同学在回答Handler的原理的时候都能回答出Handler将消息丢到MessageQueue中,然后Looper.loop死循环不断从MessageQueue中拿消息去执行。

这块我之前也有写个文章介绍,如果忘了可以去[看看](http://blog.islinjw.cn/2017/07/02/Android%E6%B8%A9%E6%95%85%E8%80%8C%E7%9F%A5%E6%96%B0-Handler/)。

但是如果再继续追问Handler.postDelay又是怎么做到的就讲不出来了。这里就给大家讲一讲。

# 源码解析

首先来看看handler里面是怎么处理postDelayed的:

```
public class Handler {
	...

	public final boolean postDelayed(Runnable r, long delayMillis) {
        return sendMessageDelayed(getPostMessage(r), delayMillis);
    }
    ...
    public final boolean sendMessageDelayed(Message msg, long delayMillis) {
        ...
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
    ...
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        ...
        return enqueueMessage(queue, msg, uptimeMillis);
    }
    ...
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        ...
        return queue.enqueueMessage(msg, uptimeMillis);
    }
    ...
}
```

可以发现最后它也是把Runnable封装成Message然后发给MessageQueue去处理的,所以我们继续看看MessageQueue.enqueueMessage方法:

```
boolean enqueueMessage(Message msg, long when) {
    ...
    synchronized (this) {
        ...
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
        	// 插入到队列头
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
        	// 按时间排序插入队列
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false; //如果不是插入队列头的话不需要唤醒线程,让它继续等到拿队列头的消息的时候再重新计算睡眠时间
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        if (needWake) {
        	// 唤醒线程
            nativeWake(mPtr);
        }
    }
    return true;
}
```

这个方法的作用其实很简单,按时间顺序把Message插入MessageQueue,形成一个按时间排序的单链表,然后唤醒线程。

然后看看唤醒了什么线程?

我们都知道MessageQueue中的消息是由Looper.loop里面的一个死循环去读取的。

```
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    ...
    for (;;) {
        Message msg = queue.next(); // might block
        ...
    }
    ...
}
```

这个这里还提示了MessageQueue.next方法也许会阻塞,所以我们看看next方法里面干了什么:

```

Message next() {
    ...

    int nextPollTimeoutMillis = 0;
    for (;;) {
        ...

        //阻塞nextPollTimeoutMillis时间
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // 跳过队列前面的无用Message
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    //如果时间没有到,就计算需要等待的时间
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    //从队列头拿出Message
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    msg.markInUse();
                    return msg;
                }
            } 
            ...
        }
        ...
    }
}

```

这里面有个native方法nativePollOnce,阻塞线程一段固定的时间,当然MessageQueue.enqueueMessage里面的nativeWake方法也能直接唤醒它。当有Message插入队列头的时候,就会唤醒线程。然后MessageQueue.next方法就会拿出队列头的Message计算是否需要再等待一段时间去执行。


# 举个例子

代码比较晕没有关系,我们用一个简单的例子把流程描述一下就好理解了。

首先假设队列里面有两个消息，分别在三秒、四秒之后执行,也就是说MessageQueue.next的线程会睡眠三秒之后才去消息队列拿队列头的消息:

{% img /Handler.postDelayed的原理/1.png %}

此时,我们又post了一个一秒之后执行的Message,于是它会被插入到队列头,然后MessageQueue.next的线程会被唤醒。但是拿到队列头的消息发现时间还没有到,于是又会再睡眠一秒:


{% img /Handler.postDelayed的原理/2.png %}

等了一秒之后MessageQueue.next的线程自己苏醒拿出队列头的MessageC去分发,然后继续拿MessageA。但是发现时间又没有到,于是又会再睡眠两秒:


{% img /Handler.postDelayed的原理/3.png %}

这个时候如果我们插入了一个立马执行的消息呢？它也是会插入到队列头,然后唤醒MessageQueue.next的线程,去队列头取消息执行。执行完之后又会拿MessageA。但是发现时间又没有到,于是又会再睡眠两秒。
