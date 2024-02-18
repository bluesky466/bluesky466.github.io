title: 安卓RRO机制
date: 2024-02-18 22:45:14
tags:
  - 技术相关
  - Android
---

年前和组内的小伙伴讨论过一个需求的RRO实现方案。我其实之前对RRO也只是处于大概了解的程度,并没有实际去操作过,趁着过年这段时间有空也写了个demo实际验证了下。

由于[官方文档](https://source.android.com/docs/core/architecture/rros?hl=zh-cn)实际上对整个RRO机制已经讲的比较清楚了,我这里只做一些补充。

# RRO的默认状态

根据官方文档的介绍普通的RRO默认是停用状态的,类似国内的系统主题包其实下载安装之后是没有启用的,需要程序用[OverlayManager.setEnable](https://source.android.com/docs/core/architecture/rros?hl=zh-cn#enabling-disabling-overlays)去启动它。

但我们是需要预装默认启用的,有下面两种方式可以实现

### 静态 RRO

第一种是使用[静态 RRO](https://source.android.com/docs/core/architecture/rros?hl=zh-cn#static-rros),即将manifest里面的`android:isStatic`设置成true。

这种方式**预装**的overlay包会默认启用,而且不能用命令或者程序去禁用它(如果是用pm install去安装的话不会生效,还是默认停用需要用命令或者程序去启用)

这种方式比较适合运行时不会改变的客制化需求(俗称换皮)。

### OverlayConfig

第二种是使用[OverlayConfig](https://source.android.com/docs/core/architecture/rros?hl=zh-cn#using-overlayconfig),在机器的`/{partition}/overlay/config/config.xml`里面添加配置:

```xml
<config>
    <merge path="OEM-common-rros-config.xml" />
    <overlay package="com.oem.overlay.device" mutable="false" enabled="true" />
    <overlay package="com.oem.green.theme" enabled="true" />
</config>
```

这里的`enabled`配置默认是否启用(默认为false),`mutable`配置运行时是否可修改(默认为true),如果配置成`mutable="false" enabled="true"`则和`android:isStatic`设置成true使用静态 RRO效果一样。

然后需要将overlay apk预装到`/{partition}/overlay/`目录下,由于OverlayConfigParser是在xml上级的overlay里面扫描overlay apk,如果不预装在这个路径的话就会报错:

```
E Zygote  : System zygote died with fatal exception
E Zygote  : java.lang.ExceptionInInitializerError
E Zygote  :      at java.lang.Class.classForName(Native Method)
E Zygote  :      at java.lang.Class.forName(Class.java:454)
E Zygote  :      at com.android.internal.os.ZygoteInit.preloadClasses(ZygoteInit.java:301)
E Zygote  :      at com.android.internal.os.ZygoteInit.preload(ZygoteInit.java:140)
E Zygote  :      at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:889)
E Zygote  : Caused by: java.lang.IllegalStateException: overlay me.linjw.demo.overlay.overlay not present in partition /vendor/overlay in /vendor/overlay/config/config.xml at START_TAG (empty) <overlay package='me.linjw.demo.overlay.overlay' enabled='true'>@2:66 in java.io.FileReader@583e450
E Zygote  :      at com.android.internal.content.om.OverlayConfigParser.parseOverlay(OverlayConfigParser.java:372)
E Zygote  :      at com.android.internal.content.om.OverlayConfigParser.readConfigFile(OverlayConfigParser.java:249)
E Zygote  :      at com.android.internal.content.om.OverlayConfigParser.getConfigurations(OverlayConfigParser.java:220)
E Zygote  :      at com.android.internal.content.om.OverlayConfig.<init>(OverlayConfig.java:152)
E Zygote  :      at com.android.internal.content.om.OverlayConfig.getZygoteInstance(OverlayConfig.java:218)
E Zygote  :      at android.content.res.AssetManager.createSystemAssetsInZygoteLocked(AssetManager.java:252)
E Zygote  :      at android.content.res.AssetManager.getSystem(AssetManager.java:276)
E Zygote  :      at android.content.res.Resources.<init>(Resources.java:347)
E Zygote  :      at android.content.res.Resources.getSystem(Resources.java:236)
E Zygote  :      at com.android.internal.telephony.GsmAlphabet.enableCountrySpecificEncodings(GsmAlphabet.java:1090)
E Zygote  :      at com.android.internal.telephony.GsmAlphabet.<clinit>(GsmAlphabet.java:1495)
E Zygote  :      ... 5 more
```

`partition`可以是下面的值,如果有多个overlay apk对同一个资源做overlay,优先级从低到高:

- system
- vendor
- odm
- oem
- product
- system_ext

我这里的和[官方文档](https://source.android.com/docs/core/architecture/rros?hl=zh-cn#overlay-precedence)里面写的优先级odm、oem反了,这是因为我看安卓13的实现代码里面定义是这样的:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/core/java/com/android/internal/content/om/OverlayConfig.java;l=132
partitions = new ArrayList<>(
                      PackagePartitions.getOrderedPartitions(OverlayPartition::new));

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/core/java/android/content/pm/PackagePartitions.java
/**
 * The list of all system partitions that may contain packages in ascending order of
 * specificity (the more generic, the earlier in the list a partition appears).
 */
private static final ArrayList<SystemPartition> SYSTEM_PARTITIONS =
        new ArrayList<>(Arrays.asList(
                new SystemPartition(Environment.getRootDirectory(),
                        PARTITION_SYSTEM, Partition.PARTITION_NAME_SYSTEM,
                        true /* containsPrivApp */, false /* containsOverlay */),
                new SystemPartition(Environment.getVendorDirectory(),
                        PARTITION_VENDOR, Partition.PARTITION_NAME_VENDOR,
                        true /* containsPrivApp */, true /* containsOverlay */),
                new SystemPartition(Environment.getOdmDirectory(),
                        PARTITION_ODM, Partition.PARTITION_NAME_ODM,
                        true /* containsPrivApp */, true /* containsOverlay */),
                new SystemPartition(Environment.getOemDirectory(),
                        PARTITION_OEM, Partition.PARTITION_NAME_OEM,
                        false /* containsPrivApp */, true /* containsOverlay */),
                new SystemPartition(Environment.getProductDirectory(),
                        PARTITION_PRODUCT, Partition.PARTITION_NAME_PRODUCT,
                        true /* containsPrivApp */, true /* containsOverlay */),
                new SystemPartition(Environment.getSystemExtDirectory(),
                        PARTITION_SYSTEM_EXT, Partition.PARTITION_NAME_SYSTEM_EXT,
                        true /* containsPrivApp */, true /* containsOverlay */)));

/**
 * Returns a list in which the elements are products of the specified function applied to the
 * list of {@link #SYSTEM_PARTITIONS} in increasing specificity order.
 */
public static <T> ArrayList<T> getOrderedPartitions(
        @NonNull Function<SystemPartition, T> producer) {
    final ArrayList<T> out = new ArrayList<>();
    for (int i = 0, n = SYSTEM_PARTITIONS.size(); i < n; i++) {
        final T v = producer.apply(SYSTEM_PARTITIONS.get(i));
        if (v != null)  {
            out.add(v);
        }
    }
    return out;
}
```

# overlay命令

根据[官方文档](https://source.android.com/docs/core/architecture/rros?hl=zh-cn#debugging-overlays)在调试阶段可以用shell命令去调试overlay。有时候对具体命令的使用有疑问的话可以直接看它的实现代码:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/om/OverlayManagerShellCommand.java;l=61

public int onCommand(@Nullable final String cmd) {
    if (cmd == null) {
        return handleDefaultCommands(cmd);
    }
    final PrintWriter err = getErrPrintWriter();
    try {
        switch (cmd) {
            case "list":
                return runList();
            case "enable":
                return runEnableDisable(true);
            case "disable":
                return runEnableDisable(false);
            case "enable-exclusive":
                return runEnableExclusive();
            case "set-priority":
                return runSetPriority();
            case "lookup":
                return runLookup();
            case "fabricate":
                return runFabricate();
            default:
                return handleDefaultCommands(cmd);
        }
    } catch (IllegalArgumentException e) {
        err.println("Error: " + e.getMessage());
    } catch (RemoteException e) {
        err.println("Remote exception: " + e);
    }
    return -1;
}
```

例如`cmd overlay list`命令列出了各个overlay apk的启用状态,`[x]`是已经启用,`[ ]`是停用:

```
me.linjw.demo.overlay1.app
[ ] me.linjw.demo.overlay1.overlay

me.linjw.demo.overlay2.app
[x] me.linjw.demo.overlay2.overlay

com.android.connectivity.resources
--- com.rockchip.networkstack.tethering.nokeepalive.overlay
```

而`---`则是代表这个overlay包处于不可用状态无法使用`cmd overlay enable {package}`或者`cmd overlay disable {package}`去启用或者停用它。这种情况会出现在目标包未预装、没有找到需要overlay的资源等情况:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/om/OverlayManagerShellCommand.java;l=193
private void printListOverlay(PrintWriter out, OverlayInfo oi) {
    String status;
    switch (oi.state) {
        case OverlayInfo.STATE_ENABLED_IMMUTABLE:
        case OverlayInfo.STATE_ENABLED:
            status = "[x]";
            break;
        case OverlayInfo.STATE_DISABLED:
            status = "[ ]";
            break;
        default:
            status = "---";
            break;
    }
    out.println(String.format("%s %s", status, oi.getOverlayIdentifier()));
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/core/java/android/content/om/OverlayInfo.java;l=45
@IntDef(prefix = "STATE_", value = {
        STATE_UNKNOWN,
        STATE_MISSING_TARGET,
        STATE_NO_IDMAP,
        STATE_DISABLED,
        STATE_ENABLED,
        STATE_ENABLED_IMMUTABLE,
        // @Deprecated STATE_TARGET_IS_BEING_REPLACED,
        STATE_OVERLAY_IS_BEING_REPLACED,
})
/** @hide */
@Retention(RetentionPolicy.SOURCE)
public @interface State {}

/**
 * An internal state used as the initial state of an overlay. OverlayInfo
 * objects exposed outside the {@link
 * com.android.server.om.OverlayManagerService} should never have this
 * state.
 *
 * @hide
 */
public static final int STATE_UNKNOWN = -1;

/**
 * The target package of the overlay is not installed. The overlay cannot be enabled.
 *
 * @hide
 */
public static final int STATE_MISSING_TARGET = 0;

/**
 * Creation of idmap file failed (e.g. no matching resources). The overlay
 * cannot be enabled.
 *
 * @hide
 */
public static final int STATE_NO_IDMAP = 1;

/**
 * The overlay is currently disabled. It can be enabled.
 *
 * @see IOverlayManager#setEnabled
 * @hide
 */
public static final int STATE_DISABLED = 2;

/**
 * The overlay is currently enabled. It can be disabled.
 *
 * @see IOverlayManager#setEnabled
 * @hide
 */
public static final int STATE_ENABLED = 3;

/**
 * The target package is currently being upgraded or downgraded; the state
 * will change once the package installation has finished.
 * @hide
 *
 * @deprecated No longer used. Caused invalid transitions from enabled -> upgrading -> enabled,
 * where an update is propagated when nothing has changed. Can occur during --dont-kill
 * installs when code and resources are hot swapped and the Activity should not be relaunched.
 * In all other cases, the process and therefore Activity is killed, so the state loop is
 * irrelevant.
 */
@Deprecated
public static final int STATE_TARGET_IS_BEING_REPLACED = 4;

/**
 * The overlay package is currently being upgraded or downgraded; the state
 * will change once the package installation has finished.
 * @hide
 */
public static final int STATE_OVERLAY_IS_BEING_REPLACED = 5;

/**
 * The overlay package is currently enabled because it is marked as
 * 'immutable'. It cannot be disabled but will change state if for instance
 * its target is uninstalled.
 * @hide
 */
@Deprecated
public static final int STATE_ENABLED_IMMUTABLE = 6;
```

然后可以用`lookup`命令查看启用、停用overlay包的情况下最终读取到的资源值是什么:

```
cmd overlay lookup me.linjw.demo.overlay.app me.linjw.demo.overlay.app:string/app_name
```

# idmap

之前我写过一系列[博客](https://blog.islinjw.cn/2019/05/18/%E5%8F%AF%E8%83%BD%E6%98%AF%E5%85%A8%E7%BD%91%E8%AE%B2%E6%9C%80%E7%BB%86%E7%9A%84%E5%AE%89%E5%8D%93resources-arsc%E8%A7%A3%E6%9E%90%E6%95%99%E7%A8%8B-%E4%B8%80/)探索过安卓的资源机制,实际上安卓是通过一个int的资源id去resources.arsc里面查询的资源,如果Overlay apk里面的id和目标apk的资源id不一致要怎么处理?

其实安卓是通过idmap机制实现的,在`/data/resource-cache/`目录下保存有各个overlay应用的idmap映射文件,可以用`idmap2 dump --idmap-path {file}`命令去打印映射关系:

```
idmap2 dump --idmap-path /data/resource-cache/system_ext@overlay@OverlayDemo.apk@idmap
Paths:
    target path  : /system_ext/app/OverlayDemo/OverlayDemo.apk
    overlay path : /system_ext/overlay/OverlayDemoOverlay.apk
Debug info:
    W failed to find resource 'string/app_name2'
Mapping:
    0x7f030000 -> 0x7f010000 (string/app_name -> string/app_name)
```

例如上面的打印指的是在OverlayDemo.apk里面用`0x7f030000`这个id去搜索资源的时候会映射到`0x7f010000`这个id去它的overlay apk(OverlayDemoOverlay.apk)里面搜索。

# 新增资源

安卓的RRO机制应该是不支持新增资源的,所以我在overlay apk里面加入了目标apk没有的`app_name2`字符串之后dump的时候打印`W failed to find resource 'string/app_name2'`。

但我们可以通过代码在指定的overlay包里面搜索资源id然后读取,实现新增资源的目的:

```java
id = getResources().getIdentifier("app_name2", "string", "me.linjw.demo.overlay.overlay");
if (id != 0) {
    sb.append("app_name2=" + getResources().getString(id));
}else {
    sb.append("app_name2=?");
}
```

# 完整Demo

完整的Demo已经上传到[Github](https://github.com/bluesky466/OverlayDemo)