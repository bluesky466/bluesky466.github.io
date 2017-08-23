title: Java多线程 - 如果正确的终止线程
date: 2017-08-24 00:50:09
tags:
	- 技术相关
	- java
---

最近打算读一下《实战java高并发程序设计》，夯实一下java多线程的知识。接下来应该会写一系列的读书笔记,里面会有多处引用到书中的代码或者文字。本文就是第一篇。


# 不推荐使用的stop方法

Thread.stop()是一个被废弃的方法,不被推荐使用的原因是stop方法太过于暴力，强行把执行到一半的线程终止,并且会立即释放这个线程所有的锁。会破坏了线程中引用对象的一致性。

例如在数据库中维护着一张用户表，记录了用户ID和用户名，使用Thread多线程写入两条记录:

```
记录1: ID=1,NAME=小明  
记录2: ID=2,NAME=小王
```

如果在记录1写到一半的时候被stop结束了，就可能出现各种奇怪的现象：

1. 记录1被记录2覆盖，没有任何数据留下。
2. 记录1只有一半，即只有ID，而NAME为空。
3. 记录1和记录2混在同一条记录中，最后写入了一条一半是记录1一半是记录2的脏数据

所以,除非你很确定你在做什么,否则不要轻易使用stop方法

# 使用判断标志位的方法中断线程

那如果的确有中断线程的需求,我们需要怎么做呢？一般我们马上就会想到设置标志位的方法,即在线程中执行每一个步骤之前都判断一下是否需要退出线程:

```
class WorkThread extends Thread {
    private boolean mExitThread = false;

    public void exitThread() {
        mExitThread = true;
    }

    @Override
    public void run() {
        if (mExitThread) {
            return;
        }
        //do something

        if (mExitThread) {
            return;
        }
        //do something

        if (mExitThread) {
            return;
        }
        //do something
    }
}
```

其实Thread类早就帮我们实现了这个中断标志了。与Thread中断相关的方法有下面三个:


> public void Thread.interrupt() //线程中断  
> public native boolean Thread.isInterrupted() //判断是否被中断  
> public static native boolean Thread.interrupted() //判断是否中断,并清除当前中断状态


所以上面的代码可以改写成这样:

```
class WorkThread extends Thread {
    @Override
    public void run() {
        if (isInterrupted()) {
            return;
        }
        //do something

        if (isInterrupted()) {
            return;
        }
        //do something

        if (isInterrupted()) {
            return;
        }
        //do something
    }
}
```

这个时候我们只需要调用Thread.interrupt()方法就能安全的中断线程了。

需要提醒一下的是Thread.interrupt()方法并不会像Thread.stop()方法一样立即结束线程,它只是设置了一个中断标志,需要在代码实现中去手动判断这个标志并且推出。

像下面这个代码就算掉了Thread.interrupt()方法也不会中断线程:

```
class WorkThread extends Thread {
    @Override
    public void run() {
        while(true){
        	//do something
        }
    }
}
```

# Thread.interrupt的优点

使用Thread.interrupt去中断线程除了可以免去自己实现标志位的烦恼之外,还可以中断sleep和wait中的线程。

还记得我们在调用Thread.sleep()这个方法的时候都需要catch或者抛出InterruptedException这个异常吗:

```
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

这个InterruptedException异常就是由于我们调用了Thread.interrupt方法抛出的。所以Thread.interrupt可以打断Thread.sleep。同样Thread.wait也是可以被Thread.interrupt打断的。

需要注意的是如果sleep方法由于中断而抛出异常，此时，它会__清除中断标记__。所以在catch到这个异常的时候__需要再次设置中断标记__：

```
Thread t1 = new Thread() {
    @Override
    public void run(){
        while(true){
            if(Thread.currentThread().isInterrupted()){
                break;
            }

            System.out.println("hello world");
            try{
                Thread.sleep(1000);
            }catch(InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
};
```
