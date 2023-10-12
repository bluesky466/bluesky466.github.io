title: bugreport原理
date: 2023-10-12 08:47:26
tags:
    - 技术相关
    - Android
---

[bugreport](https://developer.android.com/studio/debug/bug-report?hl=zh-cn)在定位分析问题的时候十分有用,一个`adb bugreport`就能打包拉取logcat、tombstones、anr堆栈等各种信息:

{% img /bugreport原理/1.png %}

但是这里面都是一些标准的安卓调试信息,对于我司这种定制系统的情况,有些特殊的信息也想在bugreport里面导出要怎么办?最近就遇到了这样一个需求,于是去研究了下整个bugreport的流程。


# adb bugreport命令原理

首先我们的adb命令都是在adb clinet里面解析处理的

```c++
//https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:packages/modules/adb/client/main.cpp
int main(int argc, char* argv[], char* envp[]) {
    __adb_argv = const_cast<const char**>(argv);
    __adb_envp = const_cast<const char**>(envp);
    adb_trace_init(argv);
    return adb_commandline(argc - 1, const_cast<const char**>(argv + 1));
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:packages/modules/adb/client/commandline.cpp
int adb_commandline(int argc, const char** argv) {
	...
	else if (!strcmp(argv[0], "bugreport")) {
        Bugreport bugreport;
        return bugreport.DoIt(argc, argv);
    }
    ...
}
```

可以看到bugreport命令由Bugreport::DoIt去处理:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:packages/modules/adb/client/bugreport.cpp
int Bugreport::DoIt(int argc, const char** argv) {
	...
	std::string bugz_command = "bugreportz -p";
	...
	BugreportStandardStreamsCallback bugz_callback(dest_dir, dest_file, show_progress, this);
    return SendShellCommand(bugz_command, false, &bugz_callback);
}
```

可以看到最终的实现命令是"bugreportz -p",它是在物理设备上生成bugreport并打印进度的命令,在`adb shell`里面执行它可以看到它生成bugreport的过程日志:

```shell
odin:/ $ bugreportz -p
PROGRESS:20/1683
BEGIN:/data/user_de/0/com.android.shell/files/bugreports/bugreport-odin-TKQ1.220829.002-2023-10-11-19-07-35.zip
PROGRESS:40/1683
PROGRESS:60/1683
PROGRESS:110/1683
...
PROGRESS:1560/1683
PROGRESS:1619/1683
OK:/data/user_de/0/com.android.shell/files/bugreports/bugreport-odin-TKQ1.220829.002-2023-10-11-19-07-35.zip
PROGRESS:1639/1683
```

`adb bugreport`实际也是执行这个命令之后不断解析它的日志来计算进度进行打印:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:packages/modules/adb/client/bugreport.cpp

class BugreportStandardStreamsCallback : public StandardStreamsCallbackInterface {
	...
	// 解析bugreportz -p的打印
	bool OnStdout(const char* buffer, size_t length) {
	    for (size_t i = 0; i < length; i++) {
	        char c = buffer[i];
	        if (c == '\n') {
	        	// 逐行解析,在里面打印进度, 例如
	        	// [ 20%] generating bugreport-odin-TKQ1.220829.002-2023-10-11-19-52-56.zip
	            ProcessLine(line_);
	            line_.clear();
	        } else {
	            line_.append(1, c);
	        }
	    }
	    return true;
	}

	// bugreportz -p执行完成的回调
	int Done(int unused_) {
        ...
        // 执行完成之后使用adb pull将bugreportz -p生成的zip文件pull到本机
        if (status_ == 0) {
            ...
            SetLineMessage("pulling");
            status_ =
                br_->DoSyncPull(srcs, destination.c_str(), false, line_message_.c_str()) ? 0 : 1;
            ...
        }
        return status_;
    }
```

总结一下就是`adb bugreport`实际就是让安卓系统执行`bugreportz -p`生成bugreport的zip包,然后再使用`adb pull`命令拉到本机。

# bugreportz原理

我们继续追踪bugreportz的源码:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/cmds/bugreportz/main.cpp
int main(int argc, char* argv[]) {
    ...
    property_set("ctl.start", "dumpstate");
    ...
    s = socket_local_client("dumpstate", ANDROID_SOCKET_NAMESPACE_RESERVED, SOCK_STREAM);
    ...
    ret = bugreportz(s, show_progress);
    ...
    if (close(s) == -1) {
        fprintf(stderr, "WARNING: error closing socket: %s\n", strerror(errno));
        ret = EXIT_FAILURE;
    }
    return ret;
}
```

从代码上看它也只是个命令的转发者,通过ctl.start属性启动dumpstate服务(/system/bin/dumpstate):

```rc
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/cmds/dumpstate/dumpstate.rc

service dumpstate /system/bin/dumpstate -s
    class main
    socket dumpstate stream 0660 shell log
    disabled
    oneshot
```

dumpstate创建了个socket的服务端,所以bugreportz就是通过socket和它进行通讯的,不断读取dumpstate的输出并打印。

# dumpstate原理

所以活又外包到了dumpstate:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/cmds/dumpstate/main.cpp
int main(int argc, char* argv[]) {
    ...
    return run_main(argc, argv);
    ...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/cmds/dumpstate/dumpstate.cpp
int run_main(int argc, char* argv[]) {
    Dumpstate::RunStatus status = ds.ParseCommandlineAndRun(argc, argv);
    ...
}

Dumpstate::RunStatus Dumpstate::ParseCommandlineAndRun(int argc, char* argv[]) {
	...
	status = Run(-1 /* calling_uid */, "" /* calling_package */);
	...
}

Dumpstate::RunStatus Dumpstate::Run(int32_t calling_uid, const std::string& calling_package) {
    Dumpstate::RunStatus status = RunInternal(calling_uid, calling_package);
    ...
}

Dumpstate::RunStatus Dumpstate::RunInternal(int32_t calling_uid,
                                            const std::string& calling_package) {
	...
	// Redirect stdout to tmp_path_. This is the main bugreport entry and will be
    // moved into zip file later, if zipping.
    TEMP_FAILURE_RETRY(dup_stdout_fd = dup(fileno(stdout)));
    // TODO: why not write to a file instead of stdout to overcome this problem?
    /* TODO: rather than generating a text file now and zipping it later,
        it would be more efficient to redirect stdout to the zip entry
        directly, but the libziparchive doesn't support that option yet. */
    if (!redirect_to_file(stdout, const_cast<char*>(tmp_path_.c_str()))) {
        return ERROR;
    }
    ...
	// Dump state for the default case. This also drops root.
    RunStatus s = DumpstateDefaultAfterCritical();
	...
	// Zip the (now complete) .tmp file within the internal directory.
    FinalizeFile();
    ...
}
```

RunInternal里面先将stdout重定向到了tmp文件,后面的打印就会去到tmp文件。然后就调用最终来的实际干活的DumpstateDefaultAfterCritical:

```c++
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r8:frameworks/native/cmds/dumpstate/dumpstate.cpp

#define PSTORE_LAST_KMSG "/sys/fs/pstore/console-ramoops"
#define ALT_PSTORE_LAST_KMSG "/sys/fs/pstore/console-ramoops-0"
#define BLK_DEV_SYS_DIR "/sys/block"

#define RECOVERY_DIR "/cache/recovery"
#define RECOVERY_DATA_DIR "/data/misc/recovery"
#define UPDATE_ENGINE_LOG_DIR "/data/misc/update_engine_log"
#define LOGPERSIST_DATA_DIR "/data/misc/logd"
#define PREREBOOT_DATA_DIR "/data/misc/prereboot"
#define PROFILE_DATA_DIR_CUR "/data/misc/profiles/cur"
#define PROFILE_DATA_DIR_REF "/data/misc/profiles/ref"
#define XFRM_STAT_PROC_FILE "/proc/net/xfrm_stat"
#define WLUTIL "/vendor/xbin/wlutil"
#define WMTRACE_DATA_DIR "/data/misc/wmtrace"
#define OTA_METADATA_DIR "/metadata/ota"
#define SNAPSHOTCTL_LOG_DIR "/data/misc/snapshotctl_log"
#define LINKERCONFIG_DIR "/linkerconfig"
#define PACKAGE_DEX_USE_LIST "/data/system/package-dex-usage.list"
#define SYSTEM_TRACE_SNAPSHOT "/data/misc/perfetto-traces/bugreport/systrace.pftrace"
#define CGROUPFS_DIR "/sys/fs/cgroup"

// TODO(narayan): Since this information has to be kept in sync
// with tombstoned, we should just put it in a common header.
//
// File: system/core/debuggerd/tombstoned/tombstoned.cpp
static const std::string TOMBSTONE_DIR = "/data/tombstones/";
static const std::string TOMBSTONE_FILE_PREFIX = "tombstone_";
static const std::string ANR_DIR = "/data/anr/";
static const std::string ANR_FILE_PREFIX = "anr_";

Dumpstate::RunStatus Dumpstate::DumpstateDefaultAfterCritical() {
    // Capture first logcat early on; useful to take a snapshot before dumpstate logs take over the
    // buffer.
    DoLogcat();
    // Capture timestamp after first logcat to use in next logcat
    time_t logcat_ts = time(nullptr);

    /* collect stack traces from Dalvik and native processes (needs root) */
    std::future<std::string> dump_traces;
    if (dump_pool_) {
        RETURN_IF_USER_DENIED_CONSENT();
        // One thread is enough since we only need to enqueue DumpTraces here.
        dump_pool_->start(/* thread_counts = */1);

        // DumpTraces takes long time, post it to the another thread in the
        // pool, if pool is available
        dump_traces = dump_pool_->enqueueTask(
            DUMP_TRACES_TASK, &Dumpstate::DumpTraces, &ds, &dump_traces_path);
    } else {
        RUN_SLOW_FUNCTION_WITH_CONSENT_CHECK_AND_LOG(DUMP_TRACES_TASK, ds.DumpTraces,
                &dump_traces_path);
    }

    /* Run some operations that require root. */
    if (!PropertiesHelper::IsDryRun()) {
        ds.tombstone_data_ = GetDumpFds(TOMBSTONE_DIR, TOMBSTONE_FILE_PREFIX);
        ds.anr_data_ = GetDumpFds(ANR_DIR, ANR_FILE_PREFIX);
    }

    ds.AddDir(RECOVERY_DIR, true);
    ds.AddDir(RECOVERY_DATA_DIR, true);
    ds.AddDir(UPDATE_ENGINE_LOG_DIR, true);
    ds.AddDir(LOGPERSIST_DATA_DIR, false);
    if (!PropertiesHelper::IsUserBuild()) {
        ds.AddDir(PROFILE_DATA_DIR_CUR, true);
        ds.AddDir(PROFILE_DATA_DIR_REF, true);
        ds.AddZipEntry(ZIP_ROOT_DIR + PACKAGE_DEX_USE_LIST, PACKAGE_DEX_USE_LIST);
    }
    ds.AddDir(PREREBOOT_DATA_DIR, false);
    add_mountinfo();
    DumpIpTablesAsRoot();
    DumpDynamicPartitionInfo();
    ds.AddDir(OTA_METADATA_DIR, true);

    // Capture any IPSec policies in play. No keys are exposed here.
    RunCommand("IP XFRM POLICY", {"ip", "xfrm", "policy"}, CommandOptions::WithTimeout(10).Build());

    // Dump IPsec stats. No keys are exposed here.
    DumpFile("XFRM STATS", XFRM_STAT_PROC_FILE);

    // Run ss as root so we can see socket marks.
    RunCommand("DETAILED SOCKET STATE", {"ss", "-eionptu"}, CommandOptions::WithTimeout(10).Build());

    // Run iotop as root to show top 100 IO threads
    RunCommand("IOTOP", {"iotop", "-n", "1", "-m", "100"});

    // Gather shared memory buffer info if the product implements it
    RunCommand("Dmabuf dump", {"dmabuf_dump"});
    RunCommand("Dmabuf per-buffer/per-exporter/per-device stats", {"dmabuf_dump", "-b"});

    DumpFile("PSI cpu", "/proc/pressure/cpu");
    DumpFile("PSI memory", "/proc/pressure/memory");
    DumpFile("PSI io", "/proc/pressure/io");

    if (dump_pool_) {
        RETURN_IF_USER_DENIED_CONSENT();
        WaitForTask(std::move(dump_traces));

        // Current running thread in the pool is the root user also. Delete
        // the pool and make a new one later to ensure none of threads in the pool are root.
        dump_pool_ = std::make_unique<DumpPool>(bugreport_internal_dir_);
    }
    if (!DropRootUser()) {
        return Dumpstate::RunStatus::ERROR;
    }

    RETURN_IF_USER_DENIED_CONSENT();
    Dumpstate::RunStatus status = dumpstate();
    // Capture logcat since the last time we did it.
    DoSystemLogcat(logcat_ts);
    return status;
}
```

从代码可以看出它的工作原理是

1. 通过AddDir、AddZipEntry往zip包里面插入目录和文件,
2. 通过RunCommand执行命令 : logcat、getprop、dumpsys等命令的结果获取用了取巧的方法,将标准输出重定向到tmp文件,所以实际是执行这些命令将结果收集到tmp文件,最终再将tmp文件加入zip包完成信息的收集。


# 添加自定义文件

分析完整个bugreport的生成流程,可以发现将我们自定义的信息导入到bugreport还是比较容易的,只需要把通过ds.AddZipEntry或者ds.AddDir把文件或者目录添加进zip包即可:

```c++
#define CUSTOMER_LOG "/mnt/tmp/customer.log"

ds.AddZipEntry(ZIP_ROOT_DIR + CUSTOMER_LOG, CUSTOMER_LOG);
```

当然如果你的信息是执行命令后的打印,也可以用RunCommand把执行结果写入文件