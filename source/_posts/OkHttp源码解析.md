title: OkHttp源码解析
date: 2017-12-30 00:53:32
tags:
    - 技术相关
    - Android
---

# 应用拦截器和网络拦截器

以前其实就有一直在使用okhttp,也有听说过拦截器这东西,但是一直没有去深入了解。最近看《安卓进阶之光》刚好看到okhttp拦截器的内容,然后自己也去挖了下源码,才发现其巧妙之处。

拦截器有两种,应用拦截器和网络拦截器。用法可以看下面的代码:

```
class LogInterceptor implements Interceptor {
    private String mName;

    LogInterceptor(String name) {
        mName = name;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        Response response = chain.proceed(chain.request());
        Log.d("LogInterceptor", "[" + mName + "] : request url = " + response.request().url() + ", " + response.headers().toString());
        return response;
    }
}

OkHttpClient client = new OkHttpClient.Builder()
        .addInterceptor(new LogInterceptor("ApplicationInterceptor"))
        .addNetworkInterceptor(new LogInterceptor("NetworkInterceptor"))
        .build();

Request request = new Request.Builder()
        .url("http://www.github.com")
        .build();

client.newCall(request).enqueue(null);
```

运行之后的打印如下:

```
12-29 00:07:02.378 12641-12859/com.example.okhttp D/LogInterceptor: [NetworkInterceptor] : request url = http://www.github.com/, Content-length: 0
    Location: https://www.github.com/
12-29 00:07:03.653 12641-12859/com.example.okhttp D/LogInterceptor: [NetworkInterceptor] : request url = https://www.github.com/, Content-length: 0
    Location: https://github.com/
12-29 00:07:04.889 12641-12859/com.example.okhttp D/LogInterceptor: [NetworkInterceptor] : request url = https://github.com/, Date: Thu, 28 Dec 2017 16:07:05 GMT
    Content-Type: text/html; charset=utf-8
    Transfer-Encoding: chunked
    Server: GitHub.com
    Status: 200 OK
    ...(省略部分打印)
12-29 00:07:04.896 12641-12859/com.example.okhttp D/LogInterceptor: [ApplicationInterceptor] : request url = https://github.com/, Date: Thu, 28 Dec 2017 16:07:05 GMT
    Content-Type: text/html; charset=utf-8
    Transfer-Encoding: chunked
    Server: GitHub.com
    Status: 200 OK
    ...(省略部分打印)
```

拦截器是一种强大的机制,可以在拦截器中进行监视、重写和重试调用。像我们上面的代码就对请求进行了监视。

从打印可以看到,网络拦截器拦截到了三个请求,同时拦截到了重定向的访问。而应用拦截器只拦截到了一个请求,同时我们可以看到它拦截到的请求的url是 __https://github.com/__ 和我们在代码中的请求 __http://www.github.com__ 并不一致。

简单来讲,网络拦截器在每一次网络访问的时候都会拦截到请求,而应用拦截器只会在OkHttpClient.newCall返回的Call执行的时候被调用一次。

# okhttp的运行流程

在讲拦截器的实现之前我们先来简单介绍一下okhttp的运行流程。

首先通过OkHttpClient.newCall我们可以获得一个RealCall:

```
public class OkHttpClient implements Cloneable, Call.Factory {
  ...
  public Call newCall(Request request) {
    return new RealCall(this, request);
  }
  ...
}
```

## 异步访问

RealCall实现了Call。接口,我们通过调用enqueue方法可以实现异步网络访问。让我们直接看看RealCall.enqueue吧:

```
final class RealCall implements Call {
  ...
  public void enqueue(Callback responseCallback) {
    enqueue(responseCallback, false);
  }

  void enqueue(Callback responseCallback, boolean forWebSocket) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    client.dispatcher().enqueue(new AsyncCall(responseCallback, forWebSocket));
  }
  ...
}
```

client.dispatcher()可以获得一个Dispatcher,它用于网络访问任务的调度,我们的异步并发网络访问就是通过Dispatcher实现的。这里创建了一个AsyncCall,然后将它传入Dispatcher.enqueue。AsyncCall是RealCall的内部类,而且它实际上是一个Runnable：

```
final class RealCall implements Call {
  ...
  final class AsyncCall extends NamedRunnable {
    ...
  }
  ...
}
```

```
public abstract class NamedRunnable implements Runnable {
  ...
  @Override public final void run() {
    String oldName = Thread.currentThread().getName();
    Thread.currentThread().setName(name);
    try {
      execute();
    } finally {
      Thread.currentThread().setName(oldName);
    }
  }

  protected abstract void execute();
}
```

NamedRunnable在run方法里面会调用抽象的execute方法,在这个方法内部就会进行实际的网络访问。那Dispatcher.enqueue又做了写什么呢？其实Dispatcher.enqueue实际上将AsyncCall这个Runnable放到了一个线程池中：

```
public final class Dispatcher {
  ...
  synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
  ...
  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
  ...
}
```

一切明了,Call.enqueue实际上是将AsyncCall这个Runnable放到了线程池中执行去访问网络,而AsyncCall是RealCall的一个内部类,它持有RealCall的引用,所以在被线程池调用的时候可以获得Request的信息。

所以将okhttp的异步流程简化之后实际上就是Dispatcher中的线程池对Runnable的执行:

{% img /OkHttp源码解析/1.png %}

然后我们看看AsyncCall.execute的具体实现:

```
final class AsyncCall extends NamedRunnable {
  ...
  @Override protected void execute() {
   boolean signalledCallback = false;
   try {
     Response response = getResponseWithInterceptorChain(forWebSocket);
     if (canceled) {
       signalledCallback = true;
       responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
     } else {
       signalledCallback = true;
       responseCallback.onResponse(RealCall.this, response);
     }
   } catch (IOException e) {
     if (signalledCallback) {
       // Do not signal the callback twice!
       Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
     } else {
       responseCallback.onFailure(RealCall.this, e);
     }
   } finally {
     client.dispatcher().finished(this);
   }
  }
  ...
}
```

可以看到它是通过getResponseWithInterceptorChain来访问网络获取Response的。

# 同步访问

如果想用OkHttp去阻塞是的访问网络我们可以这样调用:

```
Response response = client.newCall(request).execute();
```

这个execute是不是有点眼熟,但它是Call的一个方法,并不是我们上面异步访问中提到的NamedRunnable.execute:

```
public interface Call {
  ...
  Response execute() throws IOException;
  ..
}
```

现在我们来看看具体实现:

```
final class RealCall implements Call {
  ...
  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain(false);
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      client.dispatcher().finished(this);
    }
  }
  ...
}
```

它也是通过getResponseWithInterceptorChain来访问网络获取Response的。


# 拦截器的实现

我们在前面的小节中已经知道了,无论是同步还是异步,最终都是通过RealCall.getResponseWithInterceptorChain方法去访问网络的。但是在查看具体源代码的时候发现在okhttp3.4.0-RC1开始其具体的实现细节有了一些不一样的地方。所以我这边分开两部分来讲一讲okhttp3.4.0-RC1之前和之后拦截器的具体实现细节。

## okhttp3.4.0-RC1之前的实现

okhttp3.4.0-RC1之前的RealCall.getResponseWithInterceptorChain 中实际上是调用了ApplicationInterceptorChain.proceed方法去访问网络获取Response:

```
private Response getResponseWithInterceptorChain(boolean forWebSocket) throws IOException {
  Interceptor.Chain chain = new ApplicationInterceptorChain(0, originalRequest, forWebSocket);
  return chain.proceed(originalRequest);
}
```

然后继续看源码,可以发现proceed内部会从OkHttpClient获取序号为index的拦截器,并且创建新的序号加一的ApplicationInterceptorChain传递给拦截器去执行。于是有多少个拦截器就创建了多少个ApplicationInterceptorChain,他们会按照自己的序号调用对应的拦截器。这其实就是一种责任链模式的实现方式:

```
@Override public Response proceed(Request request) throws IOException {
  // If there's another interceptor in the chain, call that.
  if (index < client.interceptors().size()) {
    Interceptor.Chain chain = new ApplicationInterceptorChain(index + 1, request, forWebSocket);
    Interceptor interceptor = client.interceptors().get(index);
    Response interceptedResponse = interceptor.intercept(chain);

    if (interceptedResponse == null) {
      throw new NullPointerException("application interceptor " + interceptor
          + " returned null");
    }

    return interceptedResponse;
  }

  // No more interceptors. Do HTTP.
  return getResponse(request, forWebSocket);
}
```

如果ApplicationInterceptorChain的序号大于OkHttpClient中注册的拦截器的数量,则调用getResponse方法。这里ApplicationInterceptorChain是RealCall的内部类,getResponse调用的是RealCall.getResponse方法。

再看RealCall.getResponse方法,它内部有个while true的死循环,调用HttpEngine.sendRequest和HttpEngine.readResponse去发送请求和接收响应,如果出现了RouteException异常或者IOException异常则重新尝试访问:

```
Response getResponse(Request request, boolean forWebSocket) throws IOException {
    ...
    while (true) {
    ...
    try {
        engine.sendRequest();
        engine.readResponse();
        releaseConnection = false;
    } catch (RouteException e) {
        HttpEngine retryEngine = engine.recover(e.getLastConnectException(), true, null);
        if (retryEngine != null) {
            releaseConnection = false;
            engine = retryEngine;
            continue;
        }
        throw e.getLastConnectException();
    }catch (IOException e) {
        HttpEngine retryEngine = engine.recover(e, false, null);
        if (retryEngine != null) {
            releaseConnection = false;
            engine = retryEngine;
            continue;
        }
        throw e;
    }
    ...
}
```

我们继续看engine.readResponse的实现,可以看到它调用了NetworkInterceptorChain.proceed方法去获取响应:

```
public void readResponse() throws IOException {
...
Response networkResponse;
...
networkResponse = new NetworkInterceptorChain(0, networkRequest,
    			streamAllocation.connection()).proceed(networkRequest);
...
}
```

NetworkInterceptorChain.proceed和ApplicationInterceptorChain.proceed类似,也会不断的创建新的NetworkInterceptorChain并且调用网络拦截器,如果没有网络拦截器可以调用了,则会调用readNetworkResponse方法读取响应:

```
@Override public Response proceed(Request request) throws IOException {
...
if (index < client.networkInterceptors().size()) {
    NetworkInterceptorChain chain = new NetworkInterceptorChain(index + 1, request, connection);
    Interceptor interceptor = client.networkInterceptors().get(index);
    Response interceptedResponse = interceptor.intercept(chain);
    ...
    return interceptedResponse;
}
Response response = readNetworkResponse();
...
return response;
}
```

这里还有一点需要说明的是NetworkInterceptorChain是HttpEngine的内部类,它调用的readNetworkResponse方法实际上是HttpEngine.readNetworkResponse。现在我们就对OkHttp拦截器的请求流程和拦截器的实现原理有了比较全面的了解,下面这张图对整个流程做一个总结:

{% img /OkHttp源码解析/2.png %}


## okhttp3.4.0-RC1之后的实现

然后让我们再来看一下3.4.0-RC1之后的实现:

```
Response getResponseWithInterceptorChain() throws IOException {
  // Build a full stack of interceptors.
  List<Interceptor> interceptors = new ArrayList<>();
  interceptors.addAll(client.interceptors());
  interceptors.add(retryAndFollowUpInterceptor);
  interceptors.add(new BridgeInterceptor(client.cookieJar()));
  interceptors.add(new CacheInterceptor(client.internalCache()));
  interceptors.add(new ConnectInterceptor(client));
  if (!forWebSocket) {
    interceptors.addAll(client.networkInterceptors());
  }
  interceptors.add(new CallServerInterceptor(forWebSocket));

  Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
      originalRequest, this, eventListener, client.connectTimeoutMillis(),
      client.readTimeoutMillis(), client.writeTimeoutMillis());

  return chain.proceed(originalRequest);
}
```

这里已经不再区分ApplicationInterceptorChain和NetworkInterceptorChain了，统一用RealInterceptorChain去处理:

```
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
    RealConnection connection) throws IOException {
  ...

  // Call the next interceptor in the chain.
  RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
      connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
      writeTimeout);
  Interceptor interceptor = interceptors.get(index);
  Response response = interceptor.intercept(next);

  ...

  return response;
}
```

这里将cookie处理、缓存处理、网络连接都作为责任链的一部分，比起3.4.0.RC-1之前更加完全的实现了责任链模式。这里有必要讲一下的就是retryAndFollowUpInterceptor, 它是一个RetryAndFollowUpInterceptor实例，它负责重连和重定向我们之前在3.4.0.RC-1之前看到的getResponse的while true就放到了这里来实现。

让我们看看它的整个流程:


{% img /OkHttp源码解析/3.png %}


这样的实现是不是以前要清晰很多？所有的步骤一目了然，看过原来的版本再看看3.4.0.RC-1重构后的版本，的确有一种眼前一亮的惊艳之感。果然好代码都是需要一点点优化出来的。
