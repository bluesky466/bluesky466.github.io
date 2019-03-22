title: 大概优秀的java程序员都要会分析class文件吧
date: 2019-03-22 19:53:41
tags:
	- 技术相关
	- java
---

相信大家在学java的时候都会听到这样的一些结论:

1. enum 是一个类
2. 泛型的实现使用了类型擦除技术
3. 非静态内部类持有外部类的引用
4. 需要将自由变量声明成final才能给匿名内部类访问

...

初学的时候的时候可能在书上读过,但是很容易就会忘记,等到踩坑踩多了,就会形成慢慢记住。但是很多的同学也只是记住了而已,对于实际的原理或者原因并不了解。

这篇文章的目的主要就是教会大家查看java的字节码,然后懂得去分析这些结论背后的原理。

# 枚举最后会被编译成一个类

我们先从简单的入手.

java的新手对于枚举的理解可能是:存储几个固定值的集合,例如下面的Color枚举,使用的时候最多也就通过ordinal()方法获取下枚举的序号或者从Color.values()里面使用序号拿到一个Color:

```
public enum Color {
    RED,
    GREEN,
    BLUE
}

int index = Color.BLUE.ordinal();
Color color = Color.values()[index];
```

如果是从C/C++过来的人比如我,很容易形成这样一种固定的思维:枚举就是一种被命名的整型的集合。

在c/c++里面这种想法还能说的过去,但是到了java就大错特错了,错过了java枚举的一些好用的特性。

还是拿我们上面的Color枚举,颜色我们经常使用0xFF0000这样的16进制整型或者“#FF0000”这样的字符串去表示。

在java中,我们可以这样将这个Color枚举和整型还有字符串关联起来:


```
public enum Color {
    RED(0xFF0000, "#FF0000"),
    GREEN(0x00FF00, "#00FF00"),
    BLUE(0x0000FF, "#0000FF");

    private int mIntVal;
    private String mStrVal;

    Color(int intVal, String strVal) {
        mIntVal = intVal;
        mStrVal = strVal;
    }

    public int getIntVal() {
        return mIntVal;
    }

    public String getStrVal() {
        return mStrVal;
    }
}

System.out.println(Color.RED.getIntVal());
System.out.println(Color.RED.getStrVal());
```

可以看到我们给Color这个枚举,增加了两个成员变量用来存整型和字符串的表示,然后还提供两个get方法给外部获取。


甚至进一步的,枚举的一种比较常用的技巧就是在static块中创建映射:

```
public enum Color {
    RED(0xFF0000, "#FF0000"),
    GREEN(0x00FF00, "#00FF00"),
    BLUE(0x0000FF, "#0000FF");

    private static final Map<String, Color> sMap = new HashMap<>();

    static {
        for (Color color : Color.values()) {
            sMap.put(color.getStrVal(), color);
        }
    }

    public static Color getFromStrVal(String strVal){
        return sMap.get(strVal);
    }

    private int mIntVal;
    private String mStrVal;

    Color(int intVal, String strVal) {
        mIntVal = intVal;
        mStrVal = strVal;
    }

    public int getIntVal() {
        return mIntVal;
    }

    public String getStrVal() {
        return mStrVal;
    }
}

System.out.println(Color.getFromStrVal("#FF0000").getIntVal());
System.out.println(Color.RED.getIntVal());
```

看起来是不是感觉和一个类的用法很像?"enum 是一个类"这样句话是不是讲的很有道理。

当然用法和类很像并不能说明什么。

接下来就到了我们这篇文章想讲的第一个关键知识点了。

## 反编译class文件

首先我们还是将Color简化回最初的样子,然后保存在Color.java文件中:

```
// Color.java
public enum Color {
    RED,
    GREEN,
    BLUE
}
```

然后通过javac命令进行编译,得到Color.class

> javac Color.java

得到的class文件就是jvm可以加载运行的文件,里面都是一些java的字节码。

java其实默认提供了一个javap命令，给我们去查看class文件里面的代码。例如,在Color.class所在的目录使用下面命令:

> javap Color

可以看到下面的输出:

```
Compiled from "Color.java"
public final class Color extends java.lang.Enum<Color> {
  public static final Color RED;
  public static final Color GREEN;
  public static final Color BLUE;
  public static Color[] values();
  public static Color valueOf(java.lang.String);
  static {};
}
```

是不是有种恍然大明白的感觉?Color在class文件里面实际上是被编译成了一个继承java.lang.Enum的类,而我们定义的RED、GREEN、BLUE实际上是这个类的静态成员变量。

这么去看的话我们那些加成员变量、加方法的操作是不是就变得很常规了?

所以说"enum 是一个类"的意思其实是enum会被java编译器编译成一个继承java.lang.Enum的类!

# java运行时栈帧

相信大家都知道,java虚拟机里面的方法调用是以方法栈的形式去执行的.压人栈内的元素就叫做栈帧.

<深入理解java虚拟机>一书中是这么介绍栈帧的:

> 栈帧(Stack Frame)是用于支持虚拟机进行方法调用和方法执行的数据结构，它是虚拟机运行时数据区的虚拟机栈(Virtual Machine Stack)的栈元素。栈帧存储了方法的局部变量表，操作数栈，动态连接和方法返回地址等信息。第一个方法从调用开始到执行完成，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。

也就是说,java方法的调用,其实是一个个栈帧入栈出栈的过程,而栈帧内部又包含了局部变量表,操作数栈等部分:

{% img /大概优秀的java程序员都要会分析class文件吧/1.png %}


局部变量表和操作数栈是栈帧内进行执行字节码的重要部分.

局部变量表顾名思义,就是用来保存方法参数和方法内部定义的局部变量的一段内存区域.

而操作数栈也是一个后入先出的栈,程序运行过程中各种字节码指令往其中压入和弹出栈进行运算的.

## java字节码分析

我们用一个简单的代码做demo:

```
// Test.java
public class Test {                                                              
    public static void main(String[] args) {                                     
        int a = 12;                                                              
        int b = 21;                                                              
        int c = a + b;                                                           
        System.out.println(String.valueOf(c));                                   
    }                                                                            
}
```

首先使用javac命令编译代码,然后使用javap命令查看字节码:

> javac Test.java
> javap Test

得到下面的输出:

```
Compiled from "Test.java"
public class Test {
  public Test();
  public static void main(java.lang.String[]);
}
```

可以看到这里只有方法的声明,并没有具体的代码执行过程.这是因为执行过程都被编译成一个个字节码指令了.

我们可以用javap -c命令被这些指令也显示出来:

> javap -c Test

输出为:

```
Compiled from "Test.java"
public class Test {
  public Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: bipush        12
       2: istore_1
       3: bipush        21
       5: istore_2
       6: iload_1
       7: iload_2
       8: iadd
       9: istore_3
      10: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      13: iload_3
      14: invokestatic  #3                  // Method java/lang/String.valueOf:(I)Ljava/lang/String;
      17: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      20: return
}
```

我们来一步步分析main方法里面的字节码指令:

```
// 将12这个常量压入操作数栈
0: bipush        12

// 弹出操作数栈顶的元素,保存到局部变量表第1个位置中,即将12从栈顶弹出,保存成变量1,此时栈已空
2: istore_1

// 将21这个常量压入操作数栈
3: bipush        21

// 弹出操作数栈顶的元素,保存到局部变量表第2个位置中,即将21从栈顶弹出,保存成变量2,此时栈已空
5: istore_2

// 从局部变量表获取第1个位置的元素,压入操作数栈中,即将12压入栈中
6: iload_1

// 从局部变量表获取第2个位置的元素,压入操作数栈中,即将21压入栈中
7: iload_2

// 弹出操作数栈顶的两个元素,进行加法操作,得到的结果再压入栈中,即弹出21和12相加得到33,再压入栈中
8: iadd

// 弹出操作数栈顶的元素,保存到局部变量表第3个位置中,即将33从栈顶弹出,保存成变量3,此时栈已空
9: istore_3

// 读取System中的静态成员变量out压入栈中
10: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;

// 从局部变量表获取第3个位置的元素,压入操作数栈中,即将33压入栈中
13: iload_3

// 弹出栈顶的33,执行String.valueOf方法,并将得到的返回值"33"压回栈中
14: invokestatic  #3                  // Method java/lang/String.valueOf:(I)Ljava/lang/String;

// 弹出栈顶的"33"和System.out变量去执行println方法
17: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V

// 退出方法
20: return
```

上面的的流程比较复杂，可以结合下面这个动图理解一下:

{% img /大概优秀的java程序员都要会分析class文件吧/gif1.gif %}

如果看的比较仔细的同学可能会有疑问，为什么举报变量表里一开始位置0就会有个String[]在那呢？

其实这个字符串数组就是传入的参数args,jvm会把参数都压如举报变量表给方法去使用,如果调用的是非静态方法,还会将该方法的调用对象也一起压入栈中.

可能有同学一开始会对istore、iload...这些字节码指令的作用不那么熟悉,或者有些指令不知道有什么作用。不过这个没有关系，不需要死记硬背，遇到的时候搜索一下就是了。

# 类型擦除的原理

泛型是java中十分好用且常用的技术,之前也有写过两篇博客 [《java泛型那些事》](http://blog.islinjw.cn/2018/01/06/java%E6%B3%9B%E5%9E%8B%E9%82%A3%E4%BA%9B%E4%BA%8B/),[《再谈Java泛型》](http://blog.islinjw.cn/2018/02/04/%E5%86%8D%E8%B0%88Java%E6%B3%9B%E5%9E%8B/)总结过.感兴趣的同学可以去看看.

这里我们就从编译出来的class文件里面看看泛型的实现:

```
public class Test {                                                              
    public static void main(String[] args) {                                     
        foo(1);                                                                  
    }                                                                            

    public static <T> T foo(T a){                                                
        return a;                                                                
    }                                                                            
}
```

让我们使用"javap -c"命令看看它生成的class文件是怎样的:

```
Compiled from "Test.java"
public class Test {
  public Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: iconst_1
       1: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
       4: invokestatic  #3                  // Method foo:(Ljava/lang/Object;)Ljava/lang/Object;
       7: pop
       8: return

  public static <T> T foo(T);
    Code:
       0: aload_0
       1: areturn
}
```

可以看到虽然声明部分还是可以看到泛型的影子:

> public static <T> T foo(T);

但是在调用的时候实际上是

>  Method foo:(Ljava/lang/Object;)Ljava/lang/Object;

main 方法中先用iconst_1将常量1压入栈中,然后用Integer.valueOf方法装箱成Integer最后调用参数和返回值都是Object的foo方法.

所以说泛型的实现原理实际上是将类型都变成了Obejct,所以才能接收所有继承Object的类型,但是像int,char这种不是继承Object的类型是不能传入的.

然后由于类型最后都被擦除剩下Object了,所以jvm是不知道原来输入的类型的,于是乎下面的这种代码就不能编译通过了:

```
public <T> T foo(){
    return new T(); // 编译失败,因为T的类型最后会被擦除,变成Object
}
```

# 非静态内部类持有外部类的引用的原因

我们都知道非静态内部类是持有外部类的引用的,所以在安卓中使用Handler的话一般会声明成静态内部类,然后加上弱引用去防止内存泄露.

接下来就让我们一起看看非静态内部类是怎么持有外部类的引用的。先写一个简单的例子:

```
public class Test {
    public void foo() {
        Runnable r = new Runnable() {
            @Override
            public void run() {
                System.out.println(String.valueOf(Test.this));
            }
        };
    }
}
```

通过javac命令编译之后发现得到了两个class文件:

> Test$1.class Test.class

Test.class文件好理解应该就是Test这个类的定义,那Test$1.class定义的Test$1类又是从哪里来的呢？

这里还有个大家可能忽略的知识点,java里面变量名类名是可以包含$符号的,例如下面的代码都是合法且可以通过编译并且正常运行的

```
int x$y = 123;
System.out.println(x$y);
```

回到正题,让我们先来用"javap -c"命令看看Test.class里面的内容:

```
Compiled from "Test.java"
public class Test {
  public Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public void foo();
    Code:
       0: new           #2                  // class Test$1
       3: dup
       4: aload_0
       5: invokespecial #3                  // Method Test$1."<init>":(LTest;)V
       8: astore_1
       9: return
}
```

我们来解析下foo方法:

```
// new一个Test$1类的对象,压入栈中
0: new           #2                  // class Test$1

// 复制一份栈顶的元素压入栈中,即现在栈里面有两个相同的Test$1对象
3: dup

// 将局部变量表位置为0的元素压入栈中,由于foo方法不是静态方法,所以这个元素实际上就是Test对象,即this
4: aload_0

// 调用Test$1(Test)这个构造方法,它有一个Test类型的参数,我们传入的就是栈顶的Test对象,同时我们会将栈顶第二个元素Test$1对象也传进去(也就是说用这个Test$1对象去执行构造方法)。于是我们就弹出了栈顶的一个Test对象和一个Test$1对象
5: invokespecial #3                  // Method Test$1."<init>":(LTest;)V

// 将栈剩下的最后一个Test$1保存到局部变量表的位置1中。
8: astore_1

// 退出方法
9: return
```

根据上面的字节码,我们可以逆向得到下面的代码:

```
public class Test {
    public void foo() {
        Runnable r = new Test$1(this);
    }
}
```

接着我们再来看看Test$1.class：

```
Compiled from "Test.java"
class Test$1 implements java.lang.Runnable {
  final Test this$0;

  Test$1(Test);
    Code:
       0: aload_0
       1: aload_1
       2: putfield      #1                  // Field this$0:LTest;
       5: aload_0
       6: invokespecial #2                  // Method java/lang/Object."<init>":()V
       9: return

  public void run();
    Code:
       0: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: aload_0
       4: getfield      #1                  // Field this$0:LTest;
       7: invokestatic  #4                  // Method java/lang/String.valueOf:(Ljava/lang/Object;)Ljava/lang/String;
      10: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      13: return
}
```

这里定义了一个实现Runnable接口的Test$1类，它有一个参数为Test的构造方法和一个run方法。然后还有一个Test类型的成员变量this$0。继续解析这个两个方法的字节码:

```
  Test$1(Test);
    Code:
    	// 将局部变量表中位置为0的元素压入栈中,由于这个方法不是静态的,所以这个元素就是Test$1的this对象
       0: aload_0

       // 将局部变量表位置为1的元素压入栈中,这个元素就是我们传入的参数Test对象
       1: aload_1

       // 这里弹出栈顶的两个元素,第一个Test对象,赋值给第二元素Test$1对象的this$0成员变量。也就是把我们传进来的Test对象保存给成员变量 this$0
       2: putfield      #1                  // Field this$0:LTest;

       // 将局部变量表中位置为0的元素压入栈中,还是Test$1的this对象
       5: aload_0

       // 使用栈顶Test$1的this对象去初始化
       6: invokespecial #2                  // Method java/lang/Object."<init>":()V

       // 退出方法
       9: return

  public void run();
    Code:
        //拿到System的静态成员变量out压入栈中
       0: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;

       // 将局部变量表中位置为0的元素压入栈中,由于这个方法不是静态的,所以这个元素就是Test$1的this对象
       3: aload_0

       // 弹出栈顶Test$1的this对象,获取它的this$0成员变量,压入栈中
       4: getfield      #1                  // Field this$0:LTest;

       // 弹出栈顶的this$0对象执行String.valueOf方法,得到的String对象压入栈中
       7: invokestatic  #4                  // Method java/lang/String.valueOf:(Ljava/lang/Object;)Ljava/lang/String;

       // 弹出栈顶的String对象和System.out对象去执行println方法,即调用System.out.println打印这个String对象
       10: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V

       // 退出方法
       13: return
```

来来来,我们继续脑补它的源代码:

```
public class Test$1 implements java.lang.Runnable {
    final Test this$0;

    public Test$1(Test test) {
        this$0 = test;
    }

    @Override
    public void run() {
        System.out.println(String.valueOf(this$0));
    }
}
```

所以我们通过字节码,发现下面这个代码:

```
public class Test {
    public void foo() {
        Runnable r = new Runnable() {
            @Override
            public void run() {
                System.out.println(String.valueOf(Test.this));
            }
        };
    }
}
```

编译之后最终会生成两个类:

```
public class Test {
    public void foo() {
        Runnable r = new Test$1(this);
    }
}

public class Test$1 implements java.lang.Runnable {
    final Test this$0;

    public Test$1(Test test) {
        this$0 = test;
    }

    @Override
    public void run() {
        System.out.println(String.valueOf(this$0));
    }
}
```

这就是非静态内部类持有外部类的引用的原因啦。

到这里这篇文章想讲的东西就已经都讲完了,还剩下一个问题就当做作业让同学们自己尝试这去分析吧:

> 需要将自由变量声明成final才能给匿名内部类访问
