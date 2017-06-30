title: Android温故而知新 - launchMode
date: 2017-06-30 01:07:02
tags:
    - 技术相关
    - Android
---

Activity有四种launchMode,[android官方文档](https://developer.android.com/guide/topics/manifest/activity-element.html?hl=zh-cn)的介绍如下:

|启动模式|多个实例?|注释|
|:-----:|:-----:|:--:|
| “standard”|是|默认值。系统始终会在目标任务中创建新的 Activity 实例并向其传送 Intent。|
|“singleTop”|有条件|如果目标任务的顶部已存在一个 Activity 实例，则系统会通过调用该实例的 onNewIntent() 方法向其传送 Intent，而不是创建新的 Activity 实例。|
|“singleTask”| 否|系统在新任务的根位置创建 Activity 并向其传送 Intent。 不过，如果已存在一个 Activity 实例，则系统会通过调用该实例的 onNewIntent() 方法向其传送 Intent，而不是创建新的 Activity 实例。|
|“singleInstance”|否|与“singleTask"”相同，只是系统不会将任何其他 Activity 启动到包含实例的任务中。 该 Activity 始终是其任务唯一仅有的成员。|

# 测试demo

我们通过两个Activity之间的跳转和任务栈打印来理解launchMode的作用。

这两个Activity使用同一个布局, TextView用来打印任务栈的id和activity实例，然后是两个按钮，分别用来启动两个activity:

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:id="@+id/label"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

    <Button
        android:id="@+id/gotoFirstActivity"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:onClick="onClick"
        android:text="goto first activity" />

    <Button
        android:id="@+id/gotoSecondActivity"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:onClick="onClick"
        android:text="goto second activity" />

</LinearLayout>
```

Activty代码如下:

```
public class FirstActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_layout);

        Utils.initActivity(this);
    }

    public void onClick(View view) {
        Utils.onClick(this, view.getId());
    }
}
```

```
public class SecondActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_layout);

        Utils.initActivity(this);
    }

    public void onClick(View view) {
        Utils.onClick(this, view.getId());
    }
}
```

因为两个activity的代码很相似，所以我把它们放到了静态类Utils中:

```
public class Utils {
    public static void initActivity(Activity activity) {
        TextView label = (TextView) activity.findViewById(R.id.label);
        label.setText("task : " + activity.getTaskId() + " activity : " + activity);
    }


    public static void onClick(Context context, int id) {
        Intent intent;
        if (id == R.id.gotoFirstActivity) {
            intent = new Intent(context, FirstActivity.class);
        } else {
            intent = new Intent(context, SecondActivity.class);
        }
        context.startActivity(intent);
    }
}
```

# 查看任务栈

我们可以在adb shell中使用dumpsys activity可以看到任务栈。这个命令的打印会比较多,但是有两个部分是比较重要的。

一个是Recent tasks,这里可以看到各个任务栈的总体信息,如我们的demo在第0个任务栈,它的Task id 是483,包名是linjw.demo.launchmodedemo,栈里面有6个Activity

```
ACTIVITY MANAGER RECENT TASKS (dumpsys activity recents)
  Recent tasks:
  * Recent #0: TaskRecord{f69123d #483 A=linjw.demo.launchmodedemo U=0 sz=6}
  * Recent #1: TaskRecord{b798a32 #487 A=com.android.systemui U=0 sz=0}
  * Recent #2: TaskRecord{f874383 #486 A=com.tencent.mm U=0 sz=1}
  * Recent #3: TaskRecord{d7bd900 #479 A=com.meizu.flyme.launcher U=0 sz=1}
  * Recent #4: TaskRecord{8f5d797 #482 A=com.tencent.mobileqq U=0 sz=0}
  * Recent #5: TaskRecord{fd99539 #481 A=android.task.stk.task U=0 sz=0}
  * Recent #6: TaskRecord{7c9951e #480 A=com.android.incallui U=0 sz=1}
  * Recent #7: TaskRecord{e32177e #478 A=com.zhihu.android U=0 sz=0}
  * Recent #8: TaskRecord{7f094df #472 A=com.meizu.media.reader U=0 sz=0}
```

在它的下面可以看到里面的具体的activity,并且可以看到叫起它的Intent:

```
ACTIVITY MANAGER ACTIVITIES (dumpsys activity activities)
Display #0 (activities from top to bottom):
  Stack #1:
    Task id #483
      TaskRecord{f69123d #483 A=linjw.demo.launchmodedemo U=0 sz=6}
      Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10100000 cmp=linjw.demo.launchmodedemo/.FirstActivity }
        Hist #5: ActivityRecord{786c71e u0 linjw.demo.launchmodedemo/.FirstActivity t483}
          Intent { cmp=linjw.demo.launchmodedemo/.FirstActivity }
          ProcessRecord{245852c 7994:linjw.demo.launchmodedemo/u0a61}
        Hist #4: ActivityRecord{6d5da5d u0 linjw.demo.launchmodedemo/.SecondActivity t483}
          Intent { cmp=linjw.demo.launchmodedemo/.SecondActivity }
          ProcessRecord{245852c 7994:linjw.demo.launchmodedemo/u0a61}
        Hist #3: ActivityRecord{90b9f88 u0 linjw.demo.launchmodedemo/.FirstActivity t483}
          Intent { cmp=linjw.demo.launchmodedemo/.FirstActivity }
          ProcessRecord{245852c 7994:linjw.demo.launchmodedemo/u0a61}
        Hist #2: ActivityRecord{1de66e u0 linjw.demo.launchmodedemo/.SecondActivity t483}
          Intent { cmp=linjw.demo.launchmodedemo/.SecondActivity }
          ProcessRecord{245852c 7994:linjw.demo.launchmodedemo/u0a61}
        Hist #1: ActivityRecord{17dd5b1 u0 linjw.demo.launchmodedemo/.FirstActivity t483}
          Intent { cmp=linjw.demo.launchmodedemo/.FirstActivity }
          ProcessRecord{245852c 7994:linjw.demo.launchmodedemo/u0a61}
        Hist #0: ActivityRecord{9f195f8 u0 linjw.demo.launchmodedemo/.FirstActivity t483}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10100000 cmp=linjw.demo.launchmodedemo/.FirstActivity }
          ProcessRecord{245852c 7994:linjw.demo.launchmodedemo/u0a61}
    ...
```

例如Hist #0这个Activity是从桌面点击应用图标进入的,所以它的Intent带android.intent.category.LAUNCHER这个category和FLAG_ACTIVITY\_LAUNCHED\_FROM\_HISTORY(0x00100000)、 FLAG\_RECEIVER\_FOREGROUND(0x10000000)这两个Flag,而其他的Activity都是我们通过点击按钮叫起的,我们的Intent里面没有带任何的category和Flag,所以只有cmp标识是哪个Activity。

我们可以通过Intent#setFlags(int) 设置Flag、Intent#addCategory(String) 添加category

有时候可以通过这个命令查看Activity的启动模式结合各种launchMode的作用来定位一些bug

# standard

standard是默认的启动模式,在没有配置android:launchMode的时候就会默认用这种启动模式,当然也可以显示指定为standard。

它的特点是每次都会启动一个新的Activity实例,我们连续点击第一个按钮两次然后再连续点击两次返回键,截图如下:

{% img /Android温故而知新-launchMode/standard1.png %}

{% img /Android温故而知新-launchMode/standard2.png %}

任务栈状态图如下:

{% img /Android温故而知新-launchMode/standard3.png %}



# singleTop

当launchMode是singleTop的时候，如果task栈的栈顶Activity和将要启动的Activity是同一个Activity的话,就不会再启动第二个Activity。我们将FirstActivity设为singleTop,在启动demo之后无论按第一个按钮多少次,任务栈里面都只会有一个FirstActivity。

{% img /Android温故而知新-launchMode/singleTop1.png %}

任务栈状态图如下:

{% img /Android温故而知新-launchMode/singleTop2.png %}


但是当任务栈的栈顶Activity和将要启动的Activity不是同一个Activity的时候,就会启动新的Activity,并将它压入栈顶而不管栈里面还有没有这个Activity:

{% img /Android温故而知新-launchMode/singleTop3.png %}

{% img /Android温故而知新-launchMode/singleTop4.png %}

任务栈状态图如下:

{% img /Android温故而知新-launchMode/singleTop5.png %}

# singleTask

我们将FirstActivity和SecondActivity的launchMode都设置为singleTask,启动demo之后先按“GOTO SECOND ACTIVITY”再按“GOTO FIRST ACTIVITY”,截图如下:

{% img /Android温故而知新-launchMode/singleTask1.png %}

我们可以看到在SecondActivity中启动FirstActivity,结果就返回了第一个Activity。如果这个时候再按返回键就会推出应用。

singleTask的作用就是在任务栈中寻找将要启动的Activity,如果找到的话就将它上面的Activity都弹出栈,直到它成为栈顶。

任务栈状态图如下:

{% img /Android温故而知新-launchMode/singleTask2.png %}

# singleInstance

官方文档的介绍是:

> 与“singleTask"”相同，只是系统不会将任何其他 Activity 启动到包含实例的任务中。 该 Activity 始终是其任务唯一仅有的成员。

就是说系统会为singleInstance Activity单独创建一个任务栈,这个任务栈里是这个Activity独占的,不会再压入其他的Activity。而且它是系统唯一的,当就是说系统会为singleInstance Activity已经存在于系统的某一任务栈中,就会直接跳到那个任务栈的Activity中,而不会新启动一个Activity。

我们将FirstActivity设为standard, SecondActivity设为singleInstance。启动demo之后先按“GOTO SECOND ACTIVITY”再按“GOTO FIRST ACTIVITY”。然后再一直按返回键到退出应用。截图如下:

{% img /Android温故而知新-launchMode/singleInstance1.png %}

{% img /Android温故而知新-launchMode/singleInstance2.png %}

任务栈状态图如下:

{% img /Android温故而知新-launchMode/singleInstance3.png %}


启动应用之后先点击“GOTO SECOND ACTIVITY”,这个时候系统会新建一个任务栈(Task 20)来放SecondActivity

在SecondActivity中再启动FirstActivity因为Task 20这个任务栈是SecondActivity独占的。所以不会在这个任务栈压入其他Activity,而会回到原来的任务栈上(Task 19)。又因为FirstActivity的launchMode是standard,所以不管原来的栈里面有没有FirstActivity,都会压入一个新的FirstActivity。

这个时候再按返回键就不是回到SecondActivity了,因为它在其他的任务栈里面,要先将当前任务栈清空。

这个时候按返回键会将当前的Activity弹出栈,于是就跳到了一开始的FirstActivity。之后再按返回键,因为Task 19这个任务栈空了,就会去到SecondActivity的栈,于是就去到了SecondActivity。最后再按返回键就会推出应用了。

要再次提醒需要注意的是singleInstance的Activity是系统唯一的,也就是说你在demo这里启动了这个SecondActivity的SecondActivity,然后按home键去到launcher启动其他应用,从其他应用再启动一个SecondActivity也是去到原来的SecondActivity