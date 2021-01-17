title: Dagger2 AndroidInjector原理探究
date: 2021-01-17 12:55:30
tags:
    - 技术相关
    - Android
---


AndroidInjector在实际的项目中已经频繁使用了，但是其底层的原理一直没有关注，只是简单的使用。这篇笔记就来探究下它的实现原理。

我们要先从Component的依赖开始讲起，然后一步一步往AndroidInjector的原理靠近。

在Dagger2里面@Component.dependencies和@Subcomponent都可以实现Component依赖，而@Subcomponent又可以分成使用和不使用@Module.subcomponents去声明依赖两种。所以算下来应该是三种实现依赖的方式。今天主要想介绍的是@Module.subcomponents的方式，但是其他两种我也先过一下。

# @Component.dependencies实现依赖

@Component.dependencies是一种比较简单的方式，举个简单的例子，MainActivityComponent是子Component，它需要给MainActivity注入AppCommonData和MainActivityData。

但MainActivityModule只提供了MainActivityData，AppCommonData由dependencies指定的AppComponent提供注入:

```kotlin
@Component(modules = [MainActivityModule::class], dependencies = [AppComponent::class])
interface MainActivityComponent {
    fun inject(activity: MainActivity)
}

@Module
class MainActivityModule {
    @Provides
    fun providerMainActivityData(): MainActivityData {
        return MainActivityData()
    }
}

class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var appCommonData: AppCommonData
    @Inject
    lateinit var mainActivityData: MainActivityData
    ...
}
```

由于使用了dependencies的方式，AppComponent需要将AppCommonData暴露出来给MainActivityComponent使用，如果不去暴露的话MainActivityComponent是拿不到的:

```kotlin
@Component(modules = [AppModule::class])
interface AppComponent {
    fun provideAppCommonData(): AppCommonData
}

@Module
class AppModule {
    @Provides
    fun provideAppCommonData(): AppCommonData {
        return AppCommonData()
    }
}
```

当使用dependencies方式的适合会生成DaggerMainActivityComponent，我们需要先创建父Component，然后在创建MainActivityComponent的时候将父Component传入实现依赖的注册:

```kotlin
class DemoApplication : Application() {
    private lateinit var appComponent: AppComponent

    lateinit var mainActivityComponent: MainActivityComponent
        private set

    override fun onCreate() {
        super.onCreate()

        appComponent = DaggerAppComponent.builder().build()

        mainActivityComponent = DaggerMainActivityComponent.builder()
            .appComponent(appComponent)
            .build()
    }
}
```

注入的时候需要找到MainActivityComponent去注入:

```kotlin
class MainActivity : AppCompatActivity() {
    ...
    override fun onCreate(savedInstanceState: Bundle?) {
        ...
        (applicationContext as DemoApplication)
            .mainActivityComponent
            .inject(this)
        ...
    }
}
```

这种方式的优点是父Component不需要知道子Component的存在，缺点是父Component需要显示提供子Component需要注入的对象，一旦需要提供的注入对象比较多的时候写起来就很啰嗦。

[Demo代码](https://github.com/bluesky466/DaggerSubcomponentDemo/tree/feature_dependencies)

# @Subcomponent实现依赖

我们可以使用@Subcomponent去实现依赖，将子Component的注解改成@Subcomponent，dependencies去掉其他不变:

```kotlin
@Subcomponent(modules = [MainActivityModule::class])
interface MainActivityComponent {
    fun inject(activity: MainActivity)
}

@Module
class MainActivityModule {
    @Provides
    fun providerMainActivityData(): MainActivityData {
        return MainActivityData()
    }
}

class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var appCommonData: AppCommonData
    @Inject
    lateinit var mainActivityData: MainActivityData
    ...
}
```

AppComponent不需要直接提供AppCommonData，但是由于使用了Subcomponent之后是不会生成DaggerMainActivityComponent的，它需要AppCommonData去负责创建，所以AppCommonData需要添加DaggerMainActivityComponent的创建方法。这里的主要目的是为了明确依赖关系，因为dagger2不可能知道你想要的依赖关系是怎样的:

```kotlin
@Component(modules = [AppModule::class])
interface AppComponent {
    fun addMainActivityComponent(): MainActivityComponent
}

@Module
class AppModule {
    @Provides
    fun provideAppCommonData(): AppCommonData {
        return AppCommonData()
    }
}

class DemoApplication : Application() {
    private lateinit var appComponent: AppComponent

    lateinit var mainActivityComponent: MainActivityComponent
        private set

    override fun onCreate() {
        super.onCreate()

        appComponent = DaggerAppComponent.builder().build()
        mainActivityComponent = appComponent.addMainActivityComponent()
    }
}
```

MainActivity在使用注入的时候代码是一样的，我这里就不贴了。这种方式的好处是Subcomponent可以直接使用父类提供的所有注入对象，但是每增加一个Subcomponent，父Component都需要新增一个创建方法。

[Demo代码](https://github.com/bluesky466/DaggerSubcomponentDemo/tree/feature_subcomponent)

# @Module.subcomponents实现依赖

除了在父Component添加Subcomponent的创建方法之外，我们还可以在父Component的Module里面去声明依赖关系:

```kotlin
@Module(subcomponents = [MainActivityComponent::class])
class AppModule {
    @Provides
    fun provideAppCommonData(): AppCommonData {
        return AppCommonData()
    }
}
```

这里的@Module.subcomponents相当于Module添加了Subcomponent.Builder的注入能力，我们可以将Subcomponent.Builder注入到任意对象。这里我们注入到DemoApplication，所以AppComponent如下:

```kotlin
@Component(modules = [AppModule::class])
interface AppComponent {
    fun inject(app: DemoApplication)
}
```

既然是注入Subcomponent.Builder，那么就要求我们的Subcomponent要声明Builder:

```kotlin
@Subcomponent(modules = [MainActivityModule::class])
interface MainActivityComponent {
    fun inject(activity: MainActivity)

    @Subcomponent.Builder
    interface Builder {
        fun build(): MainActivityComponent
    }
}

@Module
class MainActivityModule {
    @Provides
    fun providerMainActivityData(): MainActivityData {
        return MainActivityData()
    }
}
```

这个时候创建MainActivityComponent就分成了两步，先创建MainActivityComponent.Builder，再创建MainActivityComponent:

```kotlin
class DemoApplication : Application() {
    @Inject
    lateinit var mainActivityComponentBuilder: MainActivityComponent.Builder

    lateinit var mainActivityComponent: MainActivityComponent

    override fun onCreate() {
        super.onCreate()

        DaggerAppComponent.builder()
            .build()
            .inject(this)
        mainActivityComponent = mainActivityComponentBuilder.builder()
    }
}
```

当然__上面的Subcomponent.Builder改成Subcomponent.Factory__也是可以的。

[Demo代码](https://github.com/bluesky466/DaggerSubcomponentDemo/tree/feature_module_subcomponents)

# @Module.subcomponents实现多绑定

这种方式看上去是麻烦了一些，但是由于Subcomponent.Builder也是dagger2注入的，所以可以用这个特性去让我们的依赖注入做的更加彻底一点。

我们可以看到目前我们的MainActivity实际上是需要知道自己是用哪个Component去注入的，也就是说我们破坏了"一个对象不应该知道它是如何被注入的"这个原则:

```kotlin
class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var appCommonData: AppCommonData

    @Inject
    lateinit var mainActivityData: MainActivityData

    override fun onCreate(savedInstanceState: Bundle?) {
        ...
        (applicationContext as DemoApplication)
            .mainActivityComponent
            .inject(this)
        ...
    }
}
```

当然我们使用重载的方式将Component的选择挪到Application中，但是这样依然是在Dagger2框架之外去决定依赖关系。那可不可以做的更彻底些，将MainActivity和Component的依赖关系也使用Dagger2框架去决定呢?当然是可以的，就是使用@Module.subcomponents。

首先我们定义一个BaseComponent作为基类，然后MainActivityComponent和SecondActivityComponent都继承它

```kotlin
interface BaseComponent<T> {
    fun inject(target: T)

    interface Builder<T> {
        fun build(): BaseComponent<T>
    }
}

@Subcomponent(modules = [MainActivityModule::class])
interface MainActivityComponent : BaseComponent<MainActivity> {
    @Subcomponent.Builder
    interface Builder : BaseComponent.Builder<MainActivity>
}

@Subcomponent(modules = [SecondActivityModule::class])
interface SecondActivityComponent : BaseComponent<SecondActivity> {
    @Subcomponent.Builder
    interface Builder : BaseComponent.Builder<SecondActivity>
}
```

然后定义Activity与Subcomponent.Builder之间的关联:

```kotlin
@Module(subcomponents = [MainActivityComponent::class, SecondActivityComponent::class])
interface ComponentBindingModule {
    @Binds
    @IntoMap
    @ClassKey(MainActivity::class)
    fun mainActivityComponentBuilder(builder: MainActivityComponent.Builder): BaseComponent.Builder<*>


    @Binds
    @IntoMap
    @ClassKey(SecondActivity::class)
    fun secondActivityComponentBuilder(builder: SecondActivityComponent.Builder): BaseComponent.Builder<*>
}

@Component(modules = [AppModule::class, ComponentBindingModule::class])
interface AppComponent {
    fun inject(app: DemoApplication)
}
```

ComponentBindingModule将MainActivityComponent.Builder和SecondActivityComponent.Builder用对应的Activity的class为key放入Map中，所以Activity在需要注入的时候可以用自己的class为key获取到Subcomponent.Builder然后进行注入:

```kotlin
class DemoApplication : Application() {
    @Inject
    lateinit var componentMap: MutableMap<Class<*>, BaseComponent.Builder<*>>

    @Inject
    lateinit var secondActivityComponentBuilder: SecondActivityComponent.Builder

    override fun onCreate() {
        super.onCreate()
        DaggerAppComponent.builder()
            .build()
            .inject(this)
    }

    fun <T : Any> inject(target: T) {
        (componentMap[target.javaClass]?.build() as BaseComponent<T>).inject(target)
    }
}
```

于是乎Activity的注入就变得很简单了:

```kotlin
class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var appCommonData: AppCommonData
    @Inject
    lateinit var mainActivityData: MainActivityData

    override fun onCreate(savedInstanceState: Bundle?) {
        ...
        (applicationContext as DemoApplication).inject(this)
        ...
    }
}

class SecondActivity : AppCompatActivity() {
    @Inject
    lateinit var appCommonData: AppCommonData
    @Inject
    lateinit var secondActivityData: SecondActivityData

    override fun onCreate(savedInstanceState: Bundle?) {
        ...
        (applicationContext as DemoApplication).inject(this)
        ...
    }
}
```

我们还可以为其编写BaseActivity去进行注入。

[Demo代码](https://github.com/bluesky466/DaggerSubcomponentDemo/tree/master)

# AndroidInjector的原理

[AndroidInjector](https://dagger.dev/dev-guide/android)的就是基于上面的技术实现的。我们可以看到DaggerApplication里面注入了一个DispatchingAndroidInjector\<Object\>，Activity在onCreate的时候就可以拿到这个东西去进行注入:

```kotlin
public abstract class DaggerApplication extends Application implements HasAndroidInjector {
  @Inject volatile DispatchingAndroidInjector<Object> androidInjector;
  ...
  @Override
  public AndroidInjector<Object> androidInjector() {
    ...
    return androidInjector;
  }
  ...
}
```

那我们来看看DispatchingAndroidInjector其实内部类似的也有两个Map用于获取不同的类对应的AndroidInjector.Factory:

```kotlin
public final class DispatchingAndroidInjector<T> implements AndroidInjector<T> {
    ...
    private final Map<String, Provider<AndroidInjector.Factory<?>>> injectorFactories;
    ...
    @Inject
    DispatchingAndroidInjector(
        Map<Class<?>, Provider<AndroidInjector.Factory<?>>> injectorFactoriesWithClassKeys,
        Map<String, Provider<AndroidInjector.Factory<?>>> injectorFactoriesWithStringKeys) {
      this.injectorFactories = merge(injectorFactoriesWithClassKeys, injectorFactoriesWithStringKeys);
    }
    ...
    @Override
    public void inject(T instance) {
        boolean wasInjected = maybeInject(instance);
        ...
    }
    ...
    public boolean maybeInject(T instance) {
        Provider<AndroidInjector.Factory<?>> factoryProvider = injectorFactories.get(instance.getClass().getName());
        ...
        AndroidInjector.Factory<T> factory = (AndroidInjector.Factory<T>) factoryProvider.get();
        try {
              AndroidInjector<T> injector = checkNotNull(factory.create(instance), "%s.create(I) should not return null.", factory.getClass());
              injector.inject(instance);
              return true;
        } 
        ...
  }
}
```

于是乎在inject的时候就能通过类名找到这个AndroidInjector.Factory进而创建AndroidInjector去进行注入。

 Map\<Class\<?\>, Provider\<AndroidInjector.Factory\<?\>\>\> 和Map\<String, Provider\<AndroidInjector.Factory\<?\>\>\>是通过AndroidInjectionModule注入的:

```java
@Module
public abstract class AndroidInjectionModule {
  @Multibinds
  abstract Map<Class<?>, AndroidInjector.Factory<?>> classKeyedInjectorFactories();

  @Multibinds
  abstract Map<String, AndroidInjector.Factory<?>> stringKeyedInjectorFactories();

  private AndroidInjectionModule() {}
}
```

所以AppComponent里面需要添加这个module依赖:

```kotlin
@Component(modules = [AndroidInjectionModule::class, AppModule::class, ComponentBindingModule::class])
interface AppComponent : AndroidInjector<DemoApplication>
```

而Map里面找到的都是AndroidInjector.Factory，所以我们的Subcomponent也应该继承AndroidInjector:

```kotlin
@Subcomponent(modules = [MainActivityModule::class])
interface MainActivityComponent : AndroidInjector<MainActivity> {
    @Subcomponent.Factory
    interface Factory : AndroidInjector.Factory<MainActivity>
}

@Subcomponent(modules = [SecondActivityModule::class])
interface SecondActivityComponent : AndroidInjector<SecondActivity> {
    @Subcomponent.Factory
    interface Factory : AndroidInjector.Factory<SecondActivity>
}

```

ComponentBindingModule里面依然是将AndroidInjector.Factory用对应的Activity的class为key放入Map中:

```kotlin
@Module(subcomponents = [MainActivityComponent::class, SecondActivityComponent::class])
interface ComponentBindingModule {
    @Binds
    @IntoMap
    @ClassKey(MainActivity::class)
    fun mainActivityComponentFactory(factory: MainActivityComponent.Factory): AndroidInjector.Factory<*>


    @Binds
    @IntoMap
    @ClassKey(SecondActivity::class)
    fun secondActivityComponentFactory(factory: SecondActivityComponent.Factory): AndroidInjector.Factory<*>
}
```

Activity在注入的时候只需要使用AndroidInjection.inject(this)即可:

```kotlin
class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var appCommonData: AppCommonData
    @Inject
    lateinit var mainActivityData: MainActivityData

    override fun onCreate(savedInstanceState: Bundle?) {
        AndroidInjection.inject(this)
        super.onCreate(savedInstanceState)
        ...
    }
}
```

[Demo代码](https://github.com/bluesky466/DaggerSubcomponentDemo/tree/feature_android_injection_principle)

## ContributesAndroidInjector

上面的MainActivityComponent、SecondActivityComponent的代码还有IntoMap的代码其实比较呆板，我们可以通过ContributesAndroidInjector去优化。所以ComponentBindingModule可以改成下面这个样子,然后:

```kotlin
@Module
abstract class ComponentBindingModule {
    @ContributesAndroidInjector(modules = [MainActivityModule::class])
    abstract fun contributesMainActivity(): MainActivity
    ...
}
```

它会帮我们生成Subcomponent、IntoMap代码:

```kotlin
//下面的代码可以省略，被ContributesAndroidInjector代替了
@Module
class MainActivityModule {
    @Provides
    fun providerMainActivityData(): MainActivityData {
        return MainActivityData()
    }
}

@Binds
@IntoMap
@ClassKey(MainActivity::class)
fun mainActivityComponentFactory(factory: MainActivityComponent.Factory): AndroidInjector.Factory<*>
```

[Demo代码](https://github.com/bluesky466/DaggerSubcomponentDemo/tree/feature_android_injection)

# 参考文章

[Dagger & Android](https://dagger.dev/dev-guide/android)

[Multibindings](https://dagger.dev/dev-guide/multibindings)

[在Dagger 2中Activities和Subcomponents的多绑定](https://www.cnblogs.com/tiantianbyconan/p/6266442.html )

