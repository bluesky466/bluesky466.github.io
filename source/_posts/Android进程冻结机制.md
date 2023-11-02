title: Android进程冻结机制
date: 2023-11-01 19:00:17
tags:
  - 技术相关
  - Android
---

# 奇怪的ANR

今天遇到了个很有意思的anr问题, 应用出现了anr:

```
7696:08-29 14:12:59.564898  7904  8341 I WindowManager: ANR in Window{3b0709 u0 me.linjw.demo.anr}. Reason:3b0709 me.linjw.demo.anr (server) is not responding. Waited 5001ms for FocusEvent(hasFocus=false)
8367:08-29 14:13:11.713363  7904 27946 E ActivityManager: ANR in me.linjw.demo.anr
```

但是trace文件里面没有任何堆栈:
```
Subject: Input dispatching timed out (3b0709 me.linjw.demo.anr (server) is not responding. Waited 5001ms for FocusEvent(hasFocus=false))

--- CriticalEventLog ---
capacity: 20
timestamp_ms: 1693311179660
window_ms: 300000

libdebuggerd_client: failed to read status response from tombstoned: timeout reached?

----- Waiting Channels: pid 26859 at 2023-08-29 14:12:59.664895544+0200 -----
Cmd line: me.linjw.demo.anr

sysTid=26859     do_freezer_trap
sysTid=26864     do_freezer_trap
sysTid=26865     do_freezer_trap
sysTid=26866     do_freezer_trap
sysTid=26867     do_freezer_trap
sysTid=26868     do_freezer_trap
sysTid=26869     do_freezer_trap
sysTid=26870     do_freezer_trap
sysTid=26871     do_freezer_trap
sysTid=26872     do_freezer_trap
sysTid=26873     do_freezer_trap
sysTid=26874     do_freezer_trap
sysTid=26875     do_freezer_trap
sysTid=26877     do_freezer_trap
sysTid=26879     do_freezer_trap
sysTid=26880     do_freezer_trap
sysTid=26882     do_freezer_trap
sysTid=26883     do_freezer_trap
sysTid=26887     do_freezer_trap
sysTid=26912     do_freezer_trap
sysTid=26918     do_freezer_trap
sysTid=26919     do_freezer_trap
sysTid=26922     do_freezer_trap
sysTid=26923     do_freezer_trap
sysTid=26938     do_freezer_trap
sysTid=27772     do_freezer_trap
sysTid=27815     do_freezer_trap
sysTid=27826     do_freezer_trap
sysTid=27827     do_freezer_trap

----- end 26859 -----

libdebuggerd_client: failed to read status response from tombstoned: Try again

----- Waiting Channels: pid 26859 at 2023-08-29 14:13:09.677383215+0200 -----
Cmd line: me.linjw.demo.anr

sysTid=26859     do_freezer_trap
sysTid=26864     do_freezer_trap
sysTid=26865     do_freezer_trap
sysTid=26866     do_freezer_trap
sysTid=26867     do_freezer_trap
sysTid=26868     do_freezer_trap
sysTid=26869     do_freezer_trap
sysTid=26870     do_freezer_trap
sysTid=26871     do_freezer_trap
sysTid=26872     do_freezer_trap
sysTid=26873     do_freezer_trap
sysTid=26874     do_freezer_trap
sysTid=26875     do_freezer_trap
sysTid=26877     do_freezer_trap
sysTid=26879     do_freezer_trap
sysTid=26880     do_freezer_trap
sysTid=26882     do_freezer_trap
sysTid=26883     do_freezer_trap
sysTid=26887     do_freezer_trap
sysTid=26912     do_freezer_trap
sysTid=26918     do_freezer_trap
sysTid=26919     do_freezer_trap
sysTid=26922     do_freezer_trap
sysTid=26923     do_freezer_trap
sysTid=26938     do_freezer_trap
sysTid=27772     do_freezer_trap
sysTid=27815     do_freezer_trap
sysTid=27826     do_freezer_trap
sysTid=27827     do_freezer_trap

----- end 26859 -----
```

从日志上过滤进程pid可以看到正在正常的执行任务,还没有执行完就被am\_freeze冻结了进程:

```
08-29 14:11:45.807967 26859 27815 V MessageEncoder: ... // 正常执行任务的打印
08-29 14:11:45.809835 26859 26859 D FloatView: ... // 正常执行任务的打印,任务没有执行完,后面应该还有打印但实际没有
08-29 14:11:45.884625  7904  8331 D ActivityManager: freezing 26859 me.linjw.demo.anr
08-29 14:11:45.885503  7904  8331 I am_freeze: [26859,me.linjw.demo.anr]
08-29 14:12:59.660658  7904 27946 I am_anr  : [0,26859,me.linjw.demo.anr,545832517,Input dispatching timed out (3b0709 me.linjw.demo.anr (server) is not responding. Waited 5001ms for FocusEvent(hasFocus=false))]
```

由于进程被冻结了,所以处理不了Input消息所以anr,由于进程被冻结了,所以anr的时候让进程去dump堆栈的请求也不会被处理。


# Freeze

很多的进程在退出前台之后会长期在后台占用内存、cpu,影响用户体验。在内存不足的时候会触发lmk清除内存,但是如果内存充足的情况下为了加速应用的切换速度,是不会杀死后台进程的。为了解决应用在后台默默消化cpu资源的问题,高版本的安卓实现了一套[冻结进程机制](https://source.android.com/docs/core/perf/cached-apps-freezer?hl=zh-cn),在Android 11以后支持。

我们可以在开发者选项里面找到"Suspend execution for cached apps"条目去控制后台进程冻结功能的开关,也可以用命令去控制:

> adb shell settings put global cached_apps_freezer <enabled|disabled|default>

- enable 打开
- disabled 关闭
- default 由系统决定是否打开


进程的OOM\_ADJ (Out of Memory Adjustment)值除了决定系统内存不足的时候是否回收该进程,进程冻结策略也是依赖它去计算的。有下面的这些场景会触发进程oom adj值的重新计算,大概有切换Activity、启动广播、绑定服务、是否可见状态改变等:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java
public class OomAdjuster {
    static final String TAG = "OomAdjuster";
    static final String OOM_ADJ_REASON_METHOD = "updateOomAdj";
    static final String OOM_ADJ_REASON_NONE = OOM_ADJ_REASON_METHOD + "_meh";
    static final String OOM_ADJ_REASON_ACTIVITY = OOM_ADJ_REASON_METHOD + "_activityChange";
    static final String OOM_ADJ_REASON_FINISH_RECEIVER = OOM_ADJ_REASON_METHOD + "_finishReceiver";
    static final String OOM_ADJ_REASON_START_RECEIVER = OOM_ADJ_REASON_METHOD + "_startReceiver";
    static final String OOM_ADJ_REASON_BIND_SERVICE = OOM_ADJ_REASON_METHOD + "_bindService";
    static final String OOM_ADJ_REASON_UNBIND_SERVICE = OOM_ADJ_REASON_METHOD + "_unbindService";
    static final String OOM_ADJ_REASON_START_SERVICE = OOM_ADJ_REASON_METHOD + "_startService";
    static final String OOM_ADJ_REASON_GET_PROVIDER = OOM_ADJ_REASON_METHOD + "_getProvider";
    static final String OOM_ADJ_REASON_REMOVE_PROVIDER = OOM_ADJ_REASON_METHOD + "_removeProvider";
    static final String OOM_ADJ_REASON_UI_VISIBILITY = OOM_ADJ_REASON_METHOD + "_uiVisibility";
    static final String OOM_ADJ_REASON_ALLOWLIST = OOM_ADJ_REASON_METHOD + "_allowlistChange";
    static final String OOM_ADJ_REASON_PROCESS_BEGIN = OOM_ADJ_REASON_METHOD + "_processBegin";
    static final String OOM_ADJ_REASON_PROCESS_END = OOM_ADJ_REASON_METHOD + "_processEnd";
    ...
}
```

## 冻结流程

例如Activity destroy的时候在ActivityRecord.setState里面就会去更新进程状态,更新进程状态的时候就会更新oom adj:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/wm/ActivityRecord.java
WindowProcessController app;      // if non-null, hosting application

void setState(State state, String reason) {
    ...
    switch (state) {
        ...
        case DESTROYING:
            if (app != null && !app.hasActivities()) {
                // Update any services we are bound to that might care about whether
                // their client may have activities.
                // No longer have activities, so update LRU list and oom adj.
                app.updateProcessInfo(true /* updateServiceConnectionActivities */,
                        false /* activityChange */, true /* updateOomAdj */,
                        false /* addPendingTopUid */);
            }
            break;
        ...
    }
    ...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/wm/WindowProcessController.java
void updateProcessInfo(boolean updateServiceConnectionActivities, boolean activityChange,
        boolean updateOomAdj, boolean addPendingTopUid) {
    if (addPendingTopUid) {
        addToPendingTop();
    }
    if (updateOomAdj) {
        prepareOomAdjustment();
    }
    // Posting on handler so WM lock isn't held when we call into AM.
    // 这里是延迟去调用mListener的WindowProcessListener::updateProcessInfo方法,而mListener实际是实现了WindowProcessListener接口的ProcessRecord
    final Message m = PooledLambda.obtainMessage(WindowProcessListener::updateProcessInfo,
            mListener, updateServiceConnectionActivities, activityChange, updateOomAdj);
    mAtm.mH.sendMessage(m);
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/am/ProcessRecord.java
class ProcessRecord implements WindowProcessListener {
    ...
    @Override
    public void updateProcessInfo(boolean updateServiceConnectionActivities, boolean activityChange,
            boolean updateOomAdj) {
        ...
        if (updateOomAdj) {
            mService.updateOomAdjLocked(this, OomAdjuster.OOM_ADJ_REASON_ACTIVITY);
        }
        ...
    }
    ...
}
```

进程oom adj值的重新计算最终会去到OomAdjuster.applyOomAdjLSP,在里面就会调用updateAppFreezeStateLSP去更新进程的进程冻结状态:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
final void updateOomAdjLocked(String oomAdjReason) {
    mOomAdjuster.updateOomAdjLocked(oomAdjReason);
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java
boolean updateOomAdjLocked(ProcessRecord app, String oomAdjReason) {
    synchronized (mProcLock) {
        return updateOomAdjLSP(app, oomAdjReason);
    }
}

private boolean performUpdateOomAdjLSP(ProcessRecord app, String oomAdjReason) {
    ...
    applyOomAdjLSP(app, false, SystemClock.uptimeMillis(),
                        SystemClock.elapsedRealtime(), oomAdjReason);
    ...
}

private boolean applyOomAdjLSP(ProcessRecord app, boolean doingAll, long now,
            long nowElapsed, String oomAdjReson) {
  ...
  updateAppFreezeStateLSP(app);
  ...
}
```

updateAppFreezeStateLSP里面判断adj >= CACHED\_APP\_MIN\_ADJ(900)的时候就会去调用freezeAppAsyncLSP, 进程的adj在900 ~ 999代表它只有不可见的activity,可以随时被干掉,所以我们去冻结它也不会有影响:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java
private void updateAppFreezeStateLSP(ProcessRecord app) {
    ...
    final ProcessStateRecord state = app.mState;
    // Use current adjustment when freezing, set adjustment when unfreezing.
    if (state.getCurAdj() >= ProcessList.CACHED_APP_MIN_ADJ && !opt.isFrozen()
            && !opt.shouldNotFreeze()) {
        mCachedAppOptimizer.freezeAppAsyncLSP(app);
    } else if (state.getSetAdj() < ProcessList.CACHED_APP_MIN_ADJ) {
        mCachedAppOptimizer.unfreezeAppLSP(app, oomAdjReason);
    }
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/am/ProcessList.java

// This is a process only hosting activities that are not visible,
// so it can be killed without any disruption.
public static final int CACHED_APP_MAX_ADJ = 999;
public static final int CACHED_APP_MIN_ADJ = 900;
```

freezeAppAsyncLSP里面会post一个10分钟的message在时间到了的时候去冻结进程(就是10分钟之后调用Process.setProcessFrozen):

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/am/CachedAppOptimizer.java
@VisibleForTesting static final long DEFAULT_FREEZER_DEBOUNCE_TIMEOUT = 600_000L;
@VisibleForTesting volatile long mFreezerDebounceTimeout = DEFAULT_FREEZER_DEBOUNCE_TIMEOUT;

void freezeAppAsyncLSP(ProcessRecord app) {
    final ProcessCachedOptimizerRecord opt = app.mOptRecord;
    if (opt.isPendingFreeze()) {
        // Skip redundant DO_FREEZE message
        return;
    }

    mFreezeHandler.sendMessageDelayed(
            mFreezeHandler.obtainMessage(
                SET_FROZEN_PROCESS_MSG, DO_FREEZE, 0, app),
            mFreezerDebounceTimeout);
    ...
}

public void handleMessage(Message msg) {
    switch (msg.what) {
        case SET_FROZEN_PROCESS_MSG:
            synchronized (mAm) {
                freezeProcess((ProcessRecord) msg.obj);
            }
            break;
        ...
    }
}

private void freezeProcess(final ProcessRecord proc) {
    ...
    Process.setProcessFrozen(pid, proc.uid, true);
    ...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/core/java/android/os/Process.java

/**
 * Freeze or unfreeze the specified process.
 *
 * @param pid Identifier of the process to freeze or unfreeze.
 * @param uid Identifier of the user the process is running under.
 * @param frozen Specify whether to free (true) or unfreeze (false).
 *
 * @hide
 */
public static final native void setProcessFrozen(int pid, int uid, boolean frozen);
```

总结一下就是,如果进程的oom adj大于CACHED\_APP\_MIN\_ADJ,就会启动一个10分钟的定时器,在10分钟之内如果进程的oom adj一直没有变回小于CACHED\_APP\_MIN\_ADJ就会冻结进程。

## 解冻流程

同样Activity start的时候在ActivityRecord.setState里面就会去调用WindowProcessController.updateProcessInfo更新进程状态,更新进程状态的时候就会更新oom adj:

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/wm/ActivityRecord.java
WindowProcessController app;      // if non-null, hosting application

void setState(State state, String reason) {
    ...
    switch (state) {
        ...
        case STARTED:
            ...
            app.updateProcessInfo(false /* updateServiceConnectionActivities */,
                    true /* activityChange */, true /* updateOomAdj */,
                    true /* addPendingTopUid */);
            ...
        ...
    }
    ...
}
```

最终也是会去到OomAdjuster.updateAppFreezeStateLSP,调用链路在上面的冻结流程里面已经追过,这里就省略了。可以看到如果adj小于CACHED\_APP\_MIN\_ADJ就会调用CachedAppOptimizer.unfreezeAppLSP进行解冻:


```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java
private void updateAppFreezeStateLSP(ProcessRecord app) {
    ...
    final ProcessStateRecord state = app.mState;
    // Use current adjustment when freezing, set adjustment when unfreezing.
    if (state.getCurAdj() >= ProcessList.CACHED_APP_MIN_ADJ && !opt.isFrozen()
            && !opt.shouldNotFreeze()) {
        mCachedAppOptimizer.freezeAppAsyncLSP(app);
    } else if (state.getSetAdj() < ProcessList.CACHED_APP_MIN_ADJ) {
        mCachedAppOptimizer.unfreezeAppLSP(app, oomAdjReason);
    }
}
```


最终去到CachedAppOptimizer.unfreezeAppInternalLSP里面,如果还在10分钟的后悔时间里面就直接removeMessages删除定时器,如果进程已经冻结了就调用Process.setProcessFrozen解冻进程(frozen参数传入false)

```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/am/CachedAppOptimizer.java
void unfreezeAppLSP(ProcessRecord app, String reason) {
    synchronized (mFreezerLock) {
        unfreezeAppInternalLSP(app, reason);
    }
}

void unfreezeAppInternalLSP(ProcessRecord app, String reason) {
    final int pid = app.getPid();
    final ProcessCachedOptimizerRecord opt = app.mOptRecord;
    if (opt.isPendingFreeze()) {
        // Remove pending DO_FREEZE message
        mFreezeHandler.removeMessages(SET_FROZEN_PROCESS_MSG, app);
        opt.setPendingFreeze(false);
        ...
    }

    opt.setFreezerOverride(false);
    if (pid == 0 || !opt.isFrozen()) {
        return;
    }

    ...
    Process.setProcessFrozen(pid, app.uid, false);
    ...
}
```

上面例子中,整个从退出Activity冻结进程到进入Activity解冻进程的流程如下:

{% plantuml %}
state "Activity destroy" as ActivityDestroy
state "设置定时器,十分钟后冻结进程" as CreateTimer
state "删除定时器" as DeleteTimer
state "冻结进程" as Freeze
state "解冻进程" as Unfreeze

[*] -down-> ActivityDestroy
ActivityDestroy --> CreateTimer : oom adj >= CACHED_APP_MIN_ADJ
CreateTimer --> Freeze : 十分钟到
Freeze --> Unfreeze : Activity start使得 oom adj < CACHED_APP_MIN_ADJ
Unfreeze --> [*]
CreateTimer --> DeleteTimer : 十分钟内Activity start使得 oom adj < CACHED_APP_MIN_ADJ
DeleteTimer --> [*]
{% endplantuml %}

# 问题定位与规避

从日志上看这个进程在被kill的时候adj就是905:

```
08-29 14:13:11.716499  7904 27946 I ActivityManager: Killing 26859:me.linjw.demo.anr/1000 (adj 905): bg anr
```

而且它的启动时间和冻结时间刚好差10分钟:

```
08-29 14:01:45.124651  7904  8283 I ActivityManager: Start proc 26859:me.linjw.demo.anr/1000 for service {me.linjw.demo.anr/me.linjw.demo.anr.RemoteService}
08-29 14:11:45.885503  7904  8331 I am_freeze: [26859,me.linjw.demo.anr]
```

也就是说应用进程启动的时候adj就是905,然后就设置了10分钟的进程冻结定时器。

问题在于我们的应用的确只有一个Service,没有启动Activity而是通过WindowManager.addView添加的全局浮窗。

addView源码太多我没有找到更新oom adj的逻辑,但是复现问题使用`cat /proc/{pid}/oom_adj`命令获取oom adj发现并不是大于900的,也复现不出10分钟被冻结的现象。

那有可能是的确没有,也有可能是在某种情况下没有更新成功。在日志里没有看到任何报错,问题转给系统哥估计也解决不了,只能应用规避了。


规避的方式也很简单,将服务[设置成前台服务](https://developer.android.com/guide/components/services?hl=zh-cn#Foreground)主动触发OOM\_ADJ\_REASON\_UI\_VISIBILITY类型的oom adj重新计算:


```java
// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
private void updateServiceForegroundLocked(ProcessServiceRecord psr, boolean oomAdj) {
    ...
    mAm.updateProcessForegroundLocked(psr.mApp, anyForeground, fgServiceTypes, oomAdj);
    ...
}

// https://cs.android.com/android/platform/superproject/+/android-13.0.0_r74:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
final void updateProcessForegroundLocked(ProcessRecord proc, boolean isForeground,
        int fgServiceTypes, boolean oomAdj) {
    ...
    if (oomAdj) {
        updateOomAdjLocked(proc, OomAdjuster.OOM_ADJ_REASON_UI_VISIBILITY);
    }
}
```
