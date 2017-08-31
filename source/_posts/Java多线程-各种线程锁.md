title: Java多线程 - 各种线程锁
date: 2017-08-31 00:07:53
tags:
	- 技术相关
	- java
---

多个线程同时对同一个对象进行读写操作,很容易会出现一些难以预料的问题。所以很多时候我们需要给代码块加锁,同一时刻只允许一个线程对某个对象进行操作。多线程之所以会容易引发一些难以发现的bug,很多时候是写代码的程序员对线程锁不熟悉或者干脆就没有在必要的地方给线程加锁导致的。这里我想总结一下java多线程中的各种锁的作用和用法,还有容易踩的坑。

这篇文章里面有很多的文字和代码都来自于《实战Java高并发程序设计》。它真的是一本很不错的书,建议大家有空可以去看一下。

# synchronized关键字

## synchronized的作用

关键字synchronized的作用是实现线程间的同步。它的工作是对同步的代码加锁,使得每一次,只能有一个线程进入同步块,从而保证线程间的安全性。

关键字synchronized可以有多张用法,这里做一个简单的整理:

> 指定加锁对象:对给定对象加锁,进入同步代码前要获取给定对象的锁。  
> 直接作用于实例方法:相当于给当前实例加锁,进入同步代码块前要获得当前实例的锁。  
> 直接作用于静态方法:相当于对当前类加锁,进入同步代码前要获取当前类的锁。

下面来分别说一下上面的三点:

假设我们有下面这样一个Runnable,在run方法里对__静态__成员变量sCount自增10000次:

```
class Count implements Runnable {
    private static int sCount = 0;

    public static int getCount() {
        return sCount;
    }

    @Override
    public void run() {
        for (int i = 0; i < 10000; i++) {
            sCount++;
        }
    }
}
```

假设我们在两个Thread里面同时跑这个Runnable:

```
Count count = new Count();
Thread t1 = new Thread(count);
Thread t2 = new Thread(count);
t1.start();
t2.start();
try {
    t1.join();
    t2.join();
} catch (InterruptedException e) {
    e.printStackTrace();
}
System.out.print(Count.getCount());
```

得到的结果并不是20000,而是一个比20000小的数,如14233。

这是为什么呢？假设两个线程分别读取sCount为0,然后各自技术得到sCount为1,并先后写入这个结果,因此,虽然sCount++执行了2次,但是实际sCount的值只增加了1。

我们可以用指定加锁对象的方法解决这个问题,这里因为两个Thread跑的是同一个Count实例,所以可以直接给this加锁:

```
class Count implements Runnable {
    private static int sCount = 0;

    public static int getCount() {
        return sCount;
    }

    @Override
    public void run() {
        for (int i = 0; i < 10000; i++) {
            synchronized (this) {
                sCount++;
            }
        }
    }
}
```

我们也可以给实例方法加锁,这种方式和上面那一种的区别就是给this加锁,锁的区域比较小,两个线程交替执行sCount++操作,而给方法加锁的话,先拿到锁的线程会连续执行1000次sCount自增,然后再释放锁给另一个线程。

```
class Count implements Runnable {
    private static int sCount = 0;

    public static int getCount() {
        return sCount;
    }

    @Override
    public synchronized void run() {
        for (int i = 0; i < 10000; i++) {
            sCount++;
        }
    }
}
```

synchronized直接作用于静态方法的用法和上面的给实例方法加锁类似,不过它是作用于静态方法:

```
class Count implements Runnable {
    private static int sCount = 0;

    public static int getCount() {
        return sCount;
    }

    @Override
    public void run() {
        for (int i = 0; i < 10000; i++) {
            increase();
        }
    }

    private static synchronized void increase() {
        sCount++;
    }
}
```

## 等待(wait)和通知(notify)

Object有两个很重要的接口:Object.wait()和Object.notify()

当在一个对象实例上调用了wait()方法后,当前线程就会在这个对象上等待。直到其他线程调用了这个对象的notify()方法或者notifyAll()方法。notifyAll()方法与notify()方法的区别是它会唤醒所有正在等待这个对象的线程,而notify()方法只会随机唤醒一个等待该对象的线程。

wait()、notify()和notifyAll()都需要在synchronized语句中使用:

```
class MyThread extends Thread {
    private Object mLock;

    public MyThread(Object lock) {
        this.mLock = lock;
    }

    @Override
    public void run() {
        super.run();

        synchronized (mLock) {
            try {
                mLock.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println("in MyThread");
        }
    }
}
```

```
Object lock = new Object();
MyThread t = new MyThread(lock);
t.start();

System.out.println("before sleep");

try {
    Thread.sleep(2000);
} catch (InterruptedException e) {
    e.printStackTrace();
}

System.out.println("after sleep");

synchronized (lock) {
    lock.notify();
}

try {
    t.join();
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

从上面的例子可以看出来,在调用wait()方法实际上已经释放了对象的锁,所以在其他线程中才能获取到这个对象的锁,从而进行notify操作。而等待的线程被唤醒后又需要重新获得对象的锁。

## synchronized容易犯的隐蔽错误

### 是否给同一个对象加锁

在用synchronized给对象加锁的时候需要注意加锁是不是同一个,如将代码改成这样:

```
Thread t1 = new Thread(new Count());
Thread t2 = new Thread(new Count());
t1.start();
t2.start();
try {
    t1.join();
    t2.join();
} catch (InterruptedException e) {
    e.printStackTrace();
}
System.out.print(Count.getCount());
```

因为两个线程跑的是不同的Count实例,所以用给指定对象加锁和给实例方法加锁的方法都不能避免两个线程同时对__静态__成员变量sCount进行自增操作。

但是如果用第三种作用于静态方法的写法,就能正确的加锁。

### 是否给错误的对象加锁

如我们将sCount的类型改成Integer,并且在sCount++的时候直接对sCount加锁会发生什么事情呢(毕竟我们会很自然的给要操作的对象加锁来实现线程同步)？

```
class Count implements Runnable {
    private static Integer sCount = 0;

    public static int getCount() {
        return sCount;
    }

    @Override
    public void run() {
        for (int i = 0; i < 10000; i++) {
            synchronized (sCount) {
                sCount++;
            }
        }
    }
}
```

```
Count count = new Count();
Thread t1 = new Thread(count);
Thread t2 = new Thread(count);
t1.start();
t2.start();
try {
    t1.join();
    t2.join();
} catch (InterruptedException e) {
    e.printStackTrace();
}
System.out.print(Count.getCount());
```

最后的得到的结果仍然是比20000小的值。

这是为什么呢？《实战Java高并发程序设计》中给出的解释是这样的:

> 在Java中,Integer使用不变对象。也就是对象一旦被创建,就不可能被修改。也就是说,如果你有一个Integer代表1,那么它就永远是1,你不可能改变Integer的值,使它位。那如果你需要2怎么办呢？也很简单,新建一个Integer,并让它表示2即可。

也就是说sCount在真实执行时变成了:

> sCount = Integer.valueOf(sCount.intValue()+1);

进一步看Integer.valueOf()，我们可以看到:

```
public static Integer valueOf(int i) {
    assert IntegerCache.high >= 127;
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

所以在多个线程中,由于sCount一直在变,并不是同一个对象,所以两个线程的加锁可能加在了不同的Integer对象上,并没有真正的锁住代码块。

我再举一个例子:

```
public void increase(Integer integer){
    integer++;
}
```

在外面这样调用它,并不会使得传入的Integer增加:

```
Integer i = 0;
increase(i);
```

# 重入锁

ReentrantLock的意思是Re-Entrant-Lock也就是重入锁,它的特点就是在同一个线程中可以重复加锁,只需要解锁同样的次数就能真正解锁:

```
class MyThread extends Thread {
    private ReentrantLock mLock = new ReentrantLock();

    @Override
    public void run() {
        super.run();

        mLock.lock();
        System.out.println("outside");

        mLock.lock();
        System.out.println("inside");
        mLock.unlock();

        mLock.unlock();
    }
}
```

事实上synchronized也是可重入的,比如下面的代码同样是可以正常退出的:

```
class MyThread extends Thread {
    @Override
    public void run() {
        super.run();
        synchronized (this) {
            System.out.println("outside");
            synchronized (this) {
                System.out.println("inside");
            }
        }
    }
}
```

与synchronized相比,重入锁需要程序员手动调用加锁和解锁,也因为如此,重入锁对逻辑控制的灵活性要远远好于synchronized。

重入锁可以完全替代synchronized关键字。在JDK 5.0的早起版本中,重入锁的性能远远好于synchronized。但从JDK 6.0开始,JDK在synchronized做了大量优化,使得两者的性能差距并不大。

## ReentrantLock是可中断的

对于synchronized,如果它在等待锁,那么它就只有两个状态:获得锁继续执行或者保持等待。但是对于重入锁,就有了另外一种可能,那就是重入锁在等待的时候可以被中断:

```
class MyThread extends Thread {
    private ReentrantLock mLock = new ReentrantLock();

    @Override
    public void run() {
        super.run();

        try {
            mLock.lockInterruptibly();
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            if(mLock.isHeldByCurrentThread()){
                mLock.unlock();
            }
        }
    }
}
```

## ReentrantLock可以设置等待限时

ReentrantLock.tryLock()方法可以给等待锁设置最长等待时间,如果在设置的时间结束之前获取到锁就会返回true,否则返回false:

```
class MyThread extends Thread {
    private ReentrantLock mLock = new ReentrantLock();

    @Override
    public void run() {
        super.run();

        try {
            if (mLock.tryLock(2, TimeUnit.SECONDS)) {
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            if (mLock.isHeldByCurrentThread()) {
                mLock.unlock();
            }
        }
    }
}
```

ReentrantLock.tryLock()也可以不带参数直接运行。在这种情况下,当前线程会尝试获得锁,如果锁并未被其他线程占用,则申请锁会成功,并立即返回true。如果锁被其他线程占用,则当前线程不会进行等待,而是立即返回false。

## ReentrantLock可以设置公平锁

大多数情况下,锁的申请是非公平的。也就是说,线程1首先请求了锁A，接着线程2也请求了锁A。那么当锁A可用时,是线程1可以获得锁还是线程2可以获得锁呢？这是不一定的。系统只是会从这个锁的等待队列里随机挑选一个:

```
class MyThread extends Thread {
    private ReentrantLock mLock;

    public MyThread(String name, ReentrantLock lock) {
        super(name);
        this.mLock = lock;
    }

    @Override
    public void run() {
        super.run();

        while (true) {
            mLock.lock();
            System.out.println(Thread.currentThread().getName() + "获得锁");
            mLock.unlock();
        }
    }
}
```

```
ReentrantLock lock = new ReentrantLock();
MyThread t1 = new MyThread("t1", lock);
t1.start();

MyThread t2 = new MyThread("t2", lock);
t2.start();

try {
    t1.join();
    t2.join();
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

打印如下:

```
t1获得锁
t2获得锁
t2获得锁
t2获得锁
t1获得锁
t2获得锁
t2获得锁
t1获得锁
t2获得锁
t1获得锁
t2获得锁
t2获得锁
t2获得锁
t1获得锁
t1获得锁
```

synchronized产生的锁也是非公平的。但如果使用ReentrantLock(boolean fair)构造函数创建ReentrantLock,并且传入true。则该重入锁是公平的:

```
ReentrantLock lock = new ReentrantLock(true);
MyThread t1 = new MyThread("t1", lock);
t1.start();

MyThread t2 = new MyThread("t2", lock);
t2.start();

try {
    t1.join();
    t2.join();
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

打印如下:

```
t1获得锁
t2获得锁
t1获得锁
t2获得锁
t1获得锁
t2获得锁
t1获得锁
t2获得锁
t1获得锁
t2获得锁
t1获得锁
```

需要注意的是实现公平锁必然要求系统维护一个有序队列,所以公平锁的实现成本较高,性能也相对低下,因此,默认情况下,锁是非公平的。

## ReentrantLock可以与Condition配合使用

Condition和之前讲过的Object.wait()还有Object.notify()的作用大致相同:

```
class MyThread extends Thread {
	private ReentrantLock mLock;
	private Condition mCondition;
	
	public MyThread(ReentrantLock lock, Condition condition) {
	    this.mLock = lock;
	    this.mCondition = condition;
	}
	
	@Override
	public void run() {
	    super.run();
	
	    mLock.lock();
	    try {
	        mCondition.await();
	    } catch (InterruptedException e) {
	        e.printStackTrace();
	    }
	    mLock.unlock();
	
	    System.out.println("in MyThread");
	}
}
```

```
ReentrantLock lock = new ReentrantLock();
Condition condition = lock.newCondition();
MyThread t = new MyThread(lock, condition);
t.start();

System.out.println("before sleep");
try {
    Thread.sleep(2000);
} catch (InterruptedException e) {
    e.printStackTrace();
}
System.out.println("after sleep");

lock.lock();
condition.signal();
lock.unlock();

try {
    t.join();
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

Condition的操作需要在ReentrantLock.lock()和ReentrantLock.unlock()之间进行的。

ReentrantLock.newCondition()可以创建一个Condition。Condition.await()方法相当于Object.wait()方法,而Condition.signal()方法相当于Object.notify()方法。当然它也有对应的Condition.signalAll()方法。

同样的在调用Condition.await()之后,线程占用的锁会被释放。这样在Condition.signal()方法调用的时候才获取到锁。

需要注意的是Condition.signal()方法调用之后,被唤醒的线程因为需要重新获取锁。所以需要等到调用Condition.signal()的线程释放了锁(调用ReentrantLock.unlock())之后才能继续执行。


Condition接口的基本方法如下,它提供了限时等待、不可中断的等待之类的操作:

```
void await() throws InterruptedException;
void awaitUninterruptibly();
long awaitNanos(long nanosTimeout) throws InterruptedException;
boolean await(long time, TimeUnit unit) throws InterruptedException;
boolean awaitUntil(Date deadline) throws InterruptedException;
void signal();
void signalAll();
```

# 信号量

信号量为多线程协作提供了更为强大的控制方法。广义上说,信号量是对锁的拓展。无论是synchronize还是重入锁,一次都只运行一个线程访问一个资源,而信号锁则可以指定多个线程,同时访问某一个资源。

像下面的代码, MyRunnable被加锁的代码块一次会被5个线程执行:

```
public class MyRunnable implements Runnable {
    private Semaphore mSemaphore;

    public MyRunnable(Semaphore semaphore) {
        mSemaphore = semaphore;
    }

    @Override
    public void run() {
        try {
            mSemaphore.acquire();
            Thread.sleep(2000);
            System.out.println("thread " + Thread.currentThread().getId() + " working");
            mSemaphore.release();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

```
 Semaphore semaphore = new Semaphore(5);
for (int i = 0; i < 19; i++) {
    new Thread(new MyRunnable(semaphore)).start();
}

Thread t = new Thread(new MyRunnable(semaphore));
t.start();
t.join();
```

Semaphore.acquire()方法尝试获得一个准入许可。如无法获得,线程就会等待。而Semaphore.release()则在线程访问资源结束后,释放一个许可。

Semaphore有下面的一些常用方法:

```
public Semaphore(int permits)
public Semaphore(int permits, boolean fair)
public void acquire() 
public void acquireUninterruptibly()
public boolean tryAcquire()
public boolean tryAcquire(long timeout, TimeUnit unit)
public void release()
```

# 其他的一些锁

## 读写锁

读写锁(ReadWriteLock)是JDK5中提供的分离锁。读写分离锁可以有效的减少锁竞争。

读写锁允许多个线程同时读,但是写写操作和读写操作就需要相互等待了。读写锁的访问约束如下:

||读|写|
|:-:|:-:|:-:|
|读|非阻塞|阻塞|
|写|阻塞|阻塞|

读写操作在某些特定操作下可以提高程序的性能,如下面的代码。如果使用重入锁,需要十一秒左右才能运行完:

```
public class Data {
    private String mData = "data";
    private ReentrantLock mLock = new ReentrantLock();

    public String readData(){
        mLock.lock();
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("read data : " + mData);
        String data = mData;
        mLock.unlock();
        return data;
    }

    public void writeData(String data){
        mLock.lock();
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        mData = data;
        System.out.println("write data : " + mData);
        mLock.unlock();
    }
}
```

```
final Data data = new Data();
for (int i = 0; i < 10; i++) {
    new Thread(new Runnable() {
        @Override
        public void run() {
            data.readData();
        }
    }).start();
}

Thread write = new Thread(new Runnable() {
    @Override
    public void run() {
       data.writeData("update data");
    }
});
write.start();
```

但是如果将重入锁改成读写锁的话只需要两秒左右就能完成:

```
public class Data {
    private String mData = "data";
    private ReadWriteLock mLock = new ReentrantReadWriteLock();
    private Lock mReadLock = mLock.readLock();
    private Lock mWriteLock = mLock.writeLock();

    public String readData(){
        mReadLock.lock();
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        String data = mData;
        System.out.println("read data : " + mData);
        mReadLock.unlock();
        return data;
    }

    public void writeData(String data){
        mWriteLock.lock();
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("write data : " + mData);
        mData = data;
        mWriteLock.unlock();
    }
}
```

# 倒计时器、循环栅栏


倒计时器(CountDownLatch)和循环栅栏(CyclicBarrier)因为比较不常用,所以这里就不讲了,有兴趣的同学可以自己去看一下《实战Java高并发程序设计》这本书。
