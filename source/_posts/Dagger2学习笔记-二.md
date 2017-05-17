title: Dagger2学习笔记(二)
date: 2017-05-17 00:27:30
tags:
	- 技术相关
	- Android
---

在上一篇[文章](http://blog.islinjw.cn/2017/05/13/Dagger2%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-%E4%B8%80/)我们讲了用于搜索的SearchActivity的实现，这一篇文章我们继续以剩下的两个Activity的实现为例，讲一下Dagger2的其他特性。这两个Activity分别是用了展示SearchActivity搜索的用户的头像和用户名的UserInfoActivity和点击用户头像跳转到的展示用户followers的FollowerActivity。


在我们的Demo中有个叫做UserInfoLoader的类，它是用来向github服务器请求用户信息和follower信息的，会在多个actiity中被使用，例如在FollowerPresenter和UserInfoPresenter中都需要注入UserInfoLoader。最简单的方式是我们可以直接使用@Inject注解标注它的构造方法，使得Dagger2可以直接创建它的实例去注入FollowerPresenter和UserInfoPresenter中。

```
class UserInfoLoader {
	...
	@Inject
	UserInfoLoader() {
	}
	...
}

```

# Module复用

当然我们也能用复用Module的方式，这种方式虽然比直接用@Inject注解构造方法复杂，但是它还有其他十分有用的功能，接下来我会慢慢分析。

首先我们把它的Module单独抽出来，放到AppModule中：

```
@Module
public class AppModule {
    @Provides
	UserInfoLoader provideUserInfoLoader() {
        return new UserInfoLoader();
	}
}
```

## 共用Module

我们复用这个Module的方式有几种，一是同时放在FollowerComponent和UserInfoComponent的modules中:

```
@Component(modules = {AppModule.class, FollowerPresenterModule.class})
public interface FollowerComponent {
    void inject(FollowerPresenter presenter);
    void inject(FollowerActivity activity);
}

```

```
@Component(modules = {AppModule.class, UserInfoPresenterModule.class})
public interface UserInfoComponent {
    void inject(UserInfoPresenter presenter);
    void inject(UserInfoActivity activity);
}
```

## 使用dependencies

第二种方式是使用dependencies，首先我们需要声明多一个AppComponent接口

```
@Component(modules = {AppModule.class})
public interface AppComponent {
    UserInfoLoader provideUserInfoLoader();
}

```

这个接口的provideUserInfoLoader()方法就是提供出来给子依赖获取UserInfoLoader的，因为dependencies子依赖是获取不了父依赖的modules里面的Provides的。

之后声明FollowerComponent和UserInfoComponent:

```
@Component(dependencies = AppComponent.class, modules = {UserInfoPresenterModule.class})
public interface UserInfoComponent {
    void inject(UserInfoPresenter presenter);
    void inject(UserInfoActivity activity);
}
```

```
@Component(dependencies = AppComponent.class, modules = {FollowerPresenterModule.class})
public interface FollowerComponent {
    void inject(FollowerPresenter presenter);
    void inject(FollowerActivity activity);
}

```

最后就再去实现注入:
```
FollowerComponent component = DaggerFollowerComponent.builder()
	.appComponent(getAppComponent())
    .followerPresenterModule(new FollowerPresenterModule(this))
    .build();
component.inject(this);
component.inject(mPresenter);
```

```
UserInfoComponent component = DaggerUserInfoComponent.builder()
    .appComponent(getAppComponent())
    .userInfoPresenterModule(new UserInfoPresenterModule(this))
    .build();
component.inject(this);
component.inject(mPresenter);
```

这里的AppComponent是公用的，所以我们放到Application中:

```
public class AppApplication extends Application {
    private AppComponent mAppComponent;

	public AppApplication() {
        super();

        mAppComponent = DaggerAppComponent.create();
	}

	public AppComponent getAppComponent() {
        return mAppComponent;
	}
}
```

然后在Activity中这样获取AppComponent:

```
AppComponent getAppComponent() {
    return ((AppApplication)getApplication()).getAppComponent();
}
```

我们尝试注释掉AppComponent.provideUserInfoLoader，rebuild一下，发现居然没有报错，这是怎么回事？其实是因为UserInfoLoader的构造方法使用@Inject注解标注了，所以可以直接通过构造方法创建UserInfoLoader来注入FollowerPresenter和FollowerActivity。

我们再把UserInfoLoader的构造方法的@Inject注解注释掉，这时候再rebuild就可以发现报错了。

然后再取消掉AppComponent.provideUserInfoLoader的注释，就能顺利编过了。因为我们的AppModule.provideUserInfoLoader是通过new 一个UserInfoLoader出来的，所以可以不依赖构造方法的@Inject注解。

## 使用Subcomponent

最后一种方法就是使用@Subcomponent注解，这中方法和使用dependencies有点像，他们的区别在于使用@Subcomponent方法AppComponent不需要提供一个provideUserInfoLoader方法，子依赖可以直接使用AppComponent中的modules。首先我们要这样声明AppComponent:

```
@Component(modules = {AppModule.class})
public interface AppComponent {
    FollowerComponent plus(FollowerPresenterModule module);
    UserInfoComponent plus(UserInfoPresenterModule module);
}
```

然后FollowerComponent和UserInfoComponent的定义如下:

```
@Subcomponent(modules = {FollowerPresenterModule.class})
public interface FollowerComponent {
    void inject(FollowerPresenter presenter);
    void inject(FollowerActivity activity);
}
```

```
@Subcomponent(modules = {UserInfoPresenterModule.class})
public interface UserInfoComponent {
    void inject(UserInfoPresenter presenter);
    void inject(UserInfoActivity activity);
}
```

注入的实现代码如下:

```
FollowerComponent component = getAppComponent().plus(new FollowerPresenterModule(this));

component.inject(this);
component.inject(mPresenter);
```

# Scope

现在还有一个问题，现在FollowerComponent和UserInfoComponent虽然都往Presenter注入了UserInfoLoader，但他们是不同的实例:

> D/UserInfoPresenter: mUserInfoLoader : com.example.linjw.dagger2demo.model.UserInfoLoader@31e117c
> D/FollowerPresenter: mUserInfoLoader : com.example.linjw.dagger2demo.model.UserInfoLoader@c9ad63b

如果我想他们使用的就是同一个UserInfoLoader实例呢？需要怎么做？

Dagger2中有作用域的概念，可以规定几个Component在同一个作用域，在同一个作用域注入的依赖就是同一个实例。

首先需要声明我们的Scope:

```
@Scope
@Retention(RUNTIME)
public @interface AppScope {
}
```

然后就只需要将Module的Provides方法和Component用同一个Scope注解标注一下，就能让他们处于同一个作用域了。

比如我们需要在AppModule.provideUserInfoLoader标注:
```
@Module
public class AppModule {
    @AppScope
    @Provides
	UserInfoLoader provideUserInfoLoader() {
        return new UserInfoLoader();
	}
}
```

像我们使用Subcomponent去实现依赖继承，我们就只需要在AppComponent中标注就好了，这样他们的子依赖也会处于AppScope中:

```
@AppScope
@Component(modules = {AppModule.class})
public interface AppComponent {
    FollowerComponent plus(FollowerPresenterModule module);
    UserInfoComponent plus(UserInfoPresenterModule module);
}
```

> D/UserInfoPresenter: mUserInfoLoader : com.example.linjw.dagger2demo.model.UserInfoLoader@31e117c
> D/FollowerPresenter: mUserInfoLoader : com.example.linjw.dagger2demo.model.UserInfoLoader@31e117c

# Demo地址

可以在[这里](https://github.com/bluesky466/Dagger2Demo)查看完整代码
