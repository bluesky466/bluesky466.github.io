title: Java多线程 - Future模式
date: 2017-09-09 15:51:43
tags:
	- 技术相关
	- java
---

# 什么是Future模式

Future模式是多线程开发中非常常见的一种设计模式。它的核心思想是异步调用。当我们需要调用一个函数方法时。如果这个函数执行很慢,那么我们就要进行等待。但有时候,我们可能并不急着要结果。因此,我们可以让被调用者立即返回,让他在后台慢慢处理这个请求。对于调用者来说,则可以先处理一些其他任务,在真正需要数据的场合再去尝试获取需要的数据。

用生活中的例子来打个比喻,就像叫外卖。比如在午休之前我们可以提前叫外卖,只需要点好食物,下个单。然后我们可以继续工作。到了中午下班的时候外卖也就到了,然后就可以吃个午餐,再美滋滋的睡个午觉。而如果你在下班的时候才叫外卖,那就只能坐在那里干等着外卖小哥,最后拿到外卖吃完午饭,午休时间也差不多结束了。

使用Future模式,获取数据的时候无法立即得到需要的数据。而是先拿到一个契约,你可以再将来需要的时候再用这个契约去获取需要的数据,这个契约就好比叫外卖的例子里的外卖订单。

# 用普通方式和Future模式的差别

我们可以看一下使用普通模式和用Future模式的时序图。可以看出来普通模式是串行的,在遇到耗时操作的时候只能等待。而Future模式,只是发起了耗时操作,函数立马就返回了,并不会阻塞客户端线程。所以在工作线程执行耗时操作的时候客户端无需等待,可以继续做其他事情,等到需要的时候再向工作线程获取结果:

{% img /Java多线程-Future模式/1.png %}

# Future模式的简单实现

首先是FutureData,它是只是一个包装类,创建它不需要耗时。在工作线程准备好数据之后可以使用setData方法将数据传入。而客户端线程只需要在需要的时候调用getData方法即可,如果这个时候数据还没有准备好,那么getData方法就会等待,如果已经准备好了就好直接返回。

```
public class FutureData<T> {
    private boolean mIsReady = false;
    private T mData;

    public synchronized void setData(T data) {
        mIsReady = true;
        mData = data;

        notifyAll();
    }

    public synchronized T getData() {
        while (!mIsReady) {
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        return mData;
    }
}
```

接着是服务端,客户端在向服务端请求数据的时候服务端不会实际去加载数据,它只是创建一个FutureData,然后创建子线程去加载,而它只需要直接返回FutureData就可以了。

```
public class Server {
    public FutureData<String> getString() {
        final FutureData<String> data = new FutureData<>();
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                data.setData("world");
            }
        }).start();

        return data;
    }
}
```

客户端代码如下,整个程序只需要运行2秒多,但如果不使用Future模式的话就需要三秒了。

```
Server server = new Server();
FutureData<String> futureData = server.getString();

//先执行其他操作
String hello = "hello";
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    e.printStackTrace();
}

System.out.print(hello + " " + futureData.getData());
```

# JDK中的Future模式

还记得我之前的一篇文章《Java多线程 - 线程池》中写的ExecutorService.execute()和ExecutorService.submit()的区别吗(如果没有看过的读者可以去看一下)？

execute方法其实是在Executor中定义的,而ExecutorService继承了Executor。它只是简单的提交了一个Runnable给线程池中的线程去调用:

```
public interface Executor {
    void execute(Runnable command);
}

public interface ExecutorService extends Executor {
	...
}
```

而submit方法是ExecutorService中定义的,它们都会返回一个Future对象。实际上submit方法就是使用的Future模式:

```
public interface ExecutorService extends Executor {
	...
	<T> Future<T> submit(Callable<T> task);
		
	<T> Future<T> submit(Runnable task, T result);
		
	Future<?> submit(Runnable task);
	...
}
```

__Future\<?\> submit(Runnable task) :__

它的返回值实际上是Future\<Void\>,子线程是不会返回数据的。

__\<T\> Future\<T\> submit(Runnable task, T result) :__

这个方法是不是很蛋疼,返回的结果在调用的时候已经给出了。如果我一开始就知道结果那我为什么又要发起子线程呢？

其实不然,这个result可以是一个代理,它不是实际的结果,它只是存储了结果。我这里给出一个例子大家体会一下吧:

```
final String[] result = new String[1];

Runnable r = new Runnable() {
    public void run() {
        result[0] = "hello world";
    }
};

Future<String[]> future = Executors.newSingleThreadExecutor().submit(r, result);
    
try {
    System.out.println("result[0]: " + future.get()[0]);
} catch (InterruptedException e) {
    e.printStackTrace();
} catch (ExecutionException e) {
    e.printStackTrace();
}
```

__\<T\> Future\<T\> submit(Callable\<T\> task) :__

这个方法就比较好理解了, Callable.call()方法在子线程中被调用,同时它有返回值,只有将加载的数据直接return出来就好:

```
Future<String> future = Executors.newSingleThreadExecutor()
        .submit(new Callable<String>() {
            @Override
            public String call() throws Exception {
                return "Hello World";
            }
        });

try {
    System.out.print(future.get());
} catch (InterruptedException e) {
    e.printStackTrace();
} catch (ExecutionException e) {
    e.printStackTrace();
}
```

# 一个实际的例子

比如我们在计算两个List\<Integer\>中的数的总和的时候就可以用Future模式提高效率:

```
public int getTotal(final List<Integer> a, final List<Integer> b) throws ExecutionException, InterruptedException {
    Future<Integer> future = Executors.newCachedThreadPool().submit(new Callable<Integer>() {
        @Override
        public Integer call() throws Exception {
            int r = 0;
            for (int num : a) {
                r += num;
            }
            return r;
        }
    });

    int r = 0;
    for (int num : b) {
        r += num;
    }
    return r + future.get();
}
```
