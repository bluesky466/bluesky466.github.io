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

如果不使用DI框架，我们也可以在构造方法里传入依赖类或着用setter方法来将依赖类注入宿主类。但是这样的话就会需要我们在业务逻辑中处理依赖类的生成和注入，其实这些依赖的注入代码和业务都没有什么关系，仅仅是一些初始化的操作而已，如果可以将这些与业务逻辑无关的代码都独立出去，这样的话我们的代码逻辑就会更加的简洁和清晰。Dagger2就是一个十分强大的DI框架，它可以帮助我们轻松的在业务逻辑之外实现依赖注入。

下面我将用一个小Demo来介绍一下Dagger2的用法。这个小Demo的功能是通过github帐号搜索用户头像和用户名，同时列出该用户的follower

## Dagger2的引入

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

## Dagger2的两个重要组件

Dagger2有两个十分重要的组件：Module和Component。

- Module

	Module是依赖的提供者，Dagger2框架通过Module的Provides方法获取被依赖类的实例。

- Component

	Component是一个注入接口，Dagger2框架通过Component将依赖注入到高层类中。

	用一个形象的比喻来说明就是Module是装有被依赖类的针筒，Component是针头。Dagger2通过选择针筒和针头的不同组合可以将不同的被依赖实例注入到高层模块中。

## 实现搜索页面

### @Inject注解

我们的搜索页面很简单，只有一个输入框和一个搜索安按钮，它的作用是输入要搜索的用户的账号。我们使用MVP模式去实现它,因为它不需要model层，所以只有View和Presenter:

```
public interface SearchView {
	...
}
```

```
public class SearchPresenter{
	...
	@Inject
	SearchView mSearchView;

    @Inject
    Context mContext;

	@Inject
    public SearchPresenter() {
        Log.d(TAG, "SearchPresenter()");
    }
	...
}
```

```
public class SearchActivity extends Activity implements SearchView {
	...
	@Inject
	SearchPresenter mSearchPresenter;
	...
}
```

我们通过@Inject注解告诉Dagger2哪些成员变量是需要被注入的，这里需要注意的是被@Inject标注的成员变量不可以是private的，因为Dagger2没有用到反射，而是通过生成代码去完成注入的，所以一旦你将成员变量声明成private的，那Dagger2就不能访问到它，从而无法无法完成注入了。@Inject还有另外一个作用就是告诉Dagger2用哪个构造函数去创建实例，如这里Dagger2就会用SearchPresenter()去创建SearchPresenter的实例，这个构造函数的作用在接下来就会被讲到。

### Module

然后再让我们来看看SearchPresenterModule:

```
@Module
public class SearchPresenterModule {
    private SearchActivity mSearchActivity;

    public SearchPresenterModule(SearchActivity view) {
        mSearchActivity = view;
    }

    @Provides
    SearchView provideSearchView() {
        return mSearchActivity;
    }

    @Provides
    Context provideContext() {
        return mSearchActivity;
    }
}
```

注入SearchPresenter所需要的SearchView和Context就是从这里提供的

Module类首先需要使用@Module注解标注，让Dagger2知道这是一个Module，然后内部的使用@Provides注解标注的方法就是用来获取被依赖类的实例的方法,例如provideSearchView就可以用来提供SearchView

一般我习惯@Provide方法加上provide前缀，但是这个也不是必须，可以没有这个前缀。

### Component

接着看看Component:

```
@Component(modules = {SearchPresenterModule.class})
public interface SearchComponent {
    void inject(SearchActivity activity);

    void inject(SearchPresenter presenter);
}
```

Component是一个被@Component注解标注的接口，Dagger2会自动生成实现这个接口的类，去完成注入的功能。我们需要用modules去告诉Component从哪个Module中获取被依赖类的实例。这里Dagger2就会自动生成实现了SearchComponent接口的DaggerSearchComponent类，它有两个方法，分别用来向SearchActivity和SearchPresenter注入依赖。

向SearchPresenter注入的SearchView和Context都是SearchPresenterModule提供的这个很容易理解，但是向SearchActivity注入的SearchPresenter又是从哪里来的呢?还记得我们用@Inject标注了SearchPresenter的一个构造函数了吗？Dagger2会使用我们标注的构造函数创建出一个SearchPresenter来给SearchActivity注入使用。

### 调用注入方法实现注入

在SearchActivity的onCreate方法中将依赖注入到SearchActivity和SearchPresenter中:


```
SearchComponent component = DaggerSearchComponent.builder()
                .searchPresenterModule(new SearchPresenterModule(this))
                .build();

component.inject(this);
component.inject(mSearchPresenter);
```

它实际是通过查找SearchActivity和SearchPresenter中带有@Inject注解的成员变量知道哪个变量需要被注入，然后通过SearchPresenterModule的provide方法和SearchPresenter被标注的构造方法获取到被依赖类的实例去实现注入的。

这里有一点需要注意的是调用顺序，inject(SearchActivity activity)要在inject(SearchPresenter presenter)前面调用，因为需要先将SearchActivity.this的mSearchPresenter注入，才能向mSearchPresenter中再注入SearchActivity

### 指定构造函数

我们在前面讲到过@Inject可以指定构造函数，其实它还有另一重意义，就是存在多个构造函数的时候选择其中一种。

我们现在添加另外一种SearchPresenter构造函数,然后中添加打印:

```
public class SearchPresenter{
	...
	@Inject
	SearchView mSearchView;

    @Inject
    Context mContext;

	@Inject
    public SearchPresenter() {
        Log.d(TAG, "SearchPresenter()");
    }

	public SearchPresenter(Context context) {
	    Log.d(TAG, "SearchPresenter(Context context)");
		mContext = context;
	}
	...
}
```

让我们看看运行的时候到底调的是哪个构造函数吧:

> D/SearchPresenter(27333): SearchPresenter()

如果我们把SearchPresenter类修改一下呢?

```
public class SearchPresenter{
	...
	@Inject
	SearchView mSearchView;

    // @Inject 注释掉
    Context mContext;

	// @Inject 注释掉
    public SearchPresenter() {
        Log.d(TAG, "SearchPresenter()");
    }

	@Inject // 添加@Inject
	public SearchPresenter(Context context) {
	    Log.d(TAG, "SearchPresenter(Context context)");
		mContext = context;
	}
	...
}
```

现在可以看到打印:

> D/SearchPresenter(27693): SearchPresenter(Context context)

从打印来看，@Inject的确是可以选择构造函数的。但还有个细节不知道大家有没有注意到,我们去掉了mContext的@Inject,改由构造函数传入。这个传入构造函数的Context又是怎么来的呢？

答案在SearchPresenterModule里:

```
@Module
public class SearchPresenterModule {
    private SearchActivity mSearchActivity;

    public SearchPresenterModule(SearchActivity view) {
        mSearchActivity = view;
    }

    @Provides
    SearchView provideSearchView() {
        return mSearchActivity;
    }

	// 是它,是它,就是它
    @Provides
    Context provideContext() {
        return mSearchActivity;
    }
}
```

没错SearchPresenterModule.provideContext()这个方法还能创建Context出来给SearchPresenter的构造函数使用！

# Demo地址

可以在[这里](https://github.com/bluesky466/Dagger2Demo)查看完整代码，剩余部分的代码会在下一篇文章里介绍。
