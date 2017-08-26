title: 一步步教你怎样用PowerMock进行Android的BDD行为驱动开发
date: 2017-08-26 17:18:28
tags:
    - 技术相关
    - Android
    - 单元测试
---

很久之前就有听说过mockito和PowerMock的大名了,无奈我司写单元测试的风气不浓,加上一直以来业务繁忙,惰性使我一直没有写单元测试的习惯。

正好现在手头上的是一个全新的项目,可以在初期有时间也有冲动将各种需要的东西都用上。于是这几天就好好学习了一番,感觉PowerMock的确是无比强大。

# 什么是mock

维基百科上是这么写的:

> 在面向对象程序设计中，模拟对象（英语：mock object，也译作模仿对象）是以可控的方式模拟真实对象行为的假的对象。程序员通常创造模拟对象来测试其他对象的行为，很类似汽车设计者使用碰撞测试假人来模拟车辆碰撞中人的动态行为。  
> 在单元测试中，模拟对象可以模拟复杂的、真实的（非模拟）对象的行为， 如果真实的对象无法放入单元测试中，使用模拟对象就很有帮助。

如果我们使用依赖注入的方式编写代码,例如Context通常都是外部传入的:

```
class ClassA{
	public static boolean staticFunc(Context context, int arg) {
		...
	}
}
```

这个方法如果不使用mock object的方法,我们很难脱离安卓环境去编写单元测试,因为Context是系统生成的。

而使用mock技术去模拟一个Context出来,就可以在android studio中编写并且脱离安卓环境运行单元测试了。

# PowerMock

[powermock](https://github.com/powermock/powermock)是一个流行的java mock框架,通过它我们可以很方便的实现模拟对象。它实际上是继承并且拓展了EasyMock、Mockito等其他的流行框架。

在android studio上导入powermock框架很简单,只需要在build.gradle中添加dependencies就好了:

```
dependencies {
	...
    testCompile 'junit:junit:4.12'

    testCompile 'org.powermock:powermock-core:1.6.1'
    testCompile 'org.powermock:powermock-module-junit4:1.6.1'
    testCompile 'org.powermock:powermock-module-junit4-rule:1.6.1'
    testCompile 'org.powermock:powermock-api-mockito:1.6.1'
}
```

这里有个坑点，之前我用的是1.5.6版本的powermock,但是我的junit是4.12版本的,于是在使用@RunWith(PowerMockRunner.class)的时候会报错:

> org.powermock.reflect.exceptions.FieldNotFoundException: Field 'fTestClass' was not found in class org.junit.internal.runners.MethodValidator.

到stackoverflow上搜索到国外大神的[回答](https://stackoverflow.com/questions/26192929/unable-to-run-junit-test-with-powermockrunner)是powermock小于1.6.1的版本在使用junit 4.12的一个bug,在1.6.1被修复。所以要么用junit 4.12 + powermock 1.6.1,要么使用junit 4.11 + powermock 1.5.6.

# mock的简单用法

先说一下最近我拿到的一个需求。我们的应用的按钮点击启动其他应用的响应需要在服务器上配置。服务器上可能配的是包名启动应用,也可能是action启动应用,还有可能是Uri启动应用。

所以我这样写了一个工具类:

```
public class AppUtils {
	public static boolean startApp(Context context, StartAppParam param) {
		...
	}
	
	 public static class StartAppParam {
        private String packageName;
        private String activity;
        private String action;
        private String uri;
        private List<String> categorys = new ArrayList<>();
        ...
    }
}
```

在服务器上配置一个json,传到客户端解析成StartAppParam,然后调用AppUtils. startApp方法。这样就可以实现这个需求了。

我们使用TDD的方式开发这个功能。首先考虑只配置Action的方式启动:

```
@Test
    public void testOpenAppByAction() {
		Context context = Mockito.mock(Context.class);
		
		AppUtils.StartAppParam param = Mockito.mock(AppUtils.StartAppParam.class);
		PowerMockito.when(param.getAction()).thenReturn("package");
		
		assertTrue(AppUtils.startApp(context, param));
		    
		Mockito.verify(context, Mockito.times(1)).startActivity(Matchers.any(Intent.class));
    }
```

首先,使用Mockito.mock方法可以创建一个模拟对象出来。我们这里使用模拟的Context就可以直接在android studio中运行单元测试了。

同时param也用mock的方式创建了出来,而且还模拟了它的getAction方法,让该方法返回"package",表示配置了使用Action去启动应用:

```
AppUtils.StartAppParam param = Mockito.mock(AppUtils.StartAppParam .class);
PowerMockito.when(param.getAction()).thenReturn("package");
```


然后Mockito.verify方法可以用来验证调用了方法的调用次数,比如这里我们就验证了startActivity被调用了一次。

# mock 方法内部创建的对象

当然这个测试不充分,因为我们没有验证到底是不是通过Action启动的。也就是说我们还需要判断是不是通过new Intent(param.getAction())的方式创建了一个Intent出来。

这就用到了PowerMock的一个很屌的功能了,它不仅可以在外部mock一个对象通过参数传给需要测试的方法,更可以直接mock方法内部创建的对象(比如这里的Intent)!

```
@RunWith(PowerMockRunner.class)
public class AppUtilsTest {

    @Test
    @PrepareForTest({AppUtils.class})
    public void testOpenAppByAction() throws Exception {
        Intent intent = Mockito.mock(Intent.class);
        PowerMockito.whenNew(Intent.class).withArguments("package").thenReturn(intent);

        Context context = Mockito.mock(Context.class);

        AppUtils.StartAppParam param = Mockito.mock(AppUtils.StartAppParam.class);
        PowerMockito.when(param.getAction()).thenReturn("package");

        assertTrue(AppUtils.startApp(context, param));
        
        Mockito.verify(context, Mockito.times(1)).startActivity(intent);
        Mockito.verify(intent, Mockito.times(0)).setData(Matchers.any(Uri.class));
        Mockito.verify(intent, Mockito.times(0)).addCategory(Matchers.anyString());
        Mockito.verify(intent, Mockito.times(0)).setClassName(Matchers.anyString(), Matchers.anyString());
    }
}
```

首先需要用@RunWith(PowerMockRunner.class)注解AppUtilsTest类,用@PrepareForTest({AppUtils.class})注解testOpenAppByAction方法,传入的AppUtils.class表示需要在AppUtils类内部实现mock操作。

然后mock一个Intent出来,接着使用下面的方法使得使用new Intent("package")得到的Intent是我们mock出来的intent,注意这里连传入的"package"参数也需要匹配才能得到我们mock出来的intent。否则只能得到null:

```
PowerMockito.whenNew(Intent.class).withArguments("package").thenReturn(intent);
```

所以我们在后面只需要验证startActivity调用的intent是不是我们mock出来的对象,就可以验证是不是通过Action启动的应用了:

```
Mockito.verify(context, Mockito.times(1)).startActivity(intent);
```

当然,为了保险我们可以顺便确认一下Intent的其他方法是不是没有被调用到:

```
Mockito.verify(intent, Mockito.times(0)).setData(Matchers.any(Uri.class));
Mockito.verify(intent, Mockito.times(0)).addCategory(Matchers.anyString());
Mockito.verify(intent, Mockito.times(0)).setClassName(Matchers.anyString(), Matchers.anyString());
```

# 使用BDD的方式编写单元测试

BDD (Behavior-driven development,行为驱动开发)通过用自然语言书写非程序员可读的测试用例扩展了测试驱动开发方法。也就是说用bdd方式写的代码就连不是程序员的人也能看得懂,这种可读性的重要性就不用我多费口舌了吧。

其实Mockito的BDD方式的写法我觉得并不是特别的像自然语言。所以我想用C++的单元测试框架Catch框架来举例:

```
GIVEN("a enable stub publish server entry") {
    StubPublishServerEntry entry(true);
	entry.Start();

    WHEN("publish service") {
        entry.PublishService(service, on_result, on_success, on_error);

        THEN("publish successfully") {
            REQUIRE(service_entry != nullptr);
            REQUIRE(service_entry->IsPublished());
            REQUIRE(is_on_success);
            REQUIRE_FALSE(is_on_error);
        }
    }
}
```

这是我之前的半成品项目中的一个代码片段。如果将代码部分去掉,只留下GIVEN、WHEN、THEN三个宏里面的东西,基本只有是懂英语的人都能看得懂这段代码想做什么:

```
GIVEN("a enable stub publish server entry") {
    ...
    WHEN("publish service") {
        ...
        THEN("publish successfully") {
            ...  
        }
    }
}
```

PowerMock也是支持BDD的(应该说Mockito是支持BDD的),我们可以将上面写的测试用例改成BDD的写法:

```
public void testOpenAppByAction() throws Exception {
    Intent intent = Mockito.mock(Intent.class);
    PowerMockito.whenNew(Intent.class).withArguments("package").thenReturn(intent);

    Context context = Mockito.mock(Context.class);

    AppUtils.StartAppParam param = Mockito.mock(AppUtils.StartAppParam .class);

    //given
    BDDMockito.given(param.getAction()).willReturn("package");

    //when
    assertTrue(AppUtils.startApp(context, param));

    //then
    BDDMockito.then(context).should().startActivity(intent);
    BDDMockito.then(intent).should(Mockito.never()).setData(Matchers.any(Uri.class));
    BDDMockito.then(intent).should(Mockito.never()).addCategory(Matchers.anyString());
    BDDMockito.then(intent).should(Mockito.never()).setClassName(Matchers.anyString(), Matchers.anyString());
}
```

感觉是不是和自然语言还是差别蛮大的,我们改造改造,将一些方法改成通过import static的方式import:

```
public void testOpenAppByAction() throws Exception {
    Intent intent = mock(Intent.class);
    whenNew(Intent.class).withArguments("package").thenReturn(intent);

    Context context = mock(Context.class);

    AppUtils.StartAppParam param = mock(AppUtils.StartAppParam .class);

    //given
    given(param.getAction()).willReturn("package");

    //when
    assertTrue(AppUtils.startApp(context, param));

    //then
    then(context).should().startActivity(intent);
    then(intent).should(never()).setData(any(Uri.class));
    then(intent).should(never()).addCategory(anyString());
    then(intent).should(never()).setClassName(anyString(), anyString());
}
```

这样是不是好多了？让我们继续改造:

```
public class AppUtilsTest {
    @Mock
    private Intent mIntent;

    @Mock
    private Context mContext;

    @Mock
    private AppUtils.StartAppParam mParam;

    @Before
    public void setUp() throws Exception {
        whenNew(Intent.class).withArguments("package").thenReturn(mIntent);
    }

    @Test
    @PrepareForTest({AppUtils.class})
    public void testOpenAppByAction() {
        given(mParam.getAction()).willReturn("package");

        //when
        assertTrue(AppUtils.startApp(mContext, mParam));

        then(mContext).should().startActivity(mIntent);
        then(mIntent).should(never()).setData(any(Uri.class));
        then(mIntent).should(never()).addCategory(anyString());
        then(mIntent).should(never()).setClassName(anyString(), anyString());
    }
}
```

因为Intent、Context、AppUtils.StartAppParam都是需要在不同测试用例中经常被用到的,我们将它写成成员变量并且用@Mock实现自动mock,省去Mockito.mock()方法的调用。

然后将whenNew方法放到由@Before注解的setUp()方法中。

现在看testOpenAppByAction是不是简洁多了？只要有一点代码功底的人都能很容易看明白这个用例到底是用来验证什么的。

当然,这里的BDD写法和上面Catch的写法比起来在像自然语言方面还是有点差距的。

现在我们已经将测试用例写出来了，就可以开始写代码让这个测试用例通过了。像这样先写行为测试用例再写代码的开发方式就叫做BDD。


# mock 静态方法

我们下一个需要实现的功能是什么呢？就实现通过包名启动应用吧。将设只配置了包名,但没有配置Activity名。我们就需要先找到这个应用的Launch Activity,然后再去启动应用。

所以我们在AppUtils中新增了一个方法,用于从包名获取Activity名:

```
public class AppUtils {
	public static boolean startApp(Context context, StartAppParam param) {
		...
	}
	
	public static String getLaunchActivityByPackage(Context context, String packageName) {
		return null;
	}
}
```

如果是正常的开发流程我们需要写一个getLaunchActivityByPackage测试用例，再实现这个方法。这里我就省略了这步,让getLaunchActivityByPackage这个方法先不实现,直接返回null,测试的时候直接mock就好了。

之后我们再去写startAppByPackage的测试用例:

```
@Test
public void startAppByPackage() {
    mockStatic(AppUtils.class);

    given(AppUtils.startApp(any(Context.class), any(AppUtils.StartAppParam.class)))
            .willCallRealMethod();
    given(AppUtils.getLaunchActivityByPackage(any(Context.class), anyString()))
            .willReturn("LauncActivity");
    given(mParam.getPackageName()).willReturn("packageName");

    //when
    assertTrue(AppUtils.startApp(mContext, mParam));

    //then
    verifyStatic(); //开启static方法的验证,需要开启才能验证AppUtils.getLaunchActivityByPackage是否被调用
    AppUtils.getLaunchActivityByPackage(any(Context.class), eq("packageName"));
    then(mIntent).should().setClassName(mParam.getPackageName(), "LauncActivity");
    then(mContext).should().startActivity(mIntent);
}
```

首先我们使用mockStatic去模拟AppUtils,然后配置AppUtils.startApp调用实际的方法,而getLaunchActivityByPackage直接返回"LauncActivity"。

在验证getLaunchActivityByPackage是否被调用的时候要先调用verifyStatic()。

之后再用下面的方式验证是不是调用了AppUtils.getLaunchActivityByPackage并且传入了"packageName"

```
AppUtils.getLaunchActivityByPackage(any(Context.class), eq("packageName"));
```

这里多说一点,假设getLaunchActivityByPackage是一个private的方法,我们可以用下面的方式去mock它:

```
when(AppUtils.class, "getLaunchActivityByPackage", any(Context.class), anyString())
        .thenReturn("LauncActivity");
```


# 完整Demo

其他剩下的测试用例我就不一个一个去讲了,基本上通过之前对PowerMock用法的介绍大家也应该能自己实现了。

完整的demo代码可以从[这里](https://github.com/bluesky466/UnitTestDemo)获取
