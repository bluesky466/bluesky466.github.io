title: Java自定义注解和动态代理
date: 2016-05-27 19:49:05
tags:
	- java
	- 技术相关
---
在学习Retrofit的时候就对它奇特的使用方式感到十分的好奇，为什么定义一个接口，使用"@GET","@Query"这些奇怪的注解就能创建出能实际访问服务器的实例出来:

```java
public interface GitHubService {
    @GET("/users/{user}/repos")
    List<Repo> listRepos(@Path("user") String user);
}

RestAdapter restAdapter = new RestAdapter.Builder()
    .setEndpoint("https://api.github.com")
    .build();
GitHubService service = restAdapter.create(GitHubService.class);
List<Repo> repos = service.listRepos("octocat");
```

尤其是对如何创建一个接口的实例感到万分的好奇，这几天回学校写毕设论文刚好有点空，于是就抽了点时间研究了一下。

# 自定义Java注解

说起Java的注解，大家都能很自然的想起"@Override","@Deprecated","@Documented"这些很常用的内置注解。但很多新手应该都不知道其实Java也是支持自定义注解的吧？(反正我以前是不知道的)

## 元注解
元注解的作用就是负责注解其他注解，Java5.0定义了4个标准的meta-annotation类型，它们被用来提供对其它 annotation类型作说明。Java5.0定义的元注解：

- @Target
- @Retention
- @Documented
- @Inherited

### @Target

@Target说明了自定义注解的修饰类型，也就是说可以用它来声明自定义注解可以用在什么地方，它的取值范围有：

1. ElementType.CONSTRUCTOR : 用于描述构造器
1. ElementType.FIELD : 用于描述域
1. ElementType.LOCAL_VARIABLE : 用于描述局部变量
1. ElementType.METHOD : 用于描述方法
1. ElementType.PACKAGE : 用于描述包
1. ElementType.PARAMETER : 用于描述参数
1. ElementType.TYPE : 用于描述类、接口(包括注解类型) 或enum声明

例如：

```java
@Target(ElementType.TYPE)
public @interface TypeAnnotation {
}

@Target(ElementType.METHOD)
public @interface MethodAnnotation {
}

@Target(ElementType.PARAMETER)
public @interface ParamAnnotation {
}
```

TypeAnnotation可以用来修饰描述类、接口(包括注解类型) 或enum声明，MethodAnnotation可以用来修饰方法，ParamAnnotation可以用来修饰参数：

```java
@TypeAnnotation
public interface MyInterface {
	@MethodAnnotation
	public void func(@ParamAnnotation String param);
}
```

## @Retention

@Retention 定义了自定义注解的生命长短：某些Annotation仅出现在源代码中，而被编译器丢弃；而另一些却被编译在class文件中；编译在class文件中的Annotation可能会被虚拟机忽略，而另一些在class被装载时将被读取（请注意并不影响class的执行，因为Annotation与class在使用上是被分离的）。使用这个meta-Annotation可以对 Annotation的“生命周期”限制。

它的取值有下面这些：

1. RetentionPolicy.SOURCE : 在源文件中有效（即源文件保留）
1. RetentionPolicy.CLASS : 在class文件中有效（即class保留）
1. RetentionPolicy.RUNTIME : 时有效（即运行时保留）

注解也是可以保存数据的，如value属性就是默认的数据保存属性:

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MethodAnnotation {
	String value() default "default value";
}

public class ClassA{
	@MethodAnnotation()
	public void func1(){}
    
	@MethodAnnotation("data")
	public void func2(){}
}

```

比如可以在程序运行的时候（因为声明了@Retention(RetentionPolicy.RUNTIME)）通过反射获取上面的ClassA.func1的注解MethodAnnotation保存的数据（默认值"default value"）和ClassA.func2的注解MethodAnnotation保存的数据（"data"）

当然如果只能保持一个数据限制就太大了，你可以定义多个数据：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MethodAnnotation {
	public enum EnumData{ DATA1, DATA2, DATA3};
	
	String data1();
	int data2() default 0xffff;
	EnumData data3() default EnumData.DATA1;
}

public class ClassA{
	@MethodAnnotation(data1="data1")
	public void func1(){}
    
	@MethodAnnotation(data1="data1", data2=0, data3=MethodAnnotation.EnumData.DATA1)
	public void func2(){}
}
```

当属性没有用default指定默认值得时候在使用的时候必须由用户设置属性值（如这里的data1）

注解参数的可支持数据类型：

1.所有基本数据类型（int,float,boolean,byte,double,char,long,short)
2.String类型
3.Class类型
4.enum类型
5.Annotation类型
6.以上所有类型的数组


## @Inherited

@Inherited 元注解是一个标记注解，@Inherited阐述了某个被标注的类型是被继承的。如果一个使用了@Inherited修饰的annotation类型被用于一个class，则这个annotation将被用于该class的子类。

# Java动态代理

我知道可以用反射调用方法甚至创建对象，但我还真的没有想到怎样创建出一个接口的实例。用了各种形容方式之后终于找到了这种技术的专业名称:"动态代理"，下面是一个简单的例子：

```java
public interface MyInterface{
	void func();
}

public class MyHandler implements InvocationHandler{

	@Override
	public Object invoke(Object proxy, Method method, Object[] args)
			throws Throwable {
		// TODO Auto-generated method stub
		System.out.print("call : " + method.getName());
		return null;
	}
}


public static void main(String[] args){
		MyInterface myInterface = (MyInterface)Proxy.newProxyInstance(
        		 MyInterface.class.getClassLoader(), 
			     new Class[]{MyInterface.class}, 
			     new MyHandler());
		myInterface.func();
}
```

这个例子的输出为：

> call : func

通过实现InvocationHandler接口，定义自己的handler类，再使用Proxy.newProxyInstance就可以实例化出一个接口的实例。当调用接口的方法的时候，InvocationHandler接口的invoke方法就会被调用，可以在这里编写实际的功能代码。

当然也能将创建实例的代码抽象出来，实现复用：

```java
private static <T>T newProxyInstance(Class<T> c){
		return (T)Proxy.newProxyInstance(
				c.getClassLoader(),
				new Class<?>[]{c},
				new DataBaseHalder(c));
	}
    
public static void main(String[] args){
    //使用方式
    MyInterface myInterface = (MyInterface)newProxyInstance(MyInterface.class, new MyHandler());
	myInterface.func();
}
```

# 通过反射获取注解保存的数据
- 通过Class.getAnnotation(XXXAnnotation.class)可以获取到方法的ElementType.METHOD或者类ElementType.TYPE类型的注解
- 通过 Method.getParameterAnnotations()可以获取到方法各个参数的注解（ElementType.PARAMETER类型）

这部分用代码来解释最直接了：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface TypeAnnotation {
	String value();
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MethodAnnotation {
	String value();
}

@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface ParamAnnotation {
	String value();
}

@TypeAnnotation("interface MyInterface")
public interface MyInterface{
	@MethodAnnotation("MyInterface.func")
	void func(@ParamAnnotation("param1") String a, @ParamAnnotation("param2") int b);
}

public class MyHandler implements InvocationHandler{
	private Class mClass;
	
	MyHandler(Class c){
		mClass = c;
		TypeAnnotation typeAnnotation = (TypeAnnotation) mClass.getAnnotation(TypeAnnotation.class);
		System.out.print("TypeAnnotation : " + typeAnnotation.value() + "\n");
	}
	@Override
	public Object invoke(Object proxy, Method method, Object[] args)
			throws Throwable {
		
		MethodAnnotation methodAnnotation = (MethodAnnotation)method.getAnnotation(MethodAnnotation.class);
		System.out.print("MethodAnnotation : " + methodAnnotation.value() + "\n");

		Class[] parameterTypes = method.getParameterTypes();
		Annotation[][] parameterAnnotations = method.getParameterAnnotations();
		for(int i = 0; i< parameterAnnotations.length ; i++){
			Class type = parameterTypes[i];
			for(Annotation annotation : parameterAnnotations[i]){
				if(annotation instanceof ParamAnnotation){
					ParamAnnotation paramAnnotation = (ParamAnnotation) annotation;
					System.out.print(paramAnnotation.value() + " - " + args[i]  + "[" + type.getName() + "]"+ "\n");
				}
			}
		}
		return null;
	}
}

//调用
public static void main(String[] args){
		MyInterface myInterface = (MyInterface)Proxy.newProxyInstance(
				MyInterface.class.getClassLoader(), 
				new Class<?>[]{MyInterface.class},
				new MyHandler(MyInterface.class));
		myInterface.func("data1",123);
}
```

输出如下：

> TypeAnnotation : interface MyInterface
> MethodAnnotation : MyInterface.func
> param1 - data1[java.lang.String]
> param2 - 123[int]

这里最难理解的就是这两行代码：

```java
Class[] parameterTypes = method.getParameterTypes();
Annotation[][] parameterAnnotations = method.getParameterAnnotations();
```

method.getParameterTypes() 可以获取到参数的类型，而method.getParameterAnnotations()则获取到一个二维数组，它保存了所有变量的全部注解。

# 一个简单的应用实例

用过Retrofit的人都知道，这种动态代理技术在框架搭建完成之后，使用起来便十分的便利了，有兴趣的同学可以去看看Retrofit的相关资料。我这里再写一个模拟操作数据库的小例子，展示一下这种框架的便捷性。

首先是接口的定义：

```java
@DataBase(database="SchoolSystem", username="root", password="123456", ip="localhost")
public interface IDataBaseOperation {
	@Table("Student")
	@Column({"name","age","sex"})
	List<Map<String,String>> getStudentByName(@Condition("name")String name);
	
	@Table("Student")
	@Column({"name","age","sex"})
	List<Map<String,String>> getStudentOlder(@Condition(value="age",compare=">")int age);

	@Table("Student")
	@Column({"name","age","sex"})
	List<Map<String,String>> getStudentBySexAndAge(@Condition("sex")String sex, @Condition("age")int age);
	
	@Table("Teacher join Course on Teacher.id=Course.teacher")
	@Column({"Course.name"})
	List<Map<String,String>> getCourseByTeacher(@Condition("Teacher.name")String teacher);
}
```

让我们先跳过实现细节，直接看它的用法：

```java
public class Main {
	public static void main(String[] args){
		IDataBaseOperation oprBaseOperation = newProxyInstance(IDataBaseOperation.class, new DataBaseHalder(IDataBaseOperation.class));

		oprBaseOperation.getStudentByName("小红");
		oprBaseOperation.getStudentOlder(12);
		oprBaseOperation.getStudentBySexAndAge("男", 12);
		oprBaseOperation.getCourseByTeacher("李老师");
	}
	
	private static <T>T newProxyInstance(Class<T> c, InvocationHandler handler){
		return (T)Proxy.newProxyInstance(
				c.getClassLoader(), 
				new Class<?>[]{c},
				handler);
	}
}
```

这样我们就能看到这样的输出:

> ===================================
> 
> connect database : 
> ip : localhost
> username : root
> password : 123456
> database : SchoolSystem
>
>===================================
> select name,age,sex from Student where name = "小红";
> select name,age,sex from Student where age > 12;
> select name,age,sex from Student where sex = "男" and age = 12;
> select Course.name from Teacher join Course on Teacher.id=Course.teacher where > > > Teacher.name = "李老师";

这里没有真的去做数据库操作，只是用打印的方法模拟了一下，但如果真的要实现的话也是不难的。

但从这几处使用代码来看，这个框架的是十分易用的，如果我们想要增加一个查询操作的话，只需要在IDataBaseOperation接口声明多一个方法，然后直接就能在得到实例后使用了。

最后将一些细节代码也贴上来：

```java
//Column.java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Column {
	String[] value();
}
```

```java
//Condition.java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface Condition {
	public String value();
	public String compare() default "=";
}
```

```java
//Table.java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Table {
	String value();
}
```

```java
//DataBase.java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface DataBase {
	String ip();
	String database();
	String username();
	String password();
}
```

```java
//DataBaseHalder.java
public class DataBaseHalder implements InvocationHandler{
	private Class mInterface;
	
	public DataBaseHalder(Class object){
		mInterface = object;
		
		DataBase db = (DataBase) mInterface.getAnnotation(DataBase.class);

		System.out.print("===================================\n");
		System.out.print("connect database : \n");
		System.out.print("ip : " + db.ip() + "\n");
		System.out.print("username : " + db.username() + "\n");
		System.out.print("password : " + db.password() + "\n");
		System.out.print("database : " + db.database() + "\n");
		System.out.print("===================================\n");
		
	}

	@Override
	public Object invoke(Object proxy, Method method, Object[] args)
			throws Throwable {
		
		if(method.getName().matches("^get.*")){
			String sql =  "select " + getColumns(method) + " from " + getTable(method) 
					+ " where " + getCondition(method, args) + ";";
			System.out.print(sql + "\n");
		}
		
		return null;
	}
	
	private String getTable(Method method){
		return method.getAnnotation(Table.class).value();
	}
	
	private String getColumns(Method method){
		String result = "";
		String conn = "";
		for (String col : method.getAnnotation(Column.class).value()) {
			result += conn + col;
			conn = ",";
		}
		return result;
	}
	
	private String getCondition(Method method, Object[] args) {
		String result = "";
		String andConnect = "";
		Class[] parameterTypes = method.getParameterTypes();

		Annotation[][] parameterAnnotations = method.getParameterAnnotations();
		for(int i = 0; i< parameterAnnotations.length ; i++){
			Class type = parameterTypes[i];
			for(Annotation annotation : parameterAnnotations[i]){
				if(annotation instanceof Condition){
					result += andConnect + parseCondition(type, args[i], (Condition) annotation);
					andConnect = " and ";
				}
			}
		}
		return result;
	}
	
	private String parseCondition(Class argType, Object arg, Annotation annotation){
		Condition condition = (Condition) annotation;
		String result = condition.value() + " " + condition.compare() + " ";
		if(argType == String.class){
			result += "\"" + arg + "\"";
		}else{
			result += arg;
		}
		return result;
	}
}
```

# 另外的一个实用的小例子

相信做安卓的同学都遇到过在Activity.onCreate初始化的时候写一大堆的findViewById吧？这种重复性的无趣工作其实也可以用注解来简化：

```java
public @interface ViewField {
    int value();

    public static class Processor{
        public static void process(Activity activity) throws IllegalAccessException {
            Field[] fields = activity.getClass().getDeclaredFields();
            ViewField ann = null;
            for (Field field : fields) {
                ann = field.getAnnotation(ViewField.class);
                if (ann!=null){
                    field.setAccessible(true);
                    field.set(activity, activity.findViewById(ann.value()));
                }
            }
        }
    }
}
```

然后我们的Activity就可以这样写来让注解自动初始化View变量了：

```java
public class MainActivity extends AppCompatActivity {
    @ViewField(R.id.text)
    private TextView mTextView;

    @ViewField(R.id.button)
    private Button mButton;

    @ViewField(R.id.image)
    private ImageView mImageView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        try {
            ViewField.Processor.process(this);
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }

        mTextView.setText("text");
        mButton.setText("button");
        mImageView.setImageResource(R.mipmap.ic_launcher);
    }
}
```

当然有人说用注解，效率会很低。但我觉得这里的额外消耗其实根本不起眼，用这点小损耗换来编码的便利性是很值得的。但如果真的很在意，也能用下面的泛型方法简化findViewById操作：

```java
protected <T extends View> T generateFindViewById(int id) {
	//return返回view时,加上泛型T
	return (T) findViewById(id);
}
mButton = generateFindViewById(R.id.button);
```

这样能减少强制转换的操作，但编写效率还是不如用注解。