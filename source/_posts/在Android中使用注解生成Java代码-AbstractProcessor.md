title: 在Android中使用注解生成Java代码 AbstractProcessor
date: 2017-06-03 06:51:26
tags:
    - 技术相关
    - Android
---
前段时间在学习Dagger2,对它生成代码的原理充满了好奇。google了之后发现原来java原生就是支持代码生成的。

通过Annotation Processor可以在编译的时候处理注解,生成我们自定义的代码,这些生成的代码会和其他手写的代码一样被javac编译。注意Annotation Processor只能用来生成代码,而不能对原来的代码进行修改。

实现的原理是通过继承AbstractProcessor,实现我们自己的Processor,然后把它注册给java编译器,编译器在编译之前使用我们定义的Processor去处理注解。

# AbstractProcessor

AbstractProcessor是一个抽象类,我们继承它需要实现一个抽象方法process,在这个方法里面去处理注解。然后它还有几个方法需要我们去重写。

```
public class MyProcessor extends AbstractProcessor {
    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {...}
    
    @Override
    public Set<String> getSupportedAnnotationTypes() {...}
    
    
    @Override
    public SourceVersion getSupportedSourceVersion() {...}
    
    
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {...}
}
```

- init方法是初始化的地方,我们可以通过ProcessingEnvironment获取到很多有用的工具类

- getSupportedAnnotationTypes 这个方法指定处理的注解,需要将要处理的注解的全名放到Set中返回

- getSupportedSourceVersion 这个方法用来指定支持的java版本

- process 是实际处理注解的地方

在Java 7后多了 SupportedAnnotationTypes 和 SupportedSourceVersion 这个两个注解用来简化指定注解和java版本的操作:

```
@SupportedAnnotationTypes({"linjw.demo.injector.InjectView"})
@SupportedSourceVersion(SourceVersion.RELEASE_7)
public class InjectorProcessor extends AbstractProcessor {
    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {...}
        
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {...}
```

# 注册Processor

编写完我们的Processor之后需要将它注册给java编译器

1. 在src/main目录下创建resources/META-INF/services/javax.annotation.processing.Processor文件(即创建resources目录,在resources目录下创建META-INF目录,继续在META-INF目录下创建services目录,最后在services目录下创建javax.annotation.processing.Processor文件)。

2. 在javax.annotation.processing.Processor中写入自定义的Processor的全名,如果有多个Processor的话,每一行写一个。

完成后 javax.annotation.processing.Processor 内容如下

>$ cat javax.annotation.processing.Processor  
>linjw.demo.injector.InjectorProcessor

# 在安卓中自定义Processor

我以前在学习Java自定义注解的时候写过一个[小例子](http://blog.islinjw.cn/2016/05/27/Java%E8%87%AA%E5%AE%9A%E4%B9%89%E6%B3%A8%E8%A7%A3%E5%92%8C%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86/#u53E6_u5916_u7684_u4E00_u4E2A_u5B9E_u7528_u7684_u5C0F_u4F8B_u5B50),它是用运行时注解通过反射简化findViewById操作的。但是这种使用运行时注解的方法在效率上是有缺陷的,因为反射的效率很低。

基本上学安卓的人都知道有个很火的开源库[ButterKnife](https://github.com/JakeWharton/butterknife),它也能简化findViewById操作,但它是通过编译时注解生成代码去实现的,效率比我们使用反射实现要高很多很多。

其实我对ButterKnife的原理也一直很好奇,下面就让我们也用生成代码的方式高效的简化findViewById操作。

## 创建配置工程

首先在android项目中是找不到AbstractProcessor的,需要新建一个Java Library Module。

Android Studio中按File -> New -> New Module... 然后选择新建Java Library, Module的名字改为libinjector。

同时在安卓中使用AbstractProcessor需要apt的支持,所以需要配置一下gradle:

1.在 project 的 build.gradle 的 dependencies 下加上 android-apt 支持

```
...
dependencies {
        classpath 'com.android.tools.build:gradle:2.2.2'
        classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
}
...
```

2.在 app 的 build.gradle 的开头加上 "apply plugin: 'com.neenbedankt.android-apt'"

```
apply plugin: 'com.android.application'
apply plugin: 'com.neenbedankt.android-apt'
...
```

## 创建注解

我们在libinjector中创建注解InjectView

```
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.SOURCE)
public @interface InjectView {
    int value();
}
```

这个是个修饰Field且作用于源码的自定义注解。关于自定义注解的知识可以看看我以前写的一篇文章[《Java自定义注解和动态代理》](http://blog.islinjw.cn/2016/05/27/Java%E8%87%AA%E5%AE%9A%E4%B9%89%E6%B3%A8%E8%A7%A3%E5%92%8C%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86/)。我们用它来修饰View成员变量并保持View的resource id,生成的代码通过resource id使用findViewById注入成员变量。

## 创建InjectorProcessor

在libinjector中创建InjectorProcessor实现代码的生成

```
@SupportedAnnotationTypes({"linjw.demo.injector.InjectView"})
@SupportedSourceVersion(SourceVersion.RELEASE_7)
public class InjectorProcessor extends AbstractProcessor {
    private static final String GEN_CLASS_SUFFIX = "Injector";
    private static final String INJECTOR_NAME = "ViewInjector";

    private Types mTypeUtils;
    private Elements mElementUtils;
    private Filer mFiler;
    private Messager mMessager;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);

        mTypeUtils = processingEnv.getTypeUtils();
        mElementUtils = processingEnv.getElementUtils();
        mFiler = processingEnv.getFiler();
        mMessager = processingEnv.getMessager();
    }

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        Set<? extends Element> elements = roundEnvironment.getElementsAnnotatedWith(InjectView.class);

        //process会被调用三次，只有一次是可以处理InjectView注解的，原因不明
        if (elements.size() == 0) {
            return true;
        }

        Map<Element, List<Element>> elementMap = new HashMap<>();

        StringBuffer buffer = new StringBuffer();
        buffer.append("package linjw.demo.injector;\n")
                .append("public class " + INJECTOR_NAME + " {\n");

        //遍历所有被InjectView注释的元素
        for (Element element : elements) {
            //如果标注的对象不是FIELD则报错,这个错误其实不会发生因为InjectView的Target已经声明为ElementType.FIELD了
            if (element.getKind()!= ElementKind.FIELD) {
                mMessager.printMessage(Diagnostic.Kind.ERROR, "is not a FIELD", element);
            }

            //这里可以先将element转换为VariableElement,但我们这里不需要
            //VariableElement variableElement = (VariableElement) element;

            //如果不是View的子类则报错
            if (!isView(element.asType())){
                mMessager.printMessage(Diagnostic.Kind.ERROR, "is not a View", element);
            }

            //获取所在类的信息
            Element clazz = element.getEnclosingElement();

            //按类存入map中
            addElement(elementMap, clazz, element);
        }

        for (Map.Entry<Element, List<Element>> entry : elementMap.entrySet()) {
            Element clazz = entry.getKey();

            //获取类名
            String className = clazz.getSimpleName().toString();

            //获取所在的包名
            String packageName = mElementUtils.getPackageOf(clazz).asType().toString();

            //生成注入代码
            generateInjectorCode(packageName, className, entry.getValue());

            //完整类名
            String fullName = clazz.asType().toString();

            buffer.append("\tpublic static void inject(" + fullName + " arg) {\n")
                    .append("\t\t" + fullName + GEN_CLASS_SUFFIX + ".inject(arg);\n")
                    .append("\t}\n");
        }

        buffer.append("}");

        generateCode(INJECTOR_NAME, buffer.toString());

        return true;
    }

    //递归判断android.view.View是不是其父类
    private boolean isView(TypeMirror type) {
        List<? extends TypeMirror> supers = mTypeUtils.directSupertypes(type);
        if (supers.size() == 0) {
            return false;
        }
        for (TypeMirror superType : supers) {
            if (superType.toString().equals("android.view.View") || isView(superType)) {
                return true;
            }
        }
        return false;
    }

    private void addElement(Map<Element, List<Element>> map, Element clazz, Element field) {
        List<Element> list = map.get(clazz);
        if (list == null) {
            list = new ArrayList<>();
            map.put(clazz, list);
        }
        list.add(field);
    }

    private void generateCode(String className, String code) {
        try {
            JavaFileObject file = mFiler.createSourceFile(className);
            Writer writer = file.openWriter();
            writer.write(code);
            writer.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 生成注入代码
     *
     * @param packageName 包名
     * @param className   类名
     * @param views       需要注入的成员变量
     */
    private void generateInjectorCode(String packageName, String className, List<Element> views) {
        StringBuilder builder = new StringBuilder();
        builder.append("package " + packageName + ";\n\n")
                .append("public class " + className + GEN_CLASS_SUFFIX + " {\n")
                .append("\tpublic static void inject(" + className + " arg) {\n");

        for (Element element : views) {
            //获取变量类型
            String type = element.asType().toString();

            //获取变量名
            String name = element.getSimpleName().toString();

            //id
            int resourceId = element.getAnnotation(InjectView.class).value();

            builder.append("\t\targ." + name + "=(" + type + ")arg.findViewById(" + resourceId + ");\n");
        }

        builder.append("\t}\n")
                .append("}");

        //生成代码
        generateCode(className + GEN_CLASS_SUFFIX, builder.toString());
    }
}
```

## 注册InjectorProcessor

在libinjector的src/main目录下创建resources/META-INF/services/javax.annotation.processing.Processor文件注册InjectorProcessor:

```
# 注册InjectorProcessor
linjw.demo.injector.InjectorProcessor
```

## 使用InjectView注解

我们在Activity中使用InjectView修饰需要赋值的View变量并且用ViewInjector.inject(this);调用生成的掉初始化修饰的成员变量。这里有两个Activity都使用了InjectView去简化findViewById操作:

```
public class MainActivity extends AppCompatActivity {
    @InjectView(R.id.label)
    TextView mLabel;

    @InjectView(R.id.button)
    Button mButton;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        //使用findViewById注入被InjectView修饰的成员变量
        ViewInjector.inject(this);

        // ViewInjector.inject(this) 已经将mLabel和mButton赋值了,可以直接使用
        mLabel.setText("MainActivity");

        mButton.setText("jump to SecondActivity");
        mButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent  = new Intent(MainActivity.this, SecondActivity.class);
                startActivity(intent);
            }
        });
    }
}
```

```
public class SecondActivity extends Activity {
    @InjectView(R.id.label)
    TextView mLabel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);

        //使用findViewById注入被InjectView修饰的成员变量
        ViewInjector.inject(this);

        // ViewInjector.inject(this) 已经将mLabel赋值了,可以直接使用
        mLabel.setText("SecondActivity");
    }
}
```

# 工具类

在 AbstractProcessor.init 方法中我们可以获得几个很有用的工具类:

```
mTypeUtils = processingEnv.getTypeUtils();
mElementUtils = processingEnv.getElementUtils();
mFiler = processingEnv.getFiler();
mMessager = processingEnv.getMessager();
```

它们的作用如下:

## Types 

Types提供了和类型相关的一些操作，如获取父类、判断两个类是不是父子关系等，我们在isView中就用它去获取父类

```
    //递归判断android.view.View是不是其父类   
    private boolean isView(TypeMirror type) {
        List<? extends TypeMirror> supers = mTypeUtils.directSupertypes(type);
        if (supers.size() == 0) {
            return false;
        }
        for (TypeMirror superType : supers) {
            if (superType.toString().equals("android.view.View") || isView(superType)) {
                return true;
            }
        }
        return false;
    }
```

## Elements

Elements提供了一些和元素相关的操作，如获取所在包的包名等:

```
//获取所在的包名
String packageName = mElementUtils.getPackageOf(clazz).asType().toString();
```

## Filer

Filer用于文件操作,我们用它去创建生成的代码文件

```
	private void generateCode(String className, String code) {
        try {
            JavaFileObject file = mFiler.createSourceFile(className);
            Writer writer = file.openWriter();
            writer.write(code);
            writer.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

## Messager

Messager 顾名思义就是用于打印的,它会打印出Element所在的源代码，它还会抛出异常。靠默认的错误打印有时很难找出错误的地方,我们可以用它去添加更直观的日志打印

当用InjectView标注了非View的成员变量我们就会打印错误并抛出异常(这里我们使用Diagnostic.Kind.ERROR,这个打印会抛出异常终止Processor):

```
//如果不是View的子类则报错
if (!isView(element.asType())){
	mMessager.printMessage(Diagnostic.Kind.ERROR, "is not a View", element);
}
```

例如我们如果在MainActivity中为一个String变量标注InjectView:

```
//在非View上使用InjectView就会报错
@InjectView(R.id.button)
String x;
```

则会报错:

```
  符号:   类 ViewInjector
  位置: 程序包 linjw.demo.injector
/Users/linjw/workspace/ProcessorDemo/app/src/main/java/linjw/demo/processordemo/MainActivity.java:22: 错误: is not a View
    String x;
           ^
```

如果我们不用Messager去打印,生成的代码之后也会有打印,但是就不是那么清晰了:

```
/Users/linjw/workspace/ProcessorDemo/app/build/generated/source/apt/debug/MainActivityInjector.java:7: 错误: 不兼容的类型: View无法转换为String
                arg.x=(java.lang.String)arg.findViewById(2131427415);
```

# Element的子接口

我们在process方法中使用getElementsAnnotatedWith获取到的都是Element接口,其实我们用Element.getKind获取到类型之后可以将他们强转成对应的子接口,这些子接口提供了一些针对性的操作。

这些子接口有:

- TypeElement：表示一个类或接口程序元素。
- PackageElement：表示一个包程序元素。
- VariableElement：表示一个属性、enum 常量、方法或构造方法参数、局部变量或异常参数。
- ExecutableElement：表示某个类或接口的方法、构造方法或初始化程序（静态或实例），包括注释类型元素。

对应关系如下

```
package linjw.demo;  // PackageElement
public class Person {  // TypeElement
    private String mName;  // VariableElement
    public Person () {}  // ExecutableElement
    public void setName (String name) {mName=name;}  // ExecutableElement
}
```

# Element的一些常用操作

获取类名:

- Element.getSimpleName().toString(); // 获取类名
- Element.asType().toString(); //获取类的全名

获取所在的包名:

- Elements.getPackageOf(Element).asType().toString();

获取所在的类:

- Element.getEnclosingElement();

获取父类:

- Types.directSupertypes(Element.asType())

获取标注对象的类型:

- Element.getKind()

# Demo地址

可以在[这里](https://github.com/bluesky466/ProcessorDemo)查看完整代码
