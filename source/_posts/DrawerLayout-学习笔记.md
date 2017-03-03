title: DrawerLayout 学习笔记
date: 2016-01-27 17:44:24
tags:
	- Android
	- 技术相关
---

DrawerLayout的使用十分简单，使用android.support.v4.widget.DrawerLayout标签即可，DrawerLayout的第一个子标签就是正文，其他布局都是抽屉布局（默认隐藏在屏幕外）。可以使用android:layout_gravity属性指定是隐藏在屏幕的左边或者右边。

## **一、使用DrawerLayout布局**

把activity_main.xml修改成下面的样子，这里声明了一个LinearLayout作为正文布局（DrawerLayout的第一个子标签），和其他两个LinearLayout布局作为抽屉布局（将android:layout_gravity设置为left或者right）：

```xml
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <android.support.v4.widget.DrawerLayout
            android:id="@+id/drawer_layout"
            android:layout_width="match_parent"
            android:layout_height="match_parent">

            <!-- 正文布局 -->
            <LinearLayout
                android:id="@+id/content"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:gravity="center_horizontal"
                android:orientation="horizontal">

                <TextView
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:text="content" />

            </LinearLayout>

            <!-- 从左边抽出来的布局 -->
            <LinearLayout
                android:id="@+id/drawer_left"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:layout_gravity="left"
                android:background="#FFFFFF"
                android:gravity="center_horizontal">

                <TextView
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:text="left" />
            </LinearLayout>

            <!-- 从右边抽出来的布局 -->
            <LinearLayout
                android:id="@+id/drawer_right"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:layout_gravity="right"
                android:background="#FFFFFF"
                android:gravity="center_horizontal">

                <TextView
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:text="right" />
            </LinearLayout>
        </android.support.v4.widget.DrawerLayout>
    </LinearLayout>
```

运行之后就长这个样子，可以用手指从左边或者右边把抽屉布局拖出来：
{% img /DrawerLayout-学习笔记/1.jpg %}

{% img /DrawerLayout-学习笔记/3.jpg %}

{% img /DrawerLayout-学习笔记/2.jpg %}

当然也能在代码里面调用openDrawer来显示：
```java 
    //打开左边的抽屉布局
    drawerLayout.openDrawer(Gravity.LEFT);
    //打开右边的抽屉布局
    drawerLayout.openDrawer(Gravity.RIGHT);
```
  
  
## **二、使用ActionBarDrawerToggle**
android提供了一个ActionBarDrawerToggle来简化DrawerLayout的操作，用法十分简单。
1.自定义一个ToolBar（可以查看我之前的一篇博文）
```xml
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar"
            android:background="#3F51B5"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize" />

        <android.support.v4.widget.DrawerLayout
            android:id="@+id/drawer_layout"
            android:layout_width="match_parent"
            android:layout_height="match_parent">

            <!-- 正文布局 -->
            <LinearLayout
                android:id="@+id/content"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:gravity="center_horizontal"
                android:orientation="horizontal">

                <TextView
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:text="content" />

            </LinearLayout>

            <!-- 从左边抽出来的布局 -->
            <LinearLayout
                android:id="@+id/drawer_left"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:layout_gravity="left"
                android:background="#FFFFFF"
                android:gravity="center_horizontal">

                <TextView
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:text="left" />
            </LinearLayout>

            <!-- 从右边抽出来的布局 -->
            <LinearLayout
                android:id="@+id/drawer_right"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:layout_gravity="right"
                android:background="#FFFFFF"
                android:gravity="center_horizontal">

                <TextView
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:text="right" />
            </LinearLayout>
        </android.support.v4.widget.DrawerLayout>
    </LinearLayout>
```

2.在acvitity代码中创建ActionBarDrawerToggle并重写下面的方法就可以了
- 在onPostCreate方法中调用ActionBarDrawerToggle.syncState()，如果不调用该方法，则ActionBarDrawerToggle不会显示 （onPostCreate在Activity完全加载成功之后调用，这个时候所有界面资源都已经创建和初始化完成）
- 在onOptionsItemSelected方法中调用ActionBarDrawerToggle.onOptionsItemSelected()。（原本我以为这里是用来实现按下按钮打开和关闭抽屉布局的，但经测试，就算不调用也能正常运行。）
```java
    public class MainActivity extends AppCompatActivity {
        ActionBarDrawerToggle mToggle;

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);

            setContentView(R.layout.activity_main);

            Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
            setSupportActionBar(toolbar);

            DrawerLayout drawerLayout = (DrawerLayout) findViewById(R.id.drawer_layout);
            mToggle = new ActionBarDrawerToggle(this, drawerLayout,toolbar,R.string.drawer_open, R.string.drawer_close);

            drawerLayout.setDrawerListener(mToggle);
        }

        @Override
        protected void onPostCreate(Bundle savedInstanceState) {
        	//onPostCreate在Activity完全加载成功之后调用
            //这个时候所有界面资源都已经创建和初始化完成
            super.onPostCreate(savedInstanceState);

            //如果不调用该方法，则ActionBarDrawerToggle不会显示
            mToggle.syncState();
        }

        @Override
        public boolean onOptionsItemSelected(MenuItem item) {
        	//原本我以为这里是用来实现按下按钮打开和关闭抽屉布局的
            //但经测试，就算不调用也能正常运行。
            return mToggle.onOptionsItemSelected(item) || super.onOptionsItemSelected(item);
        }
    }
```

运行可以得到下面这效果，可以使用控制按钮来打开和关闭左边的抽屉布局，那个控制按钮还实现了一种特别酷炫的动画：
{% img /DrawerLayout-学习笔记/4.jpg %}

{% img /DrawerLayout-学习笔记/5.jpg %}

{% img /DrawerLayout-学习笔记/6.jpg %}
  
  
## **三、将抽屉布局的层级填到toolbar之上**
知乎的安卓app也使用了DrawerLayout，但它的抽屉布局显示的时候是位于toolbar之上的。我们只要把toolbar标签放到内容布局（DrawerLayout的第一个子标签）里面就能实现这样的效果了。
```xml
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <android.support.v4.widget.DrawerLayout
            android:id="@+id/drawer_layout"
            android:layout_width="match_parent"
            android:layout_height="match_parent">

            <!-- 正文布局 -->
            <LinearLayout
                android:id="@+id/content"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:gravity="center_horizontal"
                android:orientation="horizontal">

                <!-- 把toolbar放到这里，使抽屉布局层级比toolbar高 -->
                <android.support.v7.widget.Toolbar
                    android:id="@+id/toolbar"
                    android:background="#3F51B5"
                    android:layout_width="match_parent"
                    android:layout_height="?attr/actionBarSize" />

                <TextView
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:text="content" />

            </LinearLayout>

            <!-- 从左边抽出来的布局 -->
            <LinearLayout
                android:id="@+id/drawer_left"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:layout_gravity="left"
                android:background="#FFFFFF"
                android:gravity="center_horizontal">

                <TextView
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:text="left" />
            </LinearLayout>

            <!-- 从右边抽出来的布局 -->
            <LinearLayout
                android:id="@+id/drawer_right"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:layout_gravity="right"
                android:background="#FFFFFF"
                android:gravity="center_horizontal">

                <TextView
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:text="right" />
            </LinearLayout>
        </android.support.v4.widget.DrawerLayout>
    </LinearLayout>
```

运行效果如下：

{% img /DrawerLayout-学习笔记/7.jpg %}