title: Android多用户的一些坑
date: 2022-08-04 21:27:08
tags:
    - 技术相关
    - Android
---


最近关于多用户功能报了几个bug,我觉得蛮有意思的这里记录一下。

起因是是测试报了打开了多用户功能并且切到另外一个用户之后,系统功能异常。调试发现我们的中间层服务启动了两个进程:

```
system         6074   2524 14649520 96244 SyS_epoll_wait      0 S me.linjw.demo.multiuser
u10_system     7991   2524 14582664 94148 SyS_epoll_wait      0 S me.linjw.demo.multiuser
```

从上面可以看出me.linjw.demo.multiuser这个应用分别在USER为system和u10\_system各起了一个进程。查找了下资料发现正常情况下一个应用进程的确是不能跨用户访问的,会在不同的用户下启动新的进程。

# android:singleUser配置

由于历史代码原因,我们系统上的硬件操控接口的确不支持多个进程访问,也不好修改。只能靠我们应用做规避。使用ps命令查看了下,发现像system\_server这样的系统服务在多用户下也只有一个进程。谷歌应该会考虑到这种多用户共用一个进程的场景,于是在开发者文档中找到多用户相关[文档](https://source.android.google.cn/devices/tech/admin/multiuser-apps?hl=zh-cn):

> 如需将应用识别为单例，请将 android:singleUser=”true” 添加至 Android 清单中的服务、接收器或提供程序。

由于现象是多个进程，我下意识认为这个singleUser配置是针对应用的,所以在AndroidManifest.xml的application标签中配置上去,但是发现没有作用:

```
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:singleUser="true"
        android:theme="@style/Theme.MultiuserDemo">
```

由于文档里面没有其他信息，从网上搜索找到的类似需要系统签名、android:persistent需要为true这样的信息也确认过没有效果。本来还怀疑是我们framework里面做了什么修改导把这块改坏了。

于是去看这部分相关的framework源码,主要逻辑在[ActiveServices.retrieveServiceLocked](https://cs.android.com/android/platform/superproject/+/android-11.0.0_r9:frameworks/base/services/core/java/com/android/server/am/ActiveServices.java;l=2342)里面,子用户里启动服务的时候会去通过isSingleton判断是否使用主用户的进程,如果是的话就使用主用户的进程,不需要新启动一个进程::

```
// frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
private ServiceLookupResult retrieveServiceLocked(Intent service,
        String instanceName, String resolvedType, String callingPackage,
        int callingPid, int callingUid, int userId,
        boolean createIfNeeded, boolean callingFromFg, boolean isBindExternal,
        boolean allowInstant) {
    ...
    // 这里在不同userId下查询出来的rInfo就是不一样的
    ResolveInfo rInfo = mAm.getPackageManagerInternalLocked().resolveService(service,
            resolvedType, flags, userId, callingUid);
    ServiceInfo sInfo = rInfo != null ? rInfo.serviceInfo : null;
    ...
    // userId不为0代表子用户下运行
    if (userId > 0) {
        if (mAm.isSingleton(sInfo.processName, sInfo.applicationInfo,
                sInfo.name, sInfo.flags)
                && mAm.isValidSingletonCall(callingUid, sInfo.applicationInfo.uid)) {
            // 如果组件isSingleton判断为true
            // 则将userId改成0,并使用clearCallingIdentity清除调用进程的用户信息,重新查询
            // 则查出来的rInfoForUserId0为主用户的缓存  
            userId = 0;
            smap = getServiceMapLocked(0);
            // Bypass INTERACT_ACROSS_USERS permission check
            final long token = Binder.clearCallingIdentity();
            try {
                ResolveInfo rInfoForUserId0 = mAm.getPackageManagerInternalLocked().resolveService(service,
                                resolvedType, flags, userId, callingUid);
                if (rInfoForUserId0 == null) {
                    Slog.w(TAG_SERVICE,
                            "Unable to resolve service " + service + " U=" + userId
                                    + ": not found");
                    return null;
                }
                // 然后用这个rInfoForUserId0.serviceInfo去替换之前查出来的rInfo.serviceInfo,保证多用户下都用主用户下的同一个进程
                sInfo = rInfoForUserId0.serviceInfo;
            } finally {
                Binder.restoreCallingIdentity(token);
            }
        }
        sInfo = new ServiceInfo(sInfo);
        sInfo.applicationInfo = mAm.getAppInfoForUser(sInfo.applicationInfo, userId);
    }
    ...
}

```

判断是否在多用户下只启动单个进程主要靠[isSingleton](https://cs.android.com/android/platform/superproject/+/android-11.0.0_r9:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=15103)这个方法:

```
// frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
boolean isSingleton(String componentProcessName, ApplicationInfo aInfo,
        String className, int flags) {
    boolean result = false;
    // For apps that don't have pre-defined UIDs, check for permission
    if (UserHandle.getAppId(aInfo.uid) >= FIRST_APPLICATION_UID) {
        if ((flags & ServiceInfo.FLAG_SINGLE_USER) != 0) {
            if (ActivityManager.checkUidPermission(
                    INTERACT_ACROSS_USERS,
                    aInfo.uid) != PackageManager.PERMISSION_GRANTED) {
                ComponentName comp = new ComponentName(aInfo.packageName, className);
                String msg = "Permission Denial: Component " + comp.flattenToShortString()
                        + " requests FLAG_SINGLE_USER, but app does not hold "
                        + INTERACT_ACROSS_USERS;
                Slog.w(TAG, msg);
                throw new SecurityException(msg);
            }
            // Permission passed
            result = true;
        }
    } else if ("system".equals(componentProcessName)) {
        result = true;
    } else if ((flags & ServiceInfo.FLAG_SINGLE_USER) != 0) {
        // Phone app and persistent apps are allowed to export singleuser providers.
        result = UserHandle.isSameApp(aInfo.uid, PHONE_UID)
                || (aInfo.flags & ApplicationInfo.FLAG_PERSISTENT) != 0;
    }
    if (DEBUG_MU) Slog.v(TAG_MU,
            "isSingleton(" + componentProcessName + ", " + aInfo + ", " + className + ", 0x"
            + Integer.toHexString(flags) + ") = " + result);
    return result;
}
```
打开DEBUG\_MU之后查看打印,发现singleUser是按组件来配置的:

```
08-03 13:45:20.092  3289  4023 V ActivityManager_MU: isSingleton(me.linjw.demo.multiuser, ApplicationInfo{417a0ea me.linjw.demo.multiuser}, me.linjw.demo.multiuser.TestService, 0x0) = false
```

所以应该在service里面配置:

```
<service
    android:name=".TestService"
    android:exported="true"
    android:singleUser="true">
```


实际上如果我一开始看到是[英文文档](https://source.android.google.cn/devices/tech/admin/multiuser-apps),应该就不会出现这样的误解了:

> To identify an app as a singleton, add android:singleUser=”true” to your service, receiver, or provider in the Android manifest.

# android:exported被自动关闭

修改完成自检通过,开开心心上传代码原本以为问题已经解决。没想到一天之后另外一个客户的软件报了连接不上我们的Service的问题:

```
08-03 12:13:22.108  3185  3557 W ActivityManager: Permission Denial: Accessing service me.linjw.demo.multiuser/.TestService from pid=5994, uid=10055 that is not exported from uid 1000
08-03 12:13:22.112  5994  5994 E AndroidRuntime: Caused by: java.lang.SecurityException: Not allowed to bind to service Intent { act=me.linjw.multiuser.service pkg=me.linjw.demo.multiuser }
```

从日志上来看TestService没有export,但是从AndroidManifest.xml上看android:exported的确设置成true了。而且尝试把android:singleUser改成fasle又能连上。这就意味着android:singleUser陪着会影响到android:exported。


但这里又有个问题,当初我修改完android:singleUser="true"之后是有自检通过的,如果exported为false,那自检为什么能通过?


最终排查发现我们的应用设置了sharedUserId声明为系统进程:

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="me.linjw.demo.multiuser"
    android:sharedUserId="android.uid.system">
```

当初自检的时候的那个应用的sharedUserId也是android.uid.system,所以相当于他们是同一个应用,可以相互访问exported为false的组件。


原因排查清楚了,那要怎么解决呢?还是只能从framework源码开始翻起,先去[报错的地方](https://cs.android.com/android/platform/superproject/+/android-11.0.0_r9:frameworks/base/services/core/java/com/android/server/am/ActiveServices.java;l=2532)看起,找找为什么exported会被自动改成false:

```
// frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
private ServiceLookupResult retrieveServiceLocked(Intent service,
          String instanceName, String resolvedType, String callingPackage,
          int callingPid, int callingUid, int userId,
          boolean createIfNeeded, boolean callingFromFg, boolean isBindExternal,
          boolean allowInstant) {
      ServiceRecord r = null;
      ...
      if (mAm.checkComponentPermission(r.permission,
              callingPid, callingUid, r.appInfo.uid, r.exported) != PERMISSION_GRANTED) {
          if (!r.exported) {
              Slog.w(TAG, "Permission Denial: Accessing service " + r.shortInstanceName
                      + " from pid=" + callingPid
                      + ", uid=" + callingUid
                      + " that is not exported from uid " + r.appInfo.uid);
              return new ServiceLookupResult(null, "not exported from uid "
                      + r.appInfo.uid);
          }
          Slog.w(TAG, "Permission Denial: Accessing service " + r.shortInstanceName
                  + " from pid=" + callingPid
                  + ", uid=" + callingUid
                  + " requires " + r.permission);
          return new ServiceLookupResult(null, r.permission);
      }
      ...
}
```

从这里看r的exported为false导致了这个异常,我们需要在retrieveServiceLocked里面一路追踪r的exported是怎么被singleUser影响的,由于这部分代码比较曲折我也找了很久才找到关键代码。

在[PackageManagerService](https://cs.android.com/android/platform/superproject/+/android-11.0.0_r9:frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java;l=11783)扫描应用信息的时候,会判断SCAN\_AS\_PRIVILEGED这个flag,如果没有设置就会执行markNotActivitiesAsNotExportedIfSingleUser

```
// frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java
if ((scanFlags & SCAN_AS_PRIVILEGED) == 0) {
    parsedPackage
            .markNotActivitiesAsNotExportedIfSingleUser();
}
```


[markNotActivitiesAsNotExportedIfSingleUser](https://cs.android.com/android/platform/superproject/+/android-11.0.0_r9:frameworks/base/services/core/java/com/android/server/pm/parsing/pkg/PackageImpl.java;l=454;drc=da9304867216e65874eae45db56d8d3fd3674105;bpv=1;bpt=1)顾名思义,就会在配置了SingleUser的时候去修改exported,实际上它里面除了Activity不修改,其他的三个组件都修改了:

```
public PackageImpl markNotActivitiesAsNotExportedIfSingleUser() {
    // ignore export request for single user receivers
    int receiversSize = receivers.size();
    for (int index = 0; index < receiversSize; index++) {
        ParsedActivity receiver = receivers.get(index);
        if ((receiver.getFlags() & ActivityInfo.FLAG_SINGLE_USER) != 0) {
            receiver.setExported(false);
        }
    }

    // ignore export request for single user services
    int servicesSize = services.size();
    for (int index = 0; index < servicesSize; index++) {
        ParsedService service = services.get(index);
        if ((service.getFlags() & ActivityInfo.FLAG_SINGLE_USER) != 0) {
            service.setExported(false);
        }
    }

    // ignore export request for single user providers
    int providersSize = providers.size();
    for (int index = 0; index < providersSize; index++) {
        ParsedProvider provider = providers.get(index);
        if ((provider.getFlags() & ActivityInfo.FLAG_SINGLE_USER) != 0) {
            provider.setExported(false);
        }
    }

    return this;
}
```

那问题就在于我们的应用没有携带SCAN\_AS\_PRIVILEGED,所以在singleUser为true的时候exported会被改成false。那我们要怎么带上这个flag呢?搜索了下[资料](https://wrlus.com/android-security/system-apps-and-cve-2020-0391/)发现这个flag代表着特权应用,只要预装到下面目录的就能成为特权应用

```
/system/framework
/system/priv-app
/vendor/priv-app
/odm/priv-app
/product/priv-app
/system_ext/priv-app
```

最终将预装路径从/syste/app改到/syste/priv-app解决问题。

# 全局浮动框在子用户不显示

没想到过了两天又报了另外一个问题,我们通过WindowManager.addView添加的全局浮动框在子用户不显示,在主用户是好的。又踩了一个隐藏的坑。

既然不显示,那么首先考虑是不是addView失败了,于是用dumpsys window看看有没有add成功:

```
console:/ # dumpsys window | grep me.linjw.demo.multiuser
    mPackageName=me.linjw.demo.multiuser
  Window #0 Window{a03fb0 u0 me.linjw.demo.multiuser}:
    mOwnerUid=1000 showForAllUsers=false package=me.linjw.demo.multiuser appop=SYSTEM_ALERT_WINDOW
```

从打印上来看是add成功的,但是里面有个showForAllUsers引起了我的注意,大概猜测是addView的时候有个showForAllUsers的flag没有设置,于是在源码里面搜索还真找到了:

```
// android/view/WindowManager.java
@SystemApi
@RequiresPermission(permission.INTERNAL_SYSTEM_WINDOW)
public static final int SYSTEM_FLAG_SHOW_FOR_ALL_USERS = 0x00000010;
```

但是它的值和FLAG\_NOT\_TOUCHABLE重复了:

```
public static final int FLAG_NOT_TOUCHABLE      = 0x00000010;
```

于是从搜索了下它,发现需要设置到WindowManager.LayoutParams.privateFlags而不是WindowManager.LayoutParams.flags

可惜的是无论是privateFlags还是SYSTEM\_FLAG\_SHOW\_FOR\_ALL\_USERS都是系统api,所以只能用反射去设置:

```
private int SYSTEM_FLAG_SHOW_FOR_ALL_USERS = 0x00000010;

// 多用户下需要设置这个flag才能在其他用户显示
Field privateFlags = null;
try {
    privateFlags = WindowManager.LayoutParams.class.getDeclaredField("privateFlags");
    privateFlags.set(wmParams, SYSTEM_FLAG_SHOW_FOR_ALL_USERS);
} catch (Exception e) {
    Log.e("testtest", "err", e);
}
```

设置之后的确在子用户下也显示成功了,用dumpsys window查看showForAllUsers也变成了true:

```
console:/ # dumpsys window | grep me.linjw.demo.multiuser
    mPackageName=me.linjw.demo.multiuser
  Window #0 Window{3655208 u0 me.linjw.demo.multiuser}:
    mOwnerUid=1000 showForAllUsers=true package=me.linjw.demo.multiuser appop=SYSTEM_ALERT_WINDOW
```

# 结尾吐槽

这系列问题前前后后差不多一个月才弄完,framework部分源码的源码看得人都晕了,也不知道还会不会有其他意料之外的坑。当个安卓应用开发太难了...