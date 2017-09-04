title: Java多线程 - 线程池
date: 2017-09-05 00:01:48
tags:
---

这篇文章大部分都是直接摘抄自《实战Java高并发程序设计》,基本上就是一篇笔记,用于以后忘了的时候可以回顾。

# 框架提供的ExecutorService

Executors框架提供了各种类型的线程池,主要有以下工厂方法:

```
public static ExecutorService newFixedThreadPool(int nThreads) 
public static ExecutorService newSingleThreadExecutor()
public static ExecutorService newCachedThreadPool()
public static ScheduledExecutorService newSingleThreadScheduledExecutor()
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize)
```

- newFixedThreadPool()方法. 该方法返回一个固定线程数量的线程池,该线程池中的线程数量始终不变,当有一个新任务时,线程池中若有空闲线程,则立即执行,若没有,则新任务会被暂时存在一个队列中,得有空闲线程时,便处理在任务队列中的任务

- newSingleThreadExecutor()方法,改方法返回一个只有一个线程的线程池,若多余一个任务被提交到该线程池,任务会被保存在一个队伍队列,带线程空闲,按先入先出的顺序执行队列中的任务,

- newCachedThreadPool()方法,该方法返回一个可根据实际情况调整线程数量的线程池.线程池数量是不确定的,但若有空闲线程可以复用,则会优先使用可以复用的线程,若所有线程均在工作,又有新的任务提交,则会创建新的线程处理任务,所有线程在当前任务执行完毕后,将返回线程池进行复用,

- newSingleThreadScheduledExecutor()方法: 改方法返回一个ScheduledExecutorService对象,线程池大小为1 这个接口在ExecutorService接口之上拓展了在给定时间执行某任务的功能,如在某个固定的延时之后执行,或者周期性执行某个任务.

- newScheduledThreadPool()方法:改方法也返回一个ScheduledExecutorService对象 但改线程池可以指定线程数量

前面三个工厂方法创建的ExecutorService只需要使用ExecutorService.execute()方法或者submit()方法将需要执行的任务传入即可,这里就不细讲了。关于这两个方法的差异我会在后面细说,这里也不展开讨论了。

后面两个工厂方法会创建ScheduledExecutorService。它有会多出下面三个schedule方法用于延迟执行任务:

```
public ScheduledFuture<?> schedule(Runnable command,
                                   long delay, TimeUnit unit);
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                              long initialDelay,
                                              long period,
                                              TimeUnit unit);
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                 long initialDelay,
                                                 long delay,
                                                 TimeUnit unit);
```

schedule()方法会在给定时间,对方法进行一次调度。scheduleAtFixedRate()方法和scheduleWithFixedDelay()会对任务进行周期性调度。但两者有一点小小的差别:

{% img /Java多线程-线程池/1.png %}

对于FixedRate方式来说,任务调度的频率是一样的。它是以上一个任务开始执行时间为起点,之后的period时间,调度下一次任务。而FixDelay则是在上一个任务结束后,再经过delay时间进行任务调度。

# ThreadPoolExecutor

对于Executors.newFixedThreadPool()、Executors.newSingleThreadExecutor()、Executors.newCachedThreadPool()这几个方法虽然创建的线程池的功能特点完全不一样,但是他们其实都是使用了ThreadPoolExecutor实现:

```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
    
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}

public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

ThreadPoolExecutor的最重要的构造函数如下:

```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)  
```

- corePoolSize: 指定了线程池中的线程数量
- maximumPoolSize: 指定了线程池中的最大线程数量
- keepAliveTime: 当线程池线程数量超过corePoolSize时,多余的空闲线程的存活时间。即,超过corePoolSize的空闲线程,在多长的时间内,会被销毁。
- unit: keepAliveTime的时间单位
- workQueue: 被提交但未被执行的任务
- threadFactory: 线程工厂,用于创建线程,一般用默认即可
- handler: 拒绝策略。但任务太多来不及处理,如何拒绝任务

以上参数中,大部分都很简单,只有workQueue和handler需要说一下。

内置的BlockingQueue有下面几种:

- SynchronousQueue: 一个没有容量的队列。使用SynchronousQueue,提交的任务不会真正的被保存,而总是将新任务提交给线程执行。如果没有空闲线程,就创建新线程,如果线程数量已经到达最大值,则执行拒绝策略

- ArrayBlockingQueue: 有界任务队列,若有新的任务需要执行,如果实际线程数少于corePoolSize则创建新的线程,如果大于corePoolSize,就会放入ArrayBlockingQueue中,如果ArrayBlockingQueue已满,在总线程数不大于maximumPoolSize的情况下会创建新线程,否则就执行拒绝策略

- LinkedBlockingQueue: 无界任务队列,若有新的任务需要执行,如果实际线程数少于corePoolSize则创建新的线程,如果大于corePoolSize,就会放入LinkedBlockingQueue中等待

- PriorityBlockingQueue: 它是一个特殊的无界队列,可以设定任务的优先级

而内置的拒绝策略又有下面几种:

- AbortPolicy策略: 该策略会直接抛出异常,阻止系统正常工作
- CallerRunsPolicy策略: 只要线程池没有关闭,该策略直接在调用者线程中运行被拒绝的任务。(使用这个策略可能导致在主线程执行耗时操作)
- DiscardOldestPolicy策略: 该策略丢弃一个最老的任务,并尝试重新提交任务
- DiscardPolicy策略: 该策略默默丢弃拒绝的任务,不做任何处理。


线程池任务调度的逻辑如下图所示:


{% img /Java多线程-线程池/2.png %}

# execute和submit的区别

ExecutorService.execute()和ExecutorService.submit()都可以提交任务去异步执行,但是它们之间有什么区别呢？

```
void execute(Runnable command);
Future<?> submit(Runnable task);
<T> Future<T> submit(Callable<T> task);
```

- 返回值

ExecutorService.execute()没有返回值,只能简单的提交Runnable给线程池去运行

ExecutorService.submit(),有返回值,可以获得一个Future

- 异常

ExecutorService.execute()的异常机制和普通线程的异常机制一样,必须用try、catch来捕获异常。如果没有捕获一些运行时异常,也会打印出堆栈信息:

```
Executors.newCachedThreadPool().execute(
        new Runnable() {
            @Override
            public void run() {
                int i = 1 / 0;
            }
        }
);
```
```
Exception in thread "pool-1-thread-1" java.lang.ArithmeticException: / by zero
```

ExecutorService.submit()的异常会被吃掉,下面的代码的异常会被默默吃掉,没有堆栈信息的打印:

```
Executors.newCachedThreadPool().submit(
        new Runnable() {
            @Override
            public void run() {
                int i = 1 / 0;
            }
        }
);
```
但是我们可以调用Future.get()方法,这样当抛出异常的时候系统也会打印堆栈:

```
Future future = Executors.newCachedThreadPool().submit(
        new Runnable() {
            @Override
            public void run() {
                int i = 1 / 0;
            }
        }
);
future.get();
```

需要注意的是Future.get()是阻塞的,需要需要等待线程执行完毕才会返回,所以我们可以用这个方法获得Callable.call()的返回值:

```
Future<Integer> future = Executors.newCachedThreadPool().submit(
        new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                return 123;
            }
        }
);
System.out.println(future.get());
```