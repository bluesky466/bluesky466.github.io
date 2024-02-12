title: Android PackageCache 机制
date: 2024-02-07 11:37:16
tags:
    - 技术相关
    - Android
---

今天突然接到bug说系统ota之后**必现无法使用并且重启无法恢复**,从日志上看个上个月往项目里面导入了[热更新](https://blog.islinjw.cn/2023/12/19/Android%E7%83%AD%E6%9B%B4%E6%96%B0%E5%AE%9E%E8%B7%B5/)的机制用于方便调试相关,惊出一身冷汗:

```
E AndroidRuntime: Process: com.xx.xx.xx, PID: 2012
E AndroidRuntime: java.lang.RuntimeException: Unable to instantiate application com.xx.xx.xx.XXApplication package
com.xx.xx.xx: java.lang.ClassCastException: com.xx.xx.xx.XxApplication cannot be cast to android.app.Application
```

我们在新版本里将Application改成了HotfixApplication,然后原本的com.xx.xx.xx.XxApplication父类改成了自定义的ApplicationLike和android.app.Application没有关系。所以如果启动进程的时候用com.xx.xx.xx.XxApplication去启动的确是会出现转换问题的。

但是问题在于我们已经修改了AndroidManifest.xml,这样意味着系统ota之后系统有些缓存没有清理导致读取到的还是旧的信息。这个问题虽然应用端可以规避,但是整个系统的ota机制应该还是哪个地方出现了问题,其他第三方的应用也会遇到同样的问题,需要深入定位下根因。


# package cache

为了加快开机速度,安卓在解析完一次应用信息之后会在/data/system/package_cache/{FINGERPRINT}下保存,每个应用保存成一个文件里面包括了应用的权限、Application的name等信息。除非应用有变更才会去刷新应用的缓存信息({FINGERPRINT}是根据系统信息计算的md5,用于对比确认是不是同一个版本的rom),这样可以不用每次开机都去解压apk解析应用信息:

```
console:/data/system/package_cache/d529b6afb8a5a0c7a5b626efbac421ba14e3ea55 #
ls
AndroidRemoteRs232-16             NetworkPermissionConfig-16
AutoTestServer-16                 NetworkStack-16
BasicDreams-16                    OsuLogin-16
Bluetooth-16                      PacProcessor-16
BluetoothMidiService-16           PackageInstaller-16
...
```

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/pm/parsing/PackageParser2.java;l=157
public ParsedPackage parsePackage(File packageFile, int flags, boolean useCaches,
        List<File> frameworkSplits) throws PackageManagerException {
    if (useCaches && mCacher != null) {
        ParsedPackage parsed = mCacher.getCachedResult(packageFile, flags);
        if (parsed != null) {
            return parsed;
        }
    }
    ...
    ParseResult<ParsingPackage> result = parsingUtils.parsePackage(input, packageFile, flags,
            frameworkSplits);
    ...
    ParsedPackage parsed = (ParsedPackage) result.getResult().hideAsParsed();
    ...
    if (mCacher != null) {
        mCacher.cacheResult(packageFile, flags, parsed);
    }
    ...
    return parsed;
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/pm/parsing/PackageCacher.java;l=188
public void cacheResult(File packageFile, int flags, ParsedPackage parsed) {
    try {
        final String cacheKey = getCacheKey(packageFile, flags);
        final File cacheFile = new File(mCacheDir, cacheKey);

        if (cacheFile.exists()) {
            if (!cacheFile.delete()) {
                Slog.e(TAG, "Unable to delete cache file: " + cacheFile);
            }
        }

        final byte[] cacheEntry = toCacheEntry(parsed);

        if (cacheEntry == null) {
            return;
        }

        try (FileOutputStream fos = new FileOutputStream(cacheFile)) {
            fos.write(cacheEntry);
        } catch (IOException ioe) {
            Slog.w(TAG, "Error writing cache entry.", ioe);
            cacheFile.delete();
        }
    } catch (Throwable e) {
        Slog.w(TAG, "Error saving package cache.", e);
    }
}
```

上面使用的mCacher这个缓存目录是在PackageManagerService启动的时候调用PackageManagerServiceUtils.preparePackageParserCache去创建的:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java;l=1986
public PackageManagerService(PackageManagerServiceInjector injector, boolean onlyCore,
            boolean factoryTest, final String buildFingerprint, final boolean isEngBuild,
            final boolean isUserDebugBuild, final int sdkVersion, final String incrementalVersion) {
	...
	mCacheDir = PackageManagerServiceUtils.preparePackageParserCache(
	                    mIsEngBuild, mIsUserDebugBuild, mIncrementalVersion);
	...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/pm/PackageManagerServiceUtils.java;l=1309
public static @Nullable File preparePackageParserCache(boolean forEngBuild,
        boolean isUserDebugBuild, String incrementalVersion) {
    ...
	// The base directory for the package parser cache lives under /data/system/.
    final File cacheBaseDir = Environment.getPackageCacheDirectory();
    if (!FileUtils.createDir(cacheBaseDir)) {
        return null;
    }

    // There are several items that need to be combined together to safely
    // identify cached items. In particular, changing the value of certain
    // feature flags should cause us to invalidate any caches.
    final String cacheName = FORCE_PACKAGE_PARSED_CACHE_ENABLED ? "debug"
            : PackagePartitions.FINGERPRINT;

    // Reconcile cache directories, keeping only what we'd actually use.
    for (File cacheDir : FileUtils.listFilesOrEmpty(cacheBaseDir)) {
        if (Objects.equals(cacheName, cacheDir.getName())) {
            Slog.d(TAG, "Keeping known cache " + cacheDir.getName());
        } else {
            Slog.d(TAG, "Destroying unknown cache " + cacheDir.getName());
            FileUtils.deleteContentsAndDir(cacheDir);
        }
    }

    // Return the versioned package cache directory.
    File cacheDir = FileUtils.createDir(cacheBaseDir, cacheName);
    ...
    return cacheDir;
}
```

# 系统FINGERPRINT

从preparePackageParserCache的代码可以看出来其实是在Environment.getPackageCacheDirectory()下的PackagePartitions.FINGERPRINT子目录。

从Environment代码可以看出来Environment.getPackageCacheDirectory()返回的实际就是`/data/system/package_cache/`:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/core/java/android/os/Environment.java

private static final String DIR_ANDROID_DATA_PATH = getDirectoryPath(ENV_ANDROID_DATA, "/data");
private static final File DIR_ANDROID_DATA = new File(DIR_ANDROID_DATA_PATH);

public static File getPackageCacheDirectory() {
    return new File(getDataSystemDirectory(), "package_cache");
}

public static File getDataSystemDirectory() {
    return new File(getDataDirectory(), "system");
}

public static File getDataDirectory() {
    return DIR_ANDROID_DATA;
}
```

而PackagePartitions.FINGERPRINT则是通过是一堆ro.xxxxx..build.fingerprint的属性计算出来的:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/core/java/android/content/pm/PackagePartitions.java
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

public static final String FINGERPRINT = getFingerprint();

private static String getFingerprint() {
    final String[] digestProperties = new String[SYSTEM_PARTITIONS.size() + 1];
    for (int i = 0; i < SYSTEM_PARTITIONS.size(); i++) {
        final String partitionName = SYSTEM_PARTITIONS.get(i).getName();
        digestProperties[i] = "ro." + partitionName + ".build.fingerprint";
    }
    digestProperties[SYSTEM_PARTITIONS.size()] = "ro.build.fingerprint"; // build fingerprint
    return SystemProperties.digestOf(digestProperties);
}
```

从这里可以大概猜测到PackagePartitions.FINGERPRINT在ota前后没有变化导致使用的还是旧的缓存目录,读取的应用信息里还是旧的Application name。

幸亏是必现的问题,我们刷回旧的rom看看缓存目录,然后再进行OTA对比新的缓存目录发现的确没有改变。

因为之前测试是说重启不能恢复的,这个时候只要手动删除这个缓存目录然后重启发现就能恢复正常了,确认就是这个缓存的问题。

再看这堆参与计算的属性里其中有个属性ro.build.version.incremental按道理ota之后需要改变,改变之后PackagePartitions.FINGERPRINT就会改变,从而使用新的缓存目录并且删除旧的缓存目录,但是从OTA前后读取出来看它并没有改变过。

好吧,那就是系统的锅了,找了系统组的大佬确认这个是有特殊的需求临时的调试软件,的确就是需要固定FINGERPRINT。正式生产的rom里面FINGERPRINT是会变的,虚惊一场......


# apk变更检查

由于我们这个应用配置了`android:persistent="true"`,不能`install -r`之前我们调试都是`remount`之后推到机器里面的,为什么之前调试的时候没有遇到呢?

我尝试了下修改信息之后`adb push`替换预装路径`/system_ext/app/XXX/XXX.apk`重启之后缓存的确没有修改。从日志上看实际系统已经发现它改变了,但是看起来是重新安装的时候忽略掉了所以没有更新缓存:

```
02-06 21:52:24.909   836   836 I PackageManager: /system_ext/app/XXX changed; collecting certs
02-06 21:52:24.981   836   836 W PackageManager: Failed to scan /system_ext/app/XXX: Application package com.xx.xx.xx already installed.  Skipping duplicate.
```

而我之前的调试手法都是先`rm -r /system_ext/app/XXX/`删掉预装目录,然后直接将编译的apk`adb push`到`/system_ext/app/`下,这种情况下替换`/system_ext/app/XXX.apk`可以发现缓存是会更新的,日志上看的确发现应用改变之后没有安装失败的提示:

```
02-06 21:48:59.906   839   839 I PackageManager: /system_ext/app/XXX.apk changed; collecting certs    
```

从代码上看应该是在扫描预装路径的时候就put到了mPm.mPackages导致后面不能重复安装,而`/system_ext/app/XXX.apk`非预装的路径则没有这个问题:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/pm/InstallPackageHelper.java;l=4176
// A package name must be unique; don't allow duplicates
if ((scanFlags & SCAN_NEW_INSTALL) == 0
        && mPm.mPackages.containsKey(pkg.getPackageName())) {
    throw new PackageManagerException(INSTALL_FAILED_DUPLICATE_PACKAGE,
            "Application package " + pkg.getPackageName()
                    + " already installed.  Skipping duplicate.");
}
```

我升级到正式生产的rom去验证,发现正式生产的rom里面直接替换`/system_ext/app/XXX/XXX.apk`也是能更新缓存的,意味着这个临时软件有什么奇怪的配置导致了这个现象,从系统哥那了解到这个奇葩需求的详情来看这里应该也是需求之一。由于具体的代码和配置太多不好找就不去探究哪个配置引起的了,但是能确认的是当apk被直接替换之后系统可以通过修改时间确认apk已经变更然后刷新缓存的:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/pm/ScanPackageUtils.java;l=934
public static void collectCertificatesLI(PackageSetting ps, ParsedPackage parsedPackage,
        Settings.VersionInfo settingsVersionForPackage, boolean forceCollect,
        boolean skipVerify, boolean isPreNMR1Upgrade)
        throws PackageManagerException {
    // When upgrading from pre-N MR1, verify the package time stamp using the package
    // directory and not the APK file.
    final long lastModifiedTime = isPreNMR1Upgrade
            ? new File(parsedPackage.getPath()).lastModified()
            : getLastModifiedTime(parsedPackage);
    if (ps != null && !forceCollect
            && ps.getPathString().equals(parsedPackage.getPath())
            && ps.getLastModifiedTime() == lastModifiedTime
            && !ReconcilePackageUtils.isCompatSignatureUpdateNeeded(settingsVersionForPackage)
            && !ReconcilePackageUtils.isRecoverSignatureUpdateNeeded(
            settingsVersionForPackage)) {
        。。。
    } else {
        Slog.i(TAG, parsedPackage.getPath() + " changed; collecting certs"
                + (forceCollect ? " (forced)" : ""));
    }
    ...
}
```
