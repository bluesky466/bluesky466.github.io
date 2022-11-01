title: 当Gson遇上data class
date: 2022-11-01 23:01:55
tags:
    - 技术相关
    - Android
---

当Gson遇上kotlin data class,会发生一些很有意思的现象:

现象1: 非空类型失效

```
data class TestData(
    val a: String,
    val b: String
)
val data = Gson().fromJson("{}", TestData::class.java)
println("a:${data.a}, b:${data.b}") //输出: a:null, b:null
```

现象2: 构造函数不会被调用
```
data class TestData(
    val a: String,
    val b: String
) {
    init {
        println("TestData init!!!") // 这一行代码不会执行到
    }
}
val data = Gson().fromJson("{}", TestData::class.java)
```

现象3: 默认值失效

```
data class TestData(
    val a: String,
    val b: String = "bbb"
)
val data = Gson().fromJson("{\"a\":\"aaa\"}", TestData::class.java)
println("$data") //输出: TestData(a=aaa, b=null)
```

现象4: 当全部成员都有默认值的时候默认值和构造函数又生效了

```
data class TestData(
    val a: String = "",
    val b: String = "bbb"
) {
    init {
        println("TestData init!!!") // 这一行代码能执行到
    }
}
val data = Gson().fromJson("{\"a\":\"aaa\"}", TestData::class.java)
println("$data") //输出: TestData(a=aaa, b=bbb)
```

# Gson解析流程

要理解上面的现象我们先要了解Gson是怎样工作的。

Gson解析json分两步,创建对象实例和给成员变量赋值.

创建对象实例是通过ConstructorConstructor.get(TypeToken\<T\> typeToken)方法获取到构造器去创建的:

```
public <T> ObjectConstructor<T> get(TypeToken<T> typeToken) {
    final Type type = typeToken.getType();
    final Class<? super T> rawType = typeToken.getRawType();

    // 从instanceCreators中查找,我们可以用GsonBuilder.registerTypeAdapter指定某种类型的构造器,默认情况下instanceCreators是空的
    final InstanceCreator<T> typeCreator = (InstanceCreator<T>) instanceCreators.get(type);
    if (typeCreator != null) {
      return new ObjectConstructor<T>() {
        @Override public T construct() {
          return typeCreator.createInstance(type);
        }
      };
    }

    // 这里还是在instanceCreators里查找,只不过用rawType当key
    final InstanceCreator<T> rawTypeCreator =
        (InstanceCreator<T>) instanceCreators.get(rawType);
    if (rawTypeCreator != null) {
      return new ObjectConstructor<T>() {
        @Override public T construct() {
          return rawTypeCreator.createInstance(type);
        }
      };
    }

    // 查找一些特殊集合如EnumSet、EnumMap的构造器
    ObjectConstructor<T> specialConstructor = newSpecialCollectionConstructor(type, rawType);
    if (specialConstructor != null) {
      return specialConstructor;
    }

    // 通过rawType.getDeclaredConstructor()反射获取类的无参构造函数
    FilterResult filterResult = ReflectionAccessFilterHelper.getFilterResult(reflectionFilters, rawType);
    ObjectConstructor<T> defaultConstructor = newDefaultConstructor(rawType, filterResult);
    if (defaultConstructor != null) {
      return defaultConstructor;
    }

    // 查找普通的Collection或者Map,如ArrayList、HashMap等的构造器
    ObjectConstructor<T> defaultImplementation = newDefaultImplementationConstructor(type, rawType);
    if (defaultImplementation != null) {
      return defaultImplementation;
    }

    // 判断类型是否可以实例化,例如接口和抽象类就不能实例化
    final String exceptionMessage = checkInstantiable(rawType);
    if (exceptionMessage != null) {
      return new ObjectConstructor<T>() {
        @Override public T construct() {
          throw new JsonIOException(exceptionMessage);
        }
      };
    }

    // 最后使用sun.misc.Unsafe去兜底创建实例
    if (filterResult == FilterResult.ALLOW) {
      return newUnsafeAllocator(rawType);
    } else {
      final String message = "Unable to create instance of " + rawType + "; ReflectionAccessFilter "
          + "does not permit using reflection or Unsafe. Register an InstanceCreator or a TypeAdapter "
          + "for this type or adjust the access filter to allow using reflection.";
      return new ObjectConstructor<T>() {
        @Override public T construct() {
          throw new JsonIOException(message);
        }
      };
    }
  }
```

获取到对象的构造器,之后就能用它去创建对象实例,然后遍历json字段查找对象是否有对应的成员变量,如果有就通过反射设置进去:

```
@Override
public T read(JsonReader in) throws IOException {
  if (in.peek() == JsonToken.NULL) {
    in.nextNull();
    return null;
  }

  // 通过ConstructorConstructor.get(TypeToken\<T\> typeToken)查询的构造器创建实例对象
  A accumulator = createAccumulator();

  try {
    in.beginObject();
    // 遍历json
    while (in.hasNext()) {
      String name = in.nextName();

      // 从对象的成员变量列表查询是否有该字段
      BoundField field = boundFields.get(name);
      if (field == null || !field.deserialized) {
        // 对象没有该成员变量则跳过
        in.skipValue();
      } else {
        // 对象有该成员变量则读取json的值,通过反射设置给对象
        readField(accumulator, in, field);
      }
    }
  } catch (IllegalStateException e) {
    throw new JsonSyntaxException(e);
  } catch (IllegalAccessException e) {
    throw ReflectionHelper.createExceptionForUnexpectedIllegalAccess(e);
  }
  in.endObject();
  return finalize(accumulator);
}
```

# 非空类型失效和构造函数不会被调用的原理

了解了Gson的解析流程之后我们再来看看问题1的data class对应的java代码:

```
// kotlin代码
data class TestData(
    val a: String,
    val b: String
)

// java对应的类
public final class TestData {
   private final String a;
   private final String b;
   ...
   public final String getA() {
      return this.a;
   }
   public final String getB() {
      return this.b;
   }
   ...
   public TestData(@NotNull String a, @NotNull String b) {
      Intrinsics.checkNotNullParameter(a, "a");
      Intrinsics.checkNotNullParameter(b, "b");
      super();
      this.a = a;
      this.b = b;
   }
   ...
   @NotNull
   public String toString() {
      return "TestData(a=" + this.a + ", b=" + this.b + ")";
   }
   ...
}
```

可以看到只有在构造函数里面做了判空,但是它并没有无参构造函数所以gson是通过Unsafe去兜底创建TestData实例的。Unsafe创建类的实例并不会调用到构造函数,所以就绕过类判空的步骤。

同理也能解释现象2构造函数不会被调用的问题。

## Unsafe

> Unsafe是位于sun.misc包下的一个类，主要提供一些用于执行低级别、不安全操作的方法，如直接访问系统内存资源、自主管理内存资源等，这些方法在提升Java运行效率、增强Java语言底层资源操作能力方面起到了很大的作用。 -- [Java魔法类：Unsafe应用解析](https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html)

我们可以通过下面的代码创建TestData实例而不调用TestData的构造函数:

```
val unsafeClass = Class.forName("sun.misc.Unsafe")
val theUnsafe = unsafeClass.getDeclaredField("theUnsafe")
theUnsafe.isAccessible = true
val unsafe = theUnsafe.get(null)
val allocateInstance = unsafeClass.getMethod("allocateInstance", Class::class.java)
val testData = allocateInstance.invoke(unsafe, TestData::class.java) as TestData
```

## data class 默认值的原理

接着我们继续来看现象3默认值失效的问题,这里会牵扯到data class默认值的原理,我们来看看对应的java代码:

```
// kotlin代码
data class TestData(
    val a: String,
    val b: String = "bbb"
)

// java对应的类
public final class TestData {
   private final String a;
   private final String b;

   ...

   public TestData(@NotNull String a, @NotNull String b) {
      Intrinsics.checkNotNullParameter(a, "a");
      Intrinsics.checkNotNullParameter(b, "b");
      super();
      this.a = a;
      this.b = b;
   }

   public TestData(String var1, String var2, int var3, DefaultConstructorMarker var4) {
      if ((var3 & 2) != 0) {
         var2 = "bbb";
      }

      this(var1, var2);
   }
   ...
}
```

可以看到,kotlin的默认参数并不是通过重载实现的,而是新增一个构造函数,用一个int的各个bit位来表示前面的参数是否需要设置成默认值。


```
// 例如下面这行kotlin代码:
val testData = TestData("aaa")

// 对应的java代码是这样的:
TestData testData = new TestData("aaa", (String)null, 2, (DefaultConstructorMarker)null);
```

这样做的好处在于只需要新建一个构造函数。用下面这种java传统的函数重载来做,如果有很多的默认值的话需要创建很多的构造函数:

```
public final class TestData {
   ...

   public TestData(@NotNull String a, @NotNull String b) {
      ...
   }

   public TestData(String var1) {
      this(var1, "bbb");
   }
   ...
}
```

到这里我们也能理解现象3默认值失效的原因了,和前面的两个现象一样是因为没有调用到TestData的构造函数,所以就没有赋默认值.

## DefaultConstructorMarker

另外在生成的构造函数里我们还看到了一个DefaultConstructorMarker参数:

```
public TestData(String var1, String var2, int var3, DefaultConstructorMarker var4)
```

这个参数会在kotlin自动生成的构造函数里面出现,目的是为了防止和我们自己定义的构造函数碰撞:

```
// kotlin代码
data class TestData(
    val a: String,
    val b: String = "bbb"
) {
    constructor(a: String, b: String, i: Int) : this(a, b) {

    }
}

// 对应的java代码
public final class TestData {
   private final String a;
   private final String b;

   public TestData(@NotNull String a, @NotNull String b) {
      ...
   }

   // 假设没有DefaultConstructorMarker参数,下面的两个构造函数就会撞车了
   public TestData(String var1, String var2, int var3, DefaultConstructorMarker var4) {
      ...
   }

   public TestData(@NotNull String a, @NotNull String b, int i) {
      Intrinsics.checkNotNullParameter(a, "a");
      Intrinsics.checkNotNullParameter(b, "b");
      this(a, b);
   }
   ...
}

```

# 全部成员都有默认值的情况

最后我们来分析下现象4当全部成员都有默认值的情况:

```
// kotlin代码
data class TestData(
    val a: String = "",
    val b: String = "bbb"
) {
    init {
        println("TestData init!!!")
    }
}

// 对应的java代码
public final class TestData {
   private final String a;
   private final String b;

   ...

   public TestData(@NotNull String a, @NotNull String b) {
      Intrinsics.checkNotNullParameter(a, "a");
      Intrinsics.checkNotNullParameter(b, "b");
      super();
      this.a = a;
      this.b = b;
      String var3 = "TestData init!!!";
      boolean var4 = false;
      System.out.println(var3);
   }

   public TestData(String var1, String var2, int var3, DefaultConstructorMarker var4) {
      if ((var3 & 1) != 0) {
         var1 = "";
      }

      if ((var3 & 2) != 0) {
         var2 = "bbb";
      }

      this(var1, var2);
   }

   public TestData() {
      this((String)null, (String)null, 3, (DefaultConstructorMarker)null);
   }

   ...
}
```

可以看到当所有成员都有默认值的时候,会生成无参构造函数,这样的话Gson就会调用无参构造函数去创建实例。

# 解决思路

了解完原理我们来看看怎么解决默认值无效的问题,下面有一些思路:

1. 当需要使用默认值的时候全部成员变量都加上默认值
2. 使用代码生成的方式创建InstanceCreator并注册到gson,在里面创建实例并预先填好默认值
3. 改用对kotlin支持更好的[kotlinx.serialization](https://github.com/Kotlin/kotlinx.serialization)或者[moshi](https://github.com/square/moshi)