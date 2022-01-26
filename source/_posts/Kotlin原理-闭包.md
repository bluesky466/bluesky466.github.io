title: Kotlin原理-闭包
date: 2022-01-26 13:59:49
tags:
    - 技术相关
    - Android
---

讲kotlin的闭包我想从java的闭包讲起。

# Java不完整闭包

Java的闭包是不完整的,它要求匿名内部类使用外部的变量必现是final的,这就使我们不能在匿名内部类里面修改外部变量的值:

```
final Integer data = 0; // 必须是final

View.OnClickListener listener = new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        Log.d(TAG, " data = " + data);
    }
};
findViewById(R.id.button).setOnClickListener(listener);
```

这个要求的根本原因在于方法内的临时变量是存放在栈区的,一旦方法调用完成,这部分的内存就会被释放。如果我们再去修改这块的内存就会造成不可预期的后果。

但是问题又来了,如果一旦方法调用完成,上面例子的data这个引用的内存就会被回收。那么在onClick的时候为什么还能读取到值呢?

原因在于java实现匿名内部类的原理是Java编译器会给它生成一个实际的类,将外部变量保存到这个类的成员变量里,我们可以通过下面的代码打印出这个生成的类:

```
Log.d(TAG, "class " + listener.getClass() + " { ");
for (Field field : listener.getClass().getDeclaredFields()) {
    StringBuilder sb = new StringBuilder();
    sb.append("\t");

    if (Modifier.isFinal(field.getModifiers())) {
        sb.append("final ");
    }

    sb.append(field.getType().getName())
            .append(" ")
            .append(field.getName())
            .append(";");

    Log.d(TAG, sb.toString());
}
Log.d(TAG, "}");
```

输出如下:

```
D closure : class class me.linjw.demo.MainActivity$1 {
D closure :      final me.linjw.demo.MainActivity this$0;
D closure :      final java.lang.Integer val$data;
D closure : }
```

可以看到这个类除了用成员变量保存了外部的data的副本之外,还保存了外部类MainActivity的引用。这也是匿名内部类/非静态内部类持有外部类引用的原理。

# 突破Java不完整闭包限制

知道了Java匿名内部类持有外部对象引用的原理之后,我们其实是可以通过下面的方法绕过不能修改外部对象的限制的:

```
final Integer[] data = new Integer[]{0};

View.OnClickListener listener = new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        data[0]++;
        Log.d(TAG, " data = " + data[0]);
    }
};
```

这种做法的原理在于,data这个引用本身的内存在栈区,方法调用完会被回收,但是它所指向的数组的内存在堆区,只有还有引用指向它就不会被回收。而Java编译器生成的这个类巧合就有个成员变量保存了data的副本,指向了这个数组。

用c++的话来讲就是,指针在方法结束的时候被回收了,但是指针所指向的堆内存没有被回收。

# Kotlin闭包原理

Kotlin闭包的原理实际上就是上面讲的突破Java不完整闭包限制的原理。用工具查看下面Kotlin代码生成的Java字节码:

```
var data = 0
findViewById<Button>(R.id.button).setOnClickListener {
    data++
    Log.d(TAG,"data = $data")
}
```

可以看到data实际指向了一个IntRef的堆内存,在IntRef的element成员里面保存的才是实际的值:

```
final IntRef data = new IntRef();
data.element = 0;
((Button)this.findViewById(1000095)).setOnClickListener((OnClickListener)(new OnClickListener() {
   public final void onClick(View it) {
      int var10001 = data.element++;
      Log.d(MainActivity.this.TAG, "data = " + data.element);
   }
}));
```
