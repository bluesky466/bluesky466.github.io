title: FileProvider的一些事
date: 2023-02-23 22:47:45
tags:
    - 技术相关
    - Android
---

高版本的android对文件权限的管控抓的很严格,理论上两个应用之间的文件传递现在都应该是用FileProvider去实现,这篇博客来一起了解下它的实现原理。

首先我们要明确一点,FileProvider就是一个ContentProvider,所以需要在AndroidManifest.xml里面对它进行声明:

```
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="me.linjw.demo.fileprovider.provider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_path" />
</provider>
```

和普通的ContentProvider不一样的是他多了一个android.support.FILE\_PROVIDER\_PATHS的meta-data指定了一个xml资源:

```
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <root-path name="root" path="" />
    <files-path name="files" path="images/" />
    <cache-path name="cache" path="" />
    <external-path name="external" path="" />
    <external-files-path  name="external-files" path="" />
    <external-cache-path name="external-cache" path="" />
    <external-media-path name="external-media" path="" />
</paths>
```

# 文件URI

这个xml的作用在于为文件生成URI,root-path、files-path、cache-path这些标签代表父路径:

- root-path : File("/")
- files-path : Context.getFilesDir()
- cache-path : context.getCacheDir()
- external-path : Environment.getExternalStorageDirectory()
- external-files-path : ContextCompat.getExternalFilesDirs(context, null)[0]
- external-cache-path :  ContextCompat.getExternalCacheDirs(context)[0]
- external-media-path :  context.getExternalMediaDirs()[0]

path属性代表子路径,name代表为"父路径/子路径"起的名字,

```
<files-path name="files" path="images/" />
```

例如上面配置代表的就是我们为 new File(context.getFilesDir(), "images/") 这个路径起了个名字叫做files

```
val filesDir = File(context.getFilesDir(), "images/")
val uri = FileProvider.getUriForFile(this, "me.linjw.demo.fileprovider.provider", File(filesDir, "test.jpg"))
// uri就是把filesDir的路径转换"files",然后加上content://me.linjw.demo.fileprovider.provider
// 即 "content://me.linjw.demo.fileprovider.provider/files/test.jpg"
```

从FileProvider的源码里面就能看到这部分的转换逻辑:

```
private static final String TAG_ROOT_PATH = "root-path";
private static final String TAG_FILES_PATH = "files-path";
private static final String TAG_CACHE_PATH = "cache-path";
private static final String TAG_EXTERNAL = "external-path";
private static final String TAG_EXTERNAL_FILES = "external-files-path";
private static final String TAG_EXTERNAL_CACHE = "external-cache-path";
private static final String TAG_EXTERNAL_MEDIA = "external-media-path";

...

int type;
while ((type = in.next()) != END_DOCUMENT) {
    if (type == START_TAG) {
        final String tag = in.getName();

        final String name = in.getAttributeValue(null, ATTR_NAME);
        String path = in.getAttributeValue(null, ATTR_PATH);

        File target = null;
        if (TAG_ROOT_PATH.equals(tag)) {
            target = DEVICE_ROOT;
        } else if (TAG_FILES_PATH.equals(tag)) {
            target = context.getFilesDir();
        } else if (TAG_CACHE_PATH.equals(tag)) {
            target = context.getCacheDir();
        } else if (TAG_EXTERNAL.equals(tag)) {
            target = Environment.getExternalStorageDirectory();
        } else if (TAG_EXTERNAL_FILES.equals(tag)) {
            File[] externalFilesDirs = ContextCompat.getExternalFilesDirs(context, null);
            if (externalFilesDirs.length > 0) {
                target = externalFilesDirs[0];
            }
        } else if (TAG_EXTERNAL_CACHE.equals(tag)) {
            File[] externalCacheDirs = ContextCompat.getExternalCacheDirs(context);
            if (externalCacheDirs.length > 0) {
                target = externalCacheDirs[0];
            }
        } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP
                && TAG_EXTERNAL_MEDIA.equals(tag)) {
            File[] externalMediaDirs = context.getExternalMediaDirs();
            if (externalMediaDirs.length > 0) {
                target = externalMediaDirs[0];
            }
        }

        if (target != null) {
            strat.addRoot(name, buildPath(target, path));
        }
    }
}

...

private static File buildPath(File base, String... segments) {
    File cur = base;
    for (String segment : segments) {
        if (segment != null) {
            cur = new File(cur, segment);
        }
    }
    return cur;
}
```

查询的时候就只需要从strat里面找到文件路径最匹配的name即可。

# 打开文件

有了这个uri之后我们就能通过Intent将它传给其他应用,并配置Intent.FLAG\_GRANT\_READ\_URI\_PERMISSION或者Intent.FLAG\_GRANT\_WRITE\_URI\_PERMISSION为其他应用设置读写权限:

```
val uri = FileProvider.getUriForFile(this, "me.linjw.demo.fileprovider.provider", file)
val intent = Intent()
intent.data = uri
intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
intent.setClassName("me.linjw.demo.fileprovider.recv", "me.linjw.demo.fileprovider.recv.MainActivity")
startActivity(intent)
```

其他应用拿到这个uri就可以通过ContentResolver.openInputStream打开文件流:

```
val inputStream = intent.data?.let { contentResolver.openInputStream(it) }
```

或者有时候我们希望通过String传递uri的时候可以提前使用Context.grantUriPermission为指定的包名申请权限,然后接收端Uri.parse去解析出Uri来操作文件:


```
// 发送端
val uri = FileProvider.getUriForFile(this, "me.linjw.demo.fileprovider.provider", file)
grantUriPermission("me.linjw.demo.fileprovider.recv", uri, Intent.FLAG_GRANT_READ_URI_PERMISSION)

val intent = Intent()
intent.putExtra("uri", uri.toString())
intent.setClassName("me.linjw.demo.fileprovider.recv", "me.linjw.demo.fileprovider.recv.MainActivity")
startActivity(intent)

// 接收端
val uri = Uri.parse(intent.getStringExtra("uri"))
val inputStream = contentResolver.openInputStream(uri)
```

Uri操作文件的原理实际上就是通过请求我们之前声明的me.linjw.demo.fileprovider.provider这个ContentProvider,让它给我们去打开文件:

```
// FileProvider.java
public ParcelFileDescriptor openFile(@NonNull Uri uri, @NonNull String mode)
        throws FileNotFoundException {
    // ContentProvider has already checked granted permissions
    final File file = mStrategy.getFileForUri(uri);
    final int fileMode = modeToMode(mode);
    return ParcelFileDescriptor.open(file, fileMode);
}
```

也就是说文件权限的校验实际上只发生在打开的阶段.其他应用虽然没有权限打开我们的文件,但是我们可以在ContentProvider里面帮它打开然后返回文件描述符,给其他应用去读写。

{% plantuml %}
APP1 -> APP2 : 文件uri
APP2 -> APP1 : ContentProvider.openFile
APP1 -> APP2 : ParcelFileDescriptor
APP2 -> File : 使用ParcelFileDescriptor打开APP1的文件进行读写
{% endplantuml %}

# 系统应用使用FileProvider的坑

项目中有个系统应用需要向其他应用传的文件,于是把FileProvider加上,然后发现其他应用还是没有权限。从日志里面看是说这个FileProvider并没有从UID 1000里暴露出来:

```
02-13 06:52:28.921  4292  4292 E AndroidRuntime: Caused by: java.lang.SecurityException: Permission Denial: opening provider androidx.core.content.FileProvider from ProcessRecord{806d30d 4292:me.linjw.demo.fileprovider.recv/u0a53} (pid=4292, uid=10053) that is not exported from UID 1000
```

由于这个UID 1000太显眼，所以尝试将系统签名去掉发现权限就正常了,实锤是系统签名的原因。

查看出现异常的时候的日志,发现了下面的打印: 

```
02-13 06:52:28.486   863  1393 W UriGrantsManagerService: For security reasons, the system cannot issue a Uri permission grant to content://me.linjw.demo.fileprovider.provider/root/data/user/0/me.linjw.demo.fileprovider/files/test.txt [user 0]; use startActivityAsCaller() instead
```

在代码里面搜索关键字,发现系统应用需要在源码里面配置FileProvider的authorities:

```
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r29:frameworks/base/services/core/java/com/android/server/uri/UriGrantsManagerService.java

// Bail early if system is trying to hand out permissions directly; it
// must always grant permissions on behalf of someone explicit.
final int callingAppId = UserHandle.getAppId(callingUid);
if ((callingAppId == SYSTEM_UID) || (callingAppId == ROOT_UID)) {
    if ("com.android.settings.files".equals(grantUri.uri.getAuthority())
            || "com.android.settings.module_licenses".equals(grantUri.uri.getAuthority())) {
        // Exempted authority for
        // 1. cropping user photos and sharing a generated license html
        //    file in Settings app
        // 2. sharing a generated license html file in TvSettings app
        // 3. Sharing module license files from Settings app
    } else {
        Slog.w(TAG, "For security reasons, the system cannot issue a Uri permission"
                + " grant to " + grantUri + "; use startActivityAsCaller() instead");
        return -1;
    }
}
```

# 直接传递ParcelFileDescriptor

从原理上看FileProvider实际就是打开文件的ParcelFileDescriptor传给其他应用使用,那我们能不能直接打开文件然后将ParcelFileDescriptor直接通过Intent传给其他应用呢?

```
val intent = Intent()
intent.putExtra("fd" , ParcelFileDescriptor.open(file, MODE_READ_ONLY))
intent.setClassName("me.linjw.demo.fileprovider.recv", "me.linjw.demo.fileprovider.recv.MainActivity")
startActivity(intent)
```

答案是不行:

```
02-15 20:27:24.200 16968 16968 E AndroidRuntime: Process: me.linjw.demo.fileprovider, PID: 16968
02-15 20:27:24.200 16968 16968 E AndroidRuntime: java.lang.RuntimeException: Not allowed to write file descriptors here                        
02-15 20:27:24.200 16968 16968 E AndroidRuntime:        at android.os.Parcel.nativeWriteFileDescriptor(Native Method)
02-15 20:27:24.200 16968 16968 E AndroidRuntime:        at android.os.Parcel.writeFileDescriptor(Parcel.java:922)
02-15 20:27:24.200 16968 16968 E AndroidRuntime:        at android.os.ParcelFileDescriptor.writeToParcel(ParcelFileDescriptor.java:1110)
02-15 20:27:24.200 16968 16968 E AndroidRuntime:        at android.os.Parcel.writeParcelable(Parcel.java:1953)
02-15 20:27:24.200 16968 16968 E AndroidRuntime:        at android.os.Parcel.writeValue(Parcel.java:1859)
02-15 20:27:24.200 16968 16968 E AndroidRuntime:        at android.os.Parcel.writeArrayMapInternal(Parcel.java:1024)
02-15 20:27:24.200 16968 16968 E AndroidRuntime:        at android.os.BaseBundle.writeToParcelInner(BaseBundle.java:1620)
02-15 20:27:24.200 16968 16968 E AndroidRuntime:        at android.os.Bundle.writeToParcel(Bundle.java:1304)
02-15 20:27:24.200 16968 16968 E AndroidRuntime:        at android.os.Parcel.writeBundle(Parcel.java:1093)
02-15 20:27:24.200 16968 16968 E AndroidRuntime:        at android.content.Intent.writeToParcel(Intent.java:11123)
02-15 20:27:24.200 16968 16968 E AndroidRuntime:        at android.app.IActivityTaskManager$Stub$Proxy.startActivity(IActivityTaskManager.java:
2298)
```

原因在于Instrumentation的execStartActivity启动Activity前会调用Intent.prepareToLeaveProcess最终调用到Bundle.setAllowFds(false)不允许传递ParcelFileDescriptor:

```
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r29:frameworks/base/core/java/android/app/Instrumentation.java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    ...
    intent.prepareToLeaveProcess(who);
    ...
}


// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r29:frameworks/base/core/java/android/content/Intent.java
public void prepareToLeaveProcess(Context context) {
    final boolean leavingPackage;
    if (mComponent != null) {
        leavingPackage = !Objects.equals(mComponent.getPackageName(), context.getPackageName());
    } else if (mPackage != null) {
        leavingPackage = !Objects.equals(mPackage, context.getPackageName());
    } else {
        leavingPackage = true;
    }
    prepareToLeaveProcess(leavingPackage);
}

/**
 * Prepare this {@link Intent} to leave an app process.
 *
 * @hide
 */
public void prepareToLeaveProcess(boolean leavingPackage) {
    setAllowFds(false);
    ...
}

public void setAllowFds(boolean allowFds) {
    if (mExtras != null) {
        mExtras.setAllowFds(allowFds);
    }
}
```

一开始我想通过反射去强行调用setAllowFds(true),但是发现这个方法被限制了,需要系统权限才能调用:

```
Accessing hidden method Landroid/os/Bundle;->setAllowFds(Z)Z (max-target-o, reflection, denied)
```

只能另谋出路,由于ParcelFileDescriptor实现了Parcelable,所以我们可以通过传递Binder的方式迂回的去传递:

```
// aidl
interface IFileDescriptorsProvider {
    ParcelFileDescriptor get();
}

// 发送端
val fileProvider = object : IFileDescriptorsProvider.Stub() {
    override fun get(): ParcelFileDescriptor {
        return ParcelFileDescriptor.open(file, MODE_READ_ONLY)
    }
}
val intent = Intent()
val bundle = Bundle().apply { putBinder("fileProvider", fileProvider) }
intent.putExtras(bundle)
intent.setClassName("me.linjw.demo.fileprovider.recv", "me.linjw.demo.fileprovider.recv.MainActivity")
startActivity(intent)

// 接收端
val text = intent.extras?.getBinder("fileProvider")?.let { it ->
    val fd = IFileDescriptorsProvider.Stub.asInterface(it).get()
    AssetFileDescriptor(fd, 0, -1)
        .createInputStream()
        .use { it.bufferedReader().readLine() }
}
```
