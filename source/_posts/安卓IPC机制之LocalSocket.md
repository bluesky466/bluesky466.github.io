title: 安卓IPC机制之LocalSocket
date: 2021-11-13 17:33:22
tags:
    - 技术相关
    - Android
---
LocalSocket作为安卓提供的一种IPC机制，可能应用层的同学比较陌生，我实际也是在这段时间做项目使用到才注意到它并去了解的。不过实际上framework层里面被频繁使用到了，例如我很久前写的博客[从源码看安卓应用的启动过程](https://blog.islinjw.cn/2018/03/08/%E4%BB%8E%E6%BA%90%E7%A0%81%E7%9C%8B%E5%AE%89%E5%8D%93%E5%BA%94%E7%94%A8%E7%9A%84%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/)里面提到其他进程和Zygote进程之间的通信使用的是LocalSocket。

那么LocalSocket和Socket到底有什么不同呢？[官方文档](https://developer.android.com/reference/android/net/LocalServerSocket)里面其实提到了它其实是基于UNIX-domain socket的:

> Non-standard class for creating an inbound UNIX-domain socket in the Linux abstract namespace.

Socket本来是用来做不同主机间的网络通信的，如果有人想拿来做本机的IPC通信就会发现它的性能堪忧(例如实现binder机制做不到的传输大文件)，因为它需要走网络协议栈、打包拆包、计算校验等，如果是TCP还需要走三次握手和应答。

于是后面就发展出了UNIX-domain socket (LocalSocket)，它的api和socket的基本一致，但是本质上只是一种IPC通信，不可和外部主机通信，但是因为IPC通信是可靠通信，直接将数据拷贝到目标进程内存即可，所以没有之前说的那些耗时的操作。

# 使用方式

我们先来看看它的使用方式:

```java
private void demoClient() throws IOException {
    LocalSocket client = new LocalSocket();
    client.connect(new LocalSocketAddress("me.linjw.localsocket"));
    client.getOutputStream().write(123);
    int read = client.getInputStream().read();
    Log.d(TAG, "response from server : " + read);
    client.close();
}

private void demoServer() throws IOException {
    LocalServerSocket server = new LocalServerSocket("me.linjw.localsocket");
    LocalSocket client = server.accept();
    int read = client.getInputStream().read();
    Log.d(TAG, "request from client :" + read);
    client.getOutputStream().write(read + 1);
    client.getOutputStream().flush();
    client.close();
}

// 打印
// request from client :123
// response from server : 124
```

没错，看起来和普通Socket的用法很类似了。

# 性能

性能是评判一种ipc进制好坏的重要指标，例如我们常用的Binder机制就是用了mmap机制实现了数据的一次拷贝提高了传输速度。

于是我写了一个[测试程序](https://github.com/bluesky466/IPCSpeechTest)来对比AIDL、LocalSocket和TCP Socket的传输速度。测试的逻辑大概是:

1. 每次传输读或者写1024 byte数据
2. 计算3000次读或者写的耗时(也就是计算读3000k或者写3000k数据的总耗时)
3. LocalSocket和TCP Socket每次传输完数据都断开连接,下次需要重新连接

在我们的产品设备上得到的实际数据如下:

| 方式        | 方向 | 第一次3000k | 第二次3000k | 第三次3000k | 第四次3000k | 平均时间 |
| ----------- | ---- | ----------- | ----------- | ----------- | ----------- | -------- |
| AIDL        | 读   | 1.711s      | 1.195s      | 1.25s       | 1.169s      | 1.33125s |
| LocalSocket | 读   | 1.674s      | 1.286s      | 1.185s      | 1.219s      | 1.341s   |
| TCP Socket  | 读   | 10.188s     | 8.926s      | 8.865s      | 8.803s      | 9.1955   |
| AIDL        | 写   | 1.261s      | 1.212s      | 1.175s      | 1.23s       | 1.2195   |
| LocalSocket | 写   | 1.387s      | 1.323s      | 1.23s       | 1.35s       | 1.3225   |
| TCP Socket  | 写   | 8.284s      | 8.242s      | 8.324s      | 8.285s      | 8.28375  |

从上面的数据可以看出来LocalSocket虽然会比AIDL慢但是也差的不多，而tcp的耗时就比较多了。虽然我没有具体看过LocalSocket的底层原理，但是想来既然它在framework层被频繁使用，那么谷歌应该也应该会考虑到性能这一点。

# 优缺点

优点:

1. 可以进行数据流读写,没有大小限制
2. 比TCP Socket会更加安全，因为不能通过抓包监听传输的数据
3. 不会开启线程池(Zygote之所以使用它而不是Binder也是因为Binder机制默认会启动线程池，而fork在多线程下只会fork出当前线程)

缺点:

1. 比Binder的速度还是会稍微慢那么一点点
2. 没有像AIDL这样的高层封装，需要自己实现
3. 和TCP Socket对比起来不能跨主机通信

# 需要注意的地方

1. 虽然不是真正的网络传输，但是也需要声明android.permission.INTERNET权限，要不然同样会报java.net.SocketException: Permission denied异常
2. 虽然可以通过LocalSocket和framework层直接通信，但是如果系统打开了SeLinux就会出现Permission denied异常
3. 在模拟器上LocalSocket的flush用多了耗时有时候会比较严重(Tcp没有问题，实机测试LocalSocket也没有出现问题，猜测和系统相关)