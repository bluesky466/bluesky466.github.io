title: Dagger2学习笔记(一)
date: 2017-05-13 06:53:27
tags:
	- 技术相关
	- Android
---

依赖注入是一种十分好的技巧，它能解偶高层次模块和低层次模块，使得高层模块不用将底层模块硬编码到内部。
所有依赖的底层模块都由外部注入，实际是一种面向接口编程。高层模块不依赖底层模块的实现细节，可以方便的做到替换底层模块。
这种技术在编写跨平台程序的时候可以很容易的替换调依赖系统的底层模块，并且在做单元测试的时候也可以很容易的使用stub对象注入宿主类中从而方便宿主类的测试代码的编写。

# 使用Dagger2实现依赖注入

如果不使用DI框架，我们也可以在构造方法里传入依赖类或着用setter方法来将依赖类注入宿主类。但是这样的话就会需要我们在业务逻辑中处理依赖类的生成和注入，其实这些依赖的注入代码和业务都没有什么关系仅仅是一些初始化的操作而已，如果可以将这些与业务逻辑无关的代码都独立出去，这样的话我们的代码逻辑就会更加的简洁和清晰。Dagger2就是一个十分强大的DI框架，它可以帮助我们轻松的在业务逻辑之外实现依赖注入。

## Dagger2的两个重要组件

Dagger2有两个十分重要的组件：Module和Component。

- Module

	Module是依赖的提供者，Dagger2框架通过Module的Provides方法获取被依赖类的实例。

- Component

	Component是一个注入接口，Dagger2框架通过Component将依赖注入到高层类中。

	用一个形象的比喻来说明就是Module是装有被依赖类的针筒，Component是针头。Dagger2通过选择针筒和针头的不同组合可以将不同的被依赖实例注入到高层模块中。


## 举个栗子

下面我们使用MVP模式写一个展示github头像和用户名的小Demo来具体介绍Dagger2的用法。

### Dagger2的引入

Dagger2没有使用反射，它是通过编译时生成代码来实现依赖注入的。所以需要引入apt:

```
//build.gradle(project)
...
buildscript {
	repositories {
		jcenter()
	}

	dependencies {
		classpath 'com.android.tools.build:gradle:2.3.0'
		classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
	}
}
...
```

```
//build.gradle(app)
apply plugin: 'com.android.application'
apply plugin: 'com.neenbedankt.android-apt'
...
```

之后再引入javax.annotation和dagger2:
```
//build.gradle(app)
...
dependencies {
	...
	compile 'com.google.dagger:dagger:2.4'
	apt 'com.google.dagger:dagger-compiler:2.4'
	compile 'org.glassfish:javax.annotation:10.0-b28'
	...
}
```

### 定义Module

```
@Module
public class UserInfoLoaderModule {
    @Provides
	UserInfoLoader provideUserInfoLoader() {
	    return new UserInfoLoader();
	}
}
```

Module类首先需要使用@Module注解标注，让Dagger2知道这是一个Module，然后内部的使用@Provides注解标注的方法就是用来获取被依赖类的实例的方法

这里的UserInfoLoader就是一个底层的被依赖类，它用来通过网络去请求github服务器，查询用户的头像和用户名信息

Dagger2可以通过provideUserInfoLoader方法获取到被依赖类的实例。



### 为高层类需要注入的成员变量添加@Inject注解

我们的高层类是UserInfoPresenter,是MVP模式中的Presenter，UserInfoLoader是它的一个成员变量。

```
public class UserInfoPresenter {
	...
	@Inject
	UserInfoLoader mUserInfoLoader;
	...
}
```

我们通过@Inject注解告诉Dagger2哪些成员变量是需要被注入的，这里需要注意的是被@Inject标注的成员变量不可以是private的，因为Dagger2没有用到反射，而是通过生成代码去完成注入的，所以一旦你将成员变量声明成private的，那Dagger2就不能访问到它，从而无法无法完成注入了。

### 定义Component

```
@dagger.Component(modules = {UserInfoLoaderModule.class})
public interface Component {
    void inject(UserInfoPresenter presenter);
}
```

Component是一个被@Component注解标注的接口，Dagger2会自动生成实现这个接口的类，去完成注入的功能。我们需要用modules去告诉Component从哪个Module中获取被依赖类的实例。


### 在合适的地方调用注入方法实现注入

Dagger2会生成实现Component接口的类,这个类会在接口的前面加上Dagger前缀，我们通过调用inject方法就可以将UserInfoLoader注入到UserInfoPresenter中:

```
Component component = DaggerComponent.builder().build();
component.inject(mPresenter);
```

它实际是通过查找UserInfoPresenter中带有@Inject注解的成员变量知道哪个变量需要被注入，然后通过UserInfoLoaderModule的provide方法获取到这个被依赖类的实例去实现注入的。


## 再举个栗子

有时候被依赖类的构造方法是需要传参的，例如这里的UserInfoPresenter在构造的时候需要参入UserInfoView:

```
public class UserInfoPresenter {
	...
	private UserInfoView mView;

	@Inject
	UserInfoLoader mUserInfoLoader;

	public UserInfoPresenter(UserInfoView view) {
		mView = view;
	}
	...
}

```

我们也希望通过Dagger2将它注入到Activity中。这样的话就要这么定义Module了:

```
@Module
public class UserInfoPresenterModule {
    private UserInfoView mView;

	public UserInfoPresenterModule(UserInfoView view) {
	        mView = view;
	}

	@Provides
	UserInfoPresenter provideUserInfoPresenter() {
		return new UserInfoPresenter(mView);
	}
}
```

然后在Component中加入UserInfoPresenter的注入方法:

```
@dagger.Component(modules = {UserInfoPresenterModule.class, UserInfoLoaderModule.class})
public interface Component {
    void inject(UserInfoPresenter presenter);

	void inject(MainActivity activity);
}

```

在最后实际注入的时候需要将UserInfoPresenterModule从外部传到DaggerComponent中，因为UserInfoPresenterModule的构造函数带参数，所以必须在外面创建好，然后通过userInfoPresenterModule方法传给DaggerComponent builder。

```
Component component = DaggerComponent.builder()
        .userInfoPresenterModule(new UserInfoPresenterModule(this))
		.build();
component.inject(this);
component.inject(mPresenter);

```

这段代码是在MainActivity中的，UserInfoPresenter是MainActivity的一个需要注入的成员变量。具体的实现细节可以查看[完整代码](https://github.com/bluesky466/Dagger2Demo/tree/v0.1)。



