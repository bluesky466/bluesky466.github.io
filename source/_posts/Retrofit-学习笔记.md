title: Retrofit 学习笔记
date: 2016-02-22 16:19:56
tags:
	- 技术相关
	- Android
---

工欲善其事必先利其器，使用一些强大方便的器，可以大大的提高开发的效率，我认为 Retrofit 和 RxJava 就是这样的利器。

Retrofit 是一个开源的 java http 请求库，目前已经更新到 2.0.0-beta4，官方的介绍是：

> Type-safe HTTP client for Android and Java by Square, Inc.

我在学习它的过程中遇到了不少问题，于是写了这篇东西把遇到的问题和解决方法都记录一下。

## __android studio 导入 Retrofit__

首先我使用的是 android studio，一开始搜索怎样使用第三方库的时候看到了不少的文章，有介绍导入 jar 包的，有介绍源码库的，但使用方法和我接下来介绍的都显得复杂很多。

比如现在我们要使用 Retrofit ，先登录 [http://search.maven.org/](http://search.maven.org/)，搜索 Retrofit 我们可以看到搜出了不少东西，我们直接用最新的版本 2.0.0-beta4，可以看到它有两个版本，点进去看看：

{% img /Retrofit-学习笔记/1.jpg %}

原来一个是beta3，一个是beta4：

{% img /Retrofit-学习笔记/2.jpg %}

然后我们打开 android studio 项目的 build.gradle(Module: app)， 在它的 dependencies 里面加上  retrofit 的引用:

```
    dependencies {
        compile fileTree(dir: 'libs', include: ['*.jar'])
        testCompile 'junit:junit:4.12'
        compile 'com.android.support:appcompat-v7:23.1.1'
        compile 'com.squareup.retrofit2:retrofit:2.0.0-beta4'
    }
```

要使用 [http://search.maven.org/](http://search.maven.org/)，搜索到的第三方库只需要在 dependencies 里面加上

> compile 'GroupId:ArtifactId:Version'

对应 retrofit 就是这句：

> compile 'com.squareup.retrofit2:retrofit:2.0.0-beta4'

再点击 android studio 弹出的 Sync Now，android studio 就会帮你自动下载和配置第三方库，而你就能直接使用了。

{% img /Retrofit-学习笔记/3.jpg %}

## __使用 Retrofit 的 Call 类获取 github 用户的信息__

官方文档一开始就展示了一个简单的demo [http://square.github.io/retrofit/](http://square.github.io/retrofit/)

可惜都是代码碎片，你按照它写好代码之后就会发现......报异常了。

{% img /Retrofit-学习笔记/4.jpg %}

{% img /Retrofit-学习笔记/5.jpg %}

我先把讲讲我写的demo，最后再告诉你们官方文档到底哪里出问题了。

首先定义一个 User 类用于保存获取到的用户信息：

```java
    public class User {
        private String login;
        private long id;
        private String name;

        @Override
        public String toString() {
            return "login:" + login + ", "
                    + "id:" + id + ", "
                    + "name:" + name;
        }

        public String getLogin() {
            return login;
        }

        public void setLogin(String login) {
            this.login = login;
        }

        public long getId() {
            return id;
        }

        public void setId(long id) {
            this.id = id;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }
    }
```

接着定义一个接口用于告诉 Retrofit 怎样去获取数据，如下代码就表明使用 get 方法获取 users 路径下的 user 资源，使用的最终使用的时候，传入的 user 参数会替换 users/{user} 的 {user} 字段。

不过我还真不知道它这里到底是怎么实现的，看来 java 基础还真要去补一补才行了。

```java
     public interface GitHubService {
            @GET("users/{user}")
            Call<User> getUserInfoByCall(@Path("user") String user);
        }
```

之后就可以创建一个 GitHubService 实例了（我总觉得有点 java 黑魔法的感觉）：

```java
     GitHubService service = new Retrofit.Builder()
                    .baseUrl("https://api.github.com")
                    .addConverterFactory(GsonConverterFactory.create())
                    .build()
                    .create(GitHubService.class);
```

然后调用 service 的 getUserInfoByCall 方法就能获取到一个 Call 对象了。像如下的代码，就能获取到一个用于访问 [https://api.github.com/users/bluesky466](https://api.github.com/users/bluesky466) 的 Call 对象：

```java
	final Call<User> call = service.getUserInfoByCall("bluesky466");
```

最后就能用 call 的 execute 方法访问服务器获取用户数据了。因为 execute 是同步的，而安卓不允许在 ui 线程访问网络，所以我们需要用一个子线程去访问。

```java
    new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Response<User> response = call.execute();
                        User user = response.body();
                        Log.d("result", user.toString());
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
```

结果如下：

{% img /Retrofit-学习笔记/6.jpg %}

当然，Retrofit 也提供了异步访问的方法：

```java
     call.enqueue(new Callback<User>() {
                @Override
                public void onResponse(Call<User> call, Response<User> response) {
                    User user = response.body();
                	Log.d("result", user.toString());
                }

                @Override
                public void onFailure(Call<User> call, Throwable t) {

                }
            });
```

Retrofit 用获取到的数据生成了一个User对象。这是什么黑魔法？

还记得我一开始说的按照官方文档的代码会报异常吗？

官方文档的代码：

```java
    Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("https://api.github.com")
        .build();

    GitHubService service = retrofit.create(GitHubService.class);
```

我的代码：

```java
    GitHubService service = new Retrofit.Builder()
                        .baseUrl("https://api.github.com")
                        .addConverterFactory(GsonConverterFactory.create())
                        .build()
                        .create(GitHubService.class);
```

差别就在在这里：

```
	.addConverterFactory(GsonConverterFactory.create())
```

我这里指定了一个 Gson 转换工厂，因为 [https://api.github.com](https://api.github.com) 使用josn 格式返回数据，所以我们可以使用 Gson 去解析它，然后生成一个 User 对象。

不过就算你按我这样写代码又会发现找不到 GsonConverterFactory 的包......

{% img /Retrofit-学习笔记/7.jpg %}

原因在于 GsonConverterFactory 使用来转换json的，你也可以指定其他的 Factory 去转换 xml 之类的格式。而这些 Factory 并不包含在 Retrofit 库里面，需要用户自己去导入。

{% img /Retrofit-学习笔记/8.jpg %}

在 dependencies 中加入：

```
	compile 'com.squareup.retrofit2:converter-gson:2.0.0-beta4'
```

类似的库有下面这些：
- __Gson__: com.squareup.retrofit2:converter-gson
- __Jackson__: com.squareup.retrofit2:converter-jackson
- __Moshi__: com.squareup.retrofit2:converter-moshi
- __Protobuf__: com.squareup.retrofit2:converter-protobuf
- __Wire__: com.squareup.retrofit2:converter-wire
- __Simple__ XML: com.squareup.retrofit2:converter-simplexml
- __Scalars__ (primitives, boxed, and String): com.squareup.retrofit2:converter-scalars


## __使用 Retrofit 配合 RxJava 获取 github 用户的信息__

RxJava 在 GitHub 主页上的自我介绍是 "a library for composing asynchronous and event-based programs using observable sequences for the Java VM"（一个在 Java VM 上使用可观测的序列来组成异步的、基于事件的程序的库）。这就是 RxJava ，概括得非常精准。

它可以用来替换 AsyncTask 之类的东西。

关于 RxJava 有一篇很好的博客 -- [给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)。这里基本上把 RxJava 讲的很透彻了。我这里就不多说，只是讲一讲怎样在 Retrofit 中使用 RxJava。

在 Retrofit 中使用 RxJava 首先需要导入 adapter-rxjava 库，而且因为是在安卓上使用，所以需要导入 RxJava 的 Android 平台的扩展 rxandroid 库：

```
	compile 'com.squareup.retrofit2:adapter-rxjava:2.0.0-beta4
    compile 'io.reactivex:rxandroid:1.1.0'
```

添加 GitHubService 接口的 RxJava 获取方法：
```java
    public interface GitHubService {
            @GET("users/{user}")
            Call<User> getUserInfoByCall(@Path("user") String user);

            @GET("users/{user}")
            Observable<User> getUserInfoByObservable(@Path("user") String user);
        }
```

然后创建 GitHubService 的时候需要指定 RxJava的适配工厂：

```java
     GitHubService service = new Retrofit.Builder()
                    .baseUrl("https://api.github.com")
                    .addConverterFactory(GsonConverterFactory.create())//指定RxJava适配工厂
                    .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                    .build()
                    .create(GitHubService.class);
```

然后就是指定 subscribe 去输出结果了：

```java
    service.getUserInfoByObservable("bluesky466")
                    .subscribeOn(Schedulers.newThread())
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Subscriber<User>() {
                        @Override
                        public void onNext(User user) {
                            if (user != null) {
                                Log.d("result", user.toString());
                            }
                        }

                        @Override
                        public void onCompleted() {
                        }

                        @Override
                        public void onError(Throwable e) {
                        }
                    });
```

## __一个小 Demo__

我写了一个 demo 用来展示 Retrofit 的用法，完整源码如下：

MainActivity：

```java
    public class MainActivity extends AppCompatActivity {
        private GitHubService mService;
        private EditText mUserName;
        private TextView mResult;

        public interface GitHubService {
            @GET("users/{user}")
            Call<User> getUserInfoByCall(@Path("user") String user);

            @GET("users/{user}")
            Observable<User> getUserInfoByObservable(@Path("user") String user);
        }

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);

            mUserName = (EditText) findViewById(R.id.username);
            mResult = (TextView) findViewById(R.id.result);

            Button btnCall = (Button) findViewById(R.id.btnCall);
            btnCall.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    queryByCall(mUserName.getText().toString());
                }
            });

            Button btnObservable = (Button) findViewById(R.id.btnObservable);
            btnObservable.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    queryByObservable(mUserName.getText().toString());
                }
            });

            mService = new Retrofit.Builder()
                    .baseUrl("https://api.github.com")
                    .addConverterFactory(GsonConverterFactory.create())
                    .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                    .build()
                    .create(GitHubService.class);
        }

        private void queryByCall(final String username) {
            mService.getUserInfoByCall(username).enqueue(new Callback<User>() {
                @Override
                public void onResponse(Call<User> call, Response<User> response) {
                    if (response.body() != null) {
                        mResult.setText("[ByCall] " + response.body().toString());
                    } else {
                        mResult.setText("[ByCall] Not Found");
                    }
                }

                @Override
                public void onFailure(Call<User> call, Throwable t) {
                    mResult.setText("[ByCall] Not Found");
                }
            });
        }

        private void queryByObservable(final String username) {
            mService.getUserInfoByObservable(username)
                    .subscribeOn(Schedulers.newThread())
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Subscriber<User>() {
                        @Override
                        public void onCompleted() {
                        }

                        @Override
                        public void onError(Throwable e) {
                            mResult.setText("[ByObservable] Not Found");
                        }

                        @Override
                        public void onNext(User user) {
                            if (user != null) {
                                mResult.setText("[ByObservable] " + user.toString());
                            }
                        }
                    });
        }
    }
```

activity\_main.xml:

```xml
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal">

            <EditText
                android:id="@+id/username"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:layout_weight="1"
                android:hint="user name" />

            <Button
                android:id="@+id/btnCall"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Call" />

            <Button
                android:id="@+id/btnObservable"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Observable" />
        </LinearLayout>

        <TextView
            android:id="@+id/result"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="" />
    </LinearLayout>
```

User:

```java
    public class User {
        private String login;
        private long id;
        private String name;

        @Override
        public String toString() {
            return "login:" + login + ", "
                    + "id:" + id + ", "
                    + "name:" + name;
        }

        public String getLogin() {
            return login;
        }

        public void setLogin(String login) {
            this.login = login;
        }

        public long getId() {
            return id;
        }

        public void setId(long id) {
            this.id = id;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }
    }

```

dependencies:

```
	dependencies {
        compile fileTree(dir: 'libs', include: ['*.jar'])
        testCompile 'junit:junit:4.12'
        compile 'com.android.support:appcompat-v7:23.1.1'
        compile 'com.squareup.retrofit2:retrofit:2.0.0-beta4'
        compile 'com.squareup.retrofit2:converter-gson:2.0.0-beta4'
        compile 'com.squareup.retrofit2:adapter-rxjava:2.0.0-beta4'
        compile 'io.reactivex:rxandroid:1.1.0'
    }
```
