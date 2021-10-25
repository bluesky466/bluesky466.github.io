title: Flutter线程模型
date: 2021-10-24 11:32:52
tags:
    - 技术相关
    - Flutter
---

事件循环在UI框架里面应该算是一个常见的东西，例如安卓主线程里面就有个Looper一直在MessageQueue里面读取事件。Flutter里面也有类似的东西。

实际上Flutter的事件循环应该是Dart语言层面就支持的东西。Dart是单线程模型的编程语言，它的一个线程对应一个Isolate，而一个Isolate就会带有一个事件循环。值得注意的是虽然你可以启动多个Isolate来实现多线程，但是正如它的名字"隔离"，Isolate之间是内存隔离的，它们有独立的堆内存，这意味着两个Isolate不会出现线程安全问题:


{% img /Flutter线程模型/1.png %}

Isolate启动之后就会默认开启事件循环，并不需要我们干些什么。这里有个比较有意思的事情是，不像其他语言，main函数执行完成之后整个程序就结束并退出了。Dart的main函数执行完之后会进入事件循环，监听事件并且进行回调，这些事件包括按键事件、Timer事件等:


{% img /Flutter线程模型/2.png %}

所以从这个角度看，Dart的main函数更像是一个init函数，它意味着Dart程序的启动，但它退出后Dart程序还会进行运行。例如下面的代码:

```dart
void main() {
  print("main begin");
  Timer(Duration(seconds: 3), () {
    print("on timer");
  });
  print("main begin finish");
}
```

通过打印我们可以看到，就算main函数退出了，等3秒后Timer时间到达依然会打印"on timer":

```
I/flutter (15551): main begin
I/flutter (15551): main begin finish
I/flutter (15551): on timer
```

# 消息队列

消息循环一般和消息队列配套使用，就例如安卓里面的Looper和MessageQueue。而Dart里面有两条消息队列microtask队列和event队列，时间循环会执行microtask队列内的任务，microtask队列为空之后再去执行event队列里面的任务:

{% img /Flutter线程模型/3.png %}

event队列包含Dart和来自系统其它位置的事件。但microtask队列只包含来自当前isolate的内部代码。如果我们希望在这次事件循环处理完成之后，下一次事件循环处理之前做一些事情，就可以往microtask队列里面加入任务:

```Dart
void main() {
  new Future(() {
    scheduleMicrotask(()=>print("in microtask queue 1"));
    print("in event queue 1");
  });
  new Future(() {
    scheduleMicrotask(()=>print("in microtask queue 2"));
    print("in event queue 2");
  });
  new Future(() {
    scheduleMicrotask(()=>print("in microtask queue 3"));
    print("in event queue 3");
  });
}
```

在main函数里面我们使用Future往event队列插入了3个task，但是执行到每个task的时候，又会往microtask队列插入microtask，这就造成task执行完之后重新遍历microtask队列发现microtask去执行:

```
in event queue 1
in microtask queue 1
in event queue 2
in microtask queue 2
in event queue 3
in microtask queue 3
```

Future有个then方法，可以在Future执行完之后执行指定操作:

```Dart
void main() {
  new Future(() {
    scheduleMicrotask(()=>print("in microtask queue 1"));
    print("in event queue 1");
    return "a";
  }).then((a) {
    scheduleMicrotask(()=>print("in microtask queue 2"));
    print("in event queue 2 -> $a");
    return new Future(() { return "b";});
  }).then((b) {
    scheduleMicrotask(()=>print("in microtask queue 3"));
    print("in event queue 3 -> $b");
  });
}
```

一开始我理解错误，以为是在这个Future task后面插入另一个Future，其实实际上then里面的是callback，它的参数是上一个callback的返回值。callback会在Future执行完之后立即执行，而不会插入event队列。但如果上一个callback的返回值是Future的话，就会往event队列插入任务，而后面的callback实际上监听的是这个返回的Future。所以上面的代码打印如下:

````
I/flutter (23893): in event queue 1
I/flutter (23893): in event queue 2 -> a
I/flutter (23893): in microtask queue 1
I/flutter (23893): in microtask queue 2
I/flutter (23893): in event queue 3 -> b
I/flutter (23893): in microtask queue 3
````

# 单线程模型

## 线程阻塞

另外正如一开始说的Dart是单线程模型，由于这个事件循环是在单个线程内的，所以如果我们的task耗时比较长就会阻塞后面的task:

```Dart
void main() {
  print('start time :' + DateTime.now().toString()); // 当前时间
  Timer(Duration(seconds: 1), () {
    //callback function
    print('first timer :' + DateTime.now().toString()); // 1s之后
    sleep(Duration(seconds: 3));
  });
  Timer(Duration(seconds: 2), () {
    //callback function
    print('second timer :' + DateTime.now().toString()); // 2s之后
  });
}
```

所以我们的第二个Timer虽然设定是在2秒后执行，但是实际上它会被第一个timer的3秒sleep阻塞住,等到第一个Timer结束才执行:

```
I/flutter (21196): start time :2021-10-20 21:30:11.794680
I/flutter (21196): first timer :2021-10-20 21:30:12.835643
I/flutter (21196): second timer :2021-10-20 21:30:15.841398
```

因此我们不能过分的信任这些定时任务。

## 未捕获的异常

不像java、kotlin这些语言多线程语言，如果一个线程中出现了未捕获的异常，那么这个线程就被强制结束了。由于采用事件循环的机制来运行相对独立的task，Dart不要求我们必须处理异常。当一个task出现了异常，虽然会结束这个task，但是并不影响整个线程，后续的其他task仍可以继续执行:

```dart
void main() {
  new Future(() {
    print("task1 begin");
    throw new Exception();
    print("task1 finish");
  });
  new Future(() {
    print("task2 begin");
    print("task2 finish");
  });
}
```

task1被中断了，但是task2依然会执行:

```
task1 begin
Error: Exception
    at Object.throw_ [as throw] (http://localhost:64924/dart_sdk.js:5041:11)
    at http://localhost:64924/packages/myflutter/main.dart.lib.js:365:17
    at http://localhost:64924/dart_sdk.js:32040:31
    at internalCallback (http://localhost:64924/dart_sdk.js:24253:11)
task2 begin
task2 finish
```

# 在Dart里面实现多线程

虽然通过Dart的async await可以实现类似kotlin的协程的功能，但是如果不做特殊操作，这些协程实际上是跑在同一个线程中的。一旦某个协程做了些耗时操作如复杂计算等，就会造成ui卡顿。这个时候我们可以创建多个Isolate去实现类似多线程的操作:

```Dart
int globalData = 1;

void otherIsolate(SendPort sendPort) {
  while(true){
    globalData ++;
    sleep(Duration(seconds: 1));
    sendPort.send("globalData from otherIsolate $globalData");
  }
}

void main() {
  globalData = 100;

  ReceivePort receivePort = ReceivePort();
  receivePort.listen((message) {
    print(message);
  });

  Isolate.spawn(otherIsolate, receivePort.sendPort);
  Future.delayed(Duration(seconds: 3), ()=>{
    print("globalData in Future : $globalData")
  });
}
```

我们可以通过ReceivePort这种类似管道的东西来进行Isolate之间的通信。正如之前所说Isolate是内存隔离的，它更像一个进程的概念。所以并不能像其他语言的多线程一样通过全局变量来交换数据:

```
I/flutter (17503): globalData from otherIsolate 2
I/flutter (17503): globalData from otherIsolate 3
I/flutter (17503): globalData in Future : 100
I/flutter (17503): globalData from otherIsolate 4
I/flutter (17503): globalData from otherIsolate 5
I/flutter (17503): globalData from otherIsolate 6
```


# 参考

学习这部分知识的时候参考了下面的文章，有兴趣的同学可以看看:

[给 Android 开发者的 Flutter 指南](https://flutter.cn/docs/get-started/flutter-for/android-devs#what-is-the-equivalent-of-runonuithread-in-flutter)

[Dart asynchronous programming: Isolates and event loops](https://medium.com/dartlang/dart-asynchronous-programming-isolates-and-event-loops-bffc3e296a6a)

[Dart与消息循环机制[翻译]](https://www.jianshu.com/p/7549b63a72d7)

