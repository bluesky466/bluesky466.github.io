title: 学习HTTP协议-在安卓上的使用Cookie与Session
date: 2016-03-03 20:43:26
tags:
	- Http协议
	- Android
	- 技术相关
---

大三的时候写过一段时间的 php ，那时候已经对 html、css、js、cookie、session 这些东西了一点认知，但基本都是浮于表面，知其然而不知其所以然。于是这几天翻了翻《图解http》，书上的知识和自己的以前的理解结合起来，感觉对于 http 协议有了一些比较深刻的理解。

在这里把那些知识点整理记录一下，而因为 HTTP 协议的知识点较多，所以会有一个系列的博客去介绍。这篇文章就先讲一下 Cookie 和 Session 吧。

## __Cookie & Session__

毕业设计有个功能是实现用户的注册登录，而注册账号的时候需要有输入验证码的功能。

众所周知，HTTP 协议是无状态协议，即协议对于事务处理没有记忆能力。但就像这里，我们需要实现一个验证码功能，我们从服务器获取验证码的图片，然后再将用户输入的验证码传回服务器进行对比，这就要求服务器记录之前随机生成的验证码了。于是，两种用于保持HTTP连接状态的技术就应运而生了，一个是 Cookie，而另一个则是 Session。

### _Cookie_

Cookie 是由服务器端生成，发送给 User-Agent（一般是浏览器），浏览器会将 Cookie 的 key/value 保存到某个目录下的文本文件内，下次请求同一网站时就发送该 Cookie 给服务器（前提是浏览器设置为启用 cookie ）。

我们可以在 chrome 浏览器页面按 F12 打开控制台，选择 Network 标签查看与网站进行 HTTP 协议交流的数据。

这里我们看看登录[B站](http://www.bilibili.tv)的时候究竟发生了什么事情：

首先我们输入账号密码和验证码之后点击登陆，浏览器会发生账号密码等数据给服务器,之后服务器返回数据。我们查看返回报文的 header 可以看到一堆的 Set-Cookie 字段：

{% img /学习HTTP协议-在安卓上的使用Cookie与Session/1.jpg %}

客户端会把这些 cookie 记录下来，在下次访问服务器的时候就会把它们传回给服务器，这样就能实现数据的保持：

{% img /学习HTTP协议-在安卓上的使用Cookie与Session/2.jpg %}

### _Session_

但 Cookie 的数据都是保存在客户端的，客户端很容易就能查看和修改 cookie，十分不安全。例如 chrome 有一个 EditThisCookie 插件，就能直接查看修改网页的 cookie：

{% img /学习HTTP协议-在安卓上的使用Cookie与Session/3.jpg %}

所以就有了存放在服务器端的内存中的 sessio，session可以看作一个存放在服务器的键值对集。

当服务器创建一个 session 对象的时候，就会对应的生成一个sessionId，服务器可以在 session 中写入数据，但它不会将session 的内容告诉客户端，它只会将生成的 sessionId 以 cookie 的方式传给客户端，而客户端在下次访问服务器的时候把 sessionId 又传回给服务器，这样服务器就能找到之前保存的数据了。

在 php 中这个 sessionid 的名字默认叫做 PHPSESSID，当然也能在php.ini中修改。

因为 session 保存在服务器中，所以安全性比 cookie 高的多。

关于 Cookie 和 Session，各位有兴趣的话可以自己去网上搜索一下，或者希望对 HTTP 协议有更深入的理解的话可以去读一下《图解http》。最近就在读这本书，等读完我会写一篇博客，介绍一些 HTTP 协议的重点知识，这里就不再多说了。

## __okHttp3 使用 Cookie__

okHttp3 的 cookie 管理方式对比 okHttp2 有了很大的变化，这里有一篇博客专门介绍[OkHttp3实现Cookies管理及持久化](http://www.codeceo.com/article/okhttp3-cookies-manage.html)。希望各位在读我这篇博客之前先浏览一下。

okHttp3 使用 CookieJar 接口来管理 Cookie：

```java
	public interface CookieJar {
  	/** A cookie jar that never accepts any cookies. */
  	CookieJar NO_COOKIES = new CookieJar() {
    @Override public void saveFromResponse(HttpUrl url, List<Cookie> cookies) {
    }

    @Override public List<Cookie> loadForRequest(HttpUrl url) {
      return Collections.emptyList();
    }
  	};
```

我们只要在创建 OkHttpClient 的时候指定我们自己的 CookieJar 就能让 OkHttpClient 实现 Cookie 的自动管理：

```java
	public class MainActivity extends AppCompatActivity {
    	private Map<String, List<Cookie>> mCookieStore;
        ...
        private OkHttpClient createHttpClient() {
            CookieJar cookieJar = new CookieJar() {
                @Override
                public void saveFromResponse(HttpUrl url, List<Cookie> cookies) {
                    mCookieStore.put(url.host(), cookies);
                }

                @Override
                public List<Cookie> loadForRequest(HttpUrl url) {
                    List list = mCookieStore.get(url.host());
                    return list != null ? list : new ArrayList<Cookie>();
                }
            };
            return new OkHttpClient.Builder()
                    .cookieJar(cookieJar)
                    .build();
        }
        ...
	}
```

这里有点要注意，我们是拿 host 作 map 的 key 值，[《OkHttp3实现Cookies管理及持久化》](http://www.codeceo.com/article/okhttp3-cookies-manage.html)这篇博文直接用 url 当 key 值，这样的话该 Cookie 就只能在当前页面可用了，而我们是整个网站可用。

## __Retrofit 使用 Cookie__

在 Retrofit 中使用 Cookie 就更加简单了，因为它内部使用 OkHttp3，只要把之前设置了 CookieJar 的 OkHttpClient 设置给它就可以了：

```java
	HttpService service = new Retrofit.Builder()
                .client(mHttpClient) //OkHttpClient 指定了 CookieJar，这样 Retrofit 也能使用 Cookie 了
                .baseUrl(mBaseUrl)
                .addConverterFactory(GsonConverterFactory.create())
                .build()
                .create(HttpService.class);
```

## __验证码小 Demo__

现状我们用一个实现了验证码功能的小 Demo 来更加深刻的理解之前所讲的知识。

首先我写了两个页面：

- 生成验证码的页面 ： http://www.islinjw.cn/okhttp_cookie_demo/verifycode.php
- 检查验证码的页面 ： http://www.islinjw.cn/okhttp_cookie_demo/checkverifycode.php

访问第一个页面能获得一张随机的验证码图片，而第二个页面使用 GET 方法来检测验证码（键值为 verifycode）。

### _获取验证码图片_

首先我们使用 OkHttp3 访问第一个页面，下载一张验证码图片，将它显示在 ImageView 中：

```java
	private void loadVerifyCode(final ImageView imageView, final HttpUrl url) {
        final Request request = new Request.Builder()
                .url(url)
                .build();

        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    okhttp3.Response response = mHttpClient.newCall(request).execute();

                    //创建bitmap
                    InputStream is = response.body().byteStream();
                    final Bitmap bm = BitmapFactory.decodeStream(is);

                    //加载到ImageView中
                    runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            imageView.setImageBitmap(bm);
                        }
                    });

                    //打印cookie信息
                    List<Cookie> cookies = Cookie.parseAll(request.url(), response.headers());
                    for (int i = 0; i < cookies.size(); i++) {
                        Log.d("cookie", "request headers: " + cookies.get(i).toString());
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
```

运行程序可以看到验证码被显示出来：

{% img /学习HTTP协议-在安卓上的使用Cookie与Session/5.jpg %}

我们还能能看到服务器返回的 Cookie 信息，因为我的网页使用 php 写的，所以它返回了一个 PHPSESSID ，用来标记服务器保存的 Session 对象。服务器的 Session 对象里面就保存了验证码的值。之后我们把用户输入的验证码传会服务器的时候只要把这个 PHPSESSID 一同传过去，服务器就能找到之前生成的验证码的值，并和用户所输入的进行对比了:

{% img /学习HTTP协议-在安卓上的使用Cookie与Session/6.jpg %}

### _发送用户输入的验证码_

这里我们直接使用 Retrofit 将用户输入的验证码传给服务器：

```java
	public void onClick(View v) {
        HttpService service = new Retrofit.Builder()
                .client(mHttpClient)  //OkHttpClient 指定了 CookieJar，这样 Retrofit 也能使用 Cookie 了
                .baseUrl(mBaseUrl)
                .addConverterFactory(GsonConverterFactory.create())
                .build()
                .create(HttpService.class);

        String verifyCode = mEditText.getText().toString();
        if (verifyCode == null) {
            return;
        }

        Call<VerifyCodeResult> call = service.getResult(verifyCode);
        call.enqueue(new Callback<VerifyCodeResult>() {
            @Override
            public void onResponse(Call<VerifyCodeResult> call, Response<VerifyCodeResult> response) {
                Log.d("cookie", "request headers: " + call.request().headers().toString());
                Toast.makeText(MainActivity.this, response.body().getResult(), Toast.LENGTH_SHORT).show();
            }

            @Override
            public void onFailure(Call<VerifyCodeResult> call, Throwable t) {
            }
        });
```

```java
    public interface HttpService {
        @GET("okhttp_cookie_demo/checkverifycode.php")
        Call<VerifyCodeResult> getResult(@Query("verifycode") String yam);
    }
```

```java
    public class VerifyCodeResult {
        private String result;

        public String getResult() {
            return result;
        }

        public void setResult(String result) {
            this.result = result;
        }
    }
```

运行程序，输入验证码可以看到结果：

{% img /学习HTTP协议-在安卓上的使用Cookie与Session/7.jpg %}

{% img /学习HTTP协议-在安卓上的使用Cookie与Session/8.jpg %}

程序正常运行，但看 log 输出，Request 并没有把 PHPSESSID 传过去。这是怎么回事？没有传 PHPSESSID，服务器又怎么能知道之前生成的验证码是什么？

{% img /学习HTTP协议-在安卓上的使用Cookie与Session/9.jpg %}

在 CookieJar 的 loadForRequest 方法设置断点，可以发现在发送验证码的时候确实有调用，随之运行到 HttpEngine 的源码，发现原来框架创建了个新的 Resquest 副本，将 Cookie 传入这个新的副本中去连接服务器：

```java
    public void sendRequest() throws RequestException, RouteException, IOException {
        ...
        Request request = networkRequest(userRequest);
		...
    }
	...
    private Request networkRequest(Request request) throws IOException {
        Request.Builder result = request.newBuilder();

        if (request.header("Host") == null) {
          result.header("Host", hostHeader(request.url()));
        }

        if (request.header("Connection") == null) {
          result.header("Connection", "Keep-Alive");
        }

        if (request.header("Accept-Encoding") == null) {
          transparentGzip = true;
          result.header("Accept-Encoding", "gzip");
        }

		//看这里，其实是有设置 CookieJar 中的 Cookie 的
        //也就是说 PHPSESSID 有传回去给服务器
        List<Cookie> cookies = client.cookieJar().loadForRequest(request.url());
        if (!cookies.isEmpty()) {
          result.header("Cookie", cookieHeader(cookies));
        }

        if (request.header("User-Agent") == null) {
          result.header("User-Agent", Version.userAgent());
        }

        return result.build();
      }
```

原来如此，操作都使用了副本 Request 去执行，怪不得我们直接用下面的代码输出，请求头部不能看到 PHPSESSID 的 Cookie 值：

```java
	@Override
    public void onResponse(Call<VerifyCodeResult> call, Response<VerifyCodeResult> response) {
        Log.d("cookie", "request headers: " + call.request().headers().toString());
        Toast.makeText(MainActivity.this, response.body().getResult(), Toast.LENGTH_SHORT).show();
    }
```

## __Glide 使用 Cookie__

Glide 是Google推荐的图片加载库，用来加载图片十分之方便，最少只需要三行代码就能将网络图片加载到 ImageView 上。

我有在 Glide 的文档上看到它也能使用 OkHttp3，理论上应该也能使用设置 OkHttpClient 的方法使用 Cookie。

但弄了很久还是没有搞定，等以后有时间找到实现方法再把这一节补全。

## __Demo 完整代码__

MainActivity:

```java
	public class MainActivity extends AppCompatActivity {
    private static final String mBaseUrl = "http://www.islinjw.cn";
    private static final String mVerifyCideUrl = "http://www.islinjw.cn/okhttp_cookie_demo/verifycode.php";

    private OkHttpClient mHttpClient;
    private EditText mEditText;
    private ImageView mImageView;
    private Map<String, List<Cookie>> mCookieStore;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mEditText = (EditText) findViewById(R.id.input);
        mImageView = (ImageView) findViewById(R.id.yzm);
        mCookieStore = new HashMap<>();
        mHttpClient = createHttpClient();

        loadVerifyCode(mImageView, HttpUrl.parse(mVerifyCideUrl));
    }

    private OkHttpClient createHttpClient() {
        CookieJar cookieJar = new CookieJar() {
            @Override
            public void saveFromResponse(HttpUrl url, List<Cookie> cookies) {
                mCookieStore.put(url.host(), cookies);
            }

            @Override
            public List<Cookie> loadForRequest(HttpUrl url) {
                List list = mCookieStore.get(url.host());
                return list != null ? list : new ArrayList<Cookie>();
            }
        };
        return new OkHttpClient.Builder()
                .cookieJar(cookieJar)
                .build();
    }

    private void loadVerifyCode(final ImageView imageView, final HttpUrl url) {
        final Request request = new Request.Builder()
                .url(url)
                .build();

        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    okhttp3.Response response = mHttpClient.newCall(request).execute();

                    //创建bitmap
                    InputStream is = response.body().byteStream();
                    final Bitmap bm = BitmapFactory.decodeStream(is);

                    //加载到ImageView中
                    runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            imageView.setImageBitmap(bm);
                        }
                    });

                    //打印cookie信息
                    List<Cookie> cookies = Cookie.parseAll(request.url(), response.headers());
                    for (int i = 0; i < cookies.size(); i++) {
                        Log.d("cookie", "response headers: " + cookies.get(i).toString());
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }

    public void onClick(View v) {
        HttpService service = new Retrofit.Builder()
                .client(mHttpClient)  //OkHttpClient 指定了 CookieJar，这样 Retrofit 也能使用 Cookie 了
                .baseUrl(mBaseUrl)
                .addConverterFactory(GsonConverterFactory.create())
                .build()
                .create(HttpService.class);

        String verifyCode = mEditText.getText().toString();
        if (verifyCode == null) {
            return;
        }

        Call<VerifyCodeResult> call = service.getResult(verifyCode);
        call.enqueue(new Callback<VerifyCodeResult>() {
            @Override
            public void onResponse(Call<VerifyCodeResult> call, Response<VerifyCodeResult> response) {
                Log.d("cookie", "request headers: " + call.request().headers().toString());
                Toast.makeText(MainActivity.this, response.body().getResult(), Toast.LENGTH_SHORT).show();
            }

            @Override
            public void onFailure(Call<VerifyCodeResult> call, Throwable t) {
            }
        });
    }
}
```

HttpService:

```java
	public interface HttpService {
        @GET("okhttp_cookie_demo/checkverifycode.php")
        Call<VerifyCodeResult> getResult(@Query("verifycode") String yam);
    }
```

```java
	public class VerifyCodeResult {
        private String result;

        public String getResult() {
            return result;
        }

        public void setResult(String result) {
            this.result = result;
        }
    }
```