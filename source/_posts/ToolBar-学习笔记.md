title: ToolBar 学习笔记
date: 2016-01-25 14:56:42
tags:
    - Android
    - 技术相关
---

虽然android studio在新建项目的时候就可以创建一个默认带有ToolBar的MainActivity，但是抱着学习学全套的精神，我们就从一个没有Activity的空项目入手，一步一步把ToolBar学透。

## **一、创建基础ToolBar**

创建完一个不带Activity的空项目之后的第一步就是创建自己的Activity了，注意这个Activity必须继承AppCompatActivity（ActionBarActivity已经被废弃了）。

```java
    public class MainActivity extends AppCompatActivity{
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main_activity);
        }
    }
```

如果不能import android.support.v7.app.AppCompatActivity;的话就在build.gradle（Module：App）的dependencies里面添加

    compile 'com.android.support:appcompat-v7:23.1.1'

这个R.layout.main_activity也是要自己创建的，我这里创建了一个只有一个TextView的LinearLayout

```xml
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Hello World"/>
    </LinearLayout>
```

当然，不要忘了在manifests里面注册MainActivity
```xml
	<activity android:name=".MainActivity"
            android:label="@string/app_name"
            android:theme="@style/AppTheme">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
	</activity>
```
然后运行程序就能就是下面这个样子：
{% img /ToolBar-学习笔记/1.jpg %}

## **二、自定义ToolBar样式**

因为用上面的方法创建的ToolBar是Activity自带的，在需要自定义样式的时候不够灵活，所以我们把它去掉，换成我们自己创建的ToolBar。

可以在@style/AppTheme添加如下item：

```xml
	<!-- 去掉Activity自带的ToolBar -->
	<item name="windowNoTitle">true</item>
```

这个时候再运行项目，就会发现ToolBar已经不见了。然后我们自己在R.layout.main_activity里面自己声明一个ToolBar:

```xml
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            android:theme="@style/ToolBarTheme"/>

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Hello World"/>

    </LinearLayout>
```

ToolBar的Theme分离出来放在@style/Theme,可以在这里自定义ToolBar的样式:
```xml
 	<style name="ToolBarTheme" parent="Theme.AppCompat">
        <!-- 设置ToolBar底色 -->
        <item name="android:background">#3F51B5</item>
        <!-- 设置字体颜色 -->
        <item name="android:textColorPrimary">#FFFFFF</item>
    </style>
```

最后在MainActivity里调用setSupportActionBar，顺便设置调用ToolBar的方法设置一些属性。注意这里的ToolBar是导的android.support.v7.widget.Toolbar的包：

```java
    public class MainActivity extends AppCompatActivity{
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main_activity);

            Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);

            toolbar.setTitle("title"); //setTile方法必须在setSupportActionBar之前调用
            toolbar.setSubtitle("subtitle");
            toolbar.setLogo(R.mipmap.ic_launcher);

            setSupportActionBar(toolbar);

            //给Navigate按钮加一个默认的返回箭头
            //也可以用toolbar.setNavigationIcon()直接给Navigate设置icon
        	getSupportActionBar().setDisplayHomeAsUpEnabled(true);
        }
    }
```

运行起来之后长这个样子:
{% img /ToolBar-学习笔记/2.jpg %}

## **三、添加菜单按钮**

首先创建一个menu/main.xml:

```xml
    <menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">
        <item
            android:id="@+id/btn_ico"
            android:title="btn_ico"
            android:icon="@mipmap/ic_launcher"
            app:showAsAction="ifRoom" />
        <item
            android:id="@+id/btn_search"
            android:title="btn_search"
            app:actionViewClass="android.support.v7.widget.SearchView"
            app:showAsAction="ifRoom" />
        <item
            android:id="@+id/btn_1"
            android:title="btn_1"
            app:showAsAction="never" />
        <item
            android:id="@+id/btn_2"
            android:title="btn_2"
            app:showAsAction="never"/>
    </menu>
```

这里的app:showAsAction就是按钮出现的位置，它可以填入以下的值:
1. always：这个值会使菜单项一直显示在ToolBar上。
2. ifRoom：如果有足够的空间，这个值会使菜单项显示在ToolBar上。
3. never：这个值使菜单项永远都不出现在ToolBar上。
4. withText：这个值使菜单项和它的图标，菜单文本一起显示。 


然后在MainActivity覆盖onCreateOptionsMenu方法：
```java
    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.main, menu);
        return true;
    }
```

运行程序之后长这个样子：
{% img /ToolBar-学习笔记/3.jpg %}

## **四、监听菜单按钮消息**

监听这些菜单按钮的消息有两种方法
1.覆盖Activity的onOptionsItemSelected方法

```java
    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        Toast.makeText(this, item.getTitle(), Toast.LENGTH_SHORT).show();
        return true;
    }
```

2.调用ToolBar的setOnMenuItemClickListener方法

```java
    //setOnMenuItemClickListener方法必须在setSupportActionBar之后调用才有效
    toolbar.setOnMenuItemClickListener(new Toolbar.OnMenuItemClickListener() {
        @Override
        public boolean onMenuItemClick(MenuItem item) {
            Toast.makeText(MainActivity.this, item.getTitle(), Toast.LENGTH_SHORT).show();
            return false;
        }
    });
```

这个时候Navigation按钮的消息也是在上面两个回调方法中处理的，当然也能直接调用ToolBar的setNavigationOnClickListener方法设置，这样只有在该listener方法里面才会收到Navigation按钮的消息：
```java
    //setNavigationOnClickListener方法必须在setSupportActionBar之后调用才有效
    toolbar.setNavigationOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            finish();
        }
    });
```

而ToolBar上的搜索按钮可以这样设置它的回调:
```java
    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.main, menu);
        MenuItem item = menu.findItem(R.id.btn_search);
        SearchView searcher = (SearchView) item.getActionView();
        searcher.setOnQueryTextListener(new SearchView.OnQueryTextListener() {
            @Override
            public boolean onQueryTextSubmit(String query) {
                Toast.makeText(MainActivity.this, query, Toast.LENGTH_SHORT).show();
                return false;
            }

            @Override
            public boolean onQueryTextChange(String newText) {
                return false;
            }
        });
        return true;
    }
```

## **五、设置弹出菜单的样式**

弹出菜单的样式默认是和ToolBar的样式一样的:
{% img /ToolBar-学习笔记/4.jpg %}

但也可以通过下面的方法自定义

1.新建样式PopupTheme

```xml
	<style name="PopupTheme" parent="Theme.AppCompat">
        <item name="android:background">#FFFFFF</item>
        <item name="android:textColorPrimary">#000000</item>
    </style>
```

2.设置ToolBar的PopupTheme,注意这里的前缀是app:
```xml
	<android.support.v7.widget.Toolbar
        android:id="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="?attr/actionBarSize"
        android:theme="@style/ToolBarTheme"
        app:popupTheme="@style/PopupTheme"/>
```

效果如下：
{% img /ToolBar-学习笔记/5.jpg %}
