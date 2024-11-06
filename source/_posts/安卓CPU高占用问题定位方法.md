title: 安卓CPU高占用问题定位方法
date: 2024-10-25 22:35:20
tags:
    - 技术相关
    - Android
---


今天测试报了一个问题说我们都某个应用从top命令看cpu占用到了百分之90几,这种问题要怎么分析呢?

首先adb连接上去之后用`logcat | grep 应用进程pid`查看看看有没有频繁打印日志,如果有的话根据日志就能比较简单的定位到代码,但这次我看了下并没有什么输出。

然后可以adb连接上去用Profiler抓一下进程的CPU使用信息,例如下图就可以看到`Thread-2`这个线程一直处于Running的状态没有停过,所以就是这个线程在不停占用CPU:

{% img /安卓CPU高占用问题定位方法/1.png %}

如果某些机器不能使用adb只能用串口查看的话还能怎么做呢?

我们可以用`kill -3 应用进程pid`命令强行在`/data/anr`下生成所有线程的堆栈文件,然后可以一个个线程分析过去看看线程运行到哪里了。当然如果线程数量比较多的时候就比较低效了,尤其是开了混淆的情况下。

但其实堆栈文件里面是有信息可以辅助我们分析是哪个线程不停在占用cpu的,例如`Thread-6`的线程堆栈如下:

```
"Thread-6" prio=5 tid=3 Native
  | group="main" sCount=1 ucsCount=0 flags=1 obj=0x75c801b8 self=0xb400007140d4ca60
  | sysTid=5030 nice=0 cgrp=default sched=0/0 handle=0x6fb6d0bcb0
  | state=R schedstat=( 43469364687935 414576037821 2915248 ) utm=2781704 stm=1565231 core=6 HZ=100
  | stack=0x6fb650c000-0x6fb650e000 stackSize=8191KB
  | held mutexes=
  native: #00 pc 00000000000a22d8  /apex/com.android.runtime/lib64/bionic/libc.so (__ioctl+8) (BuildId: 058e3ec96fa600fb840a6a6956c6b64e)
  native: #01 pc 000000000005b40c  /apex/com.android.runtime/lib64/bionic/libc.so (ioctl+156) (BuildId: 058e3ec96fa600fb840a6a6956c6b64e)
  ...
```

`| state=R schedstat=( 43469364687935 414576037821 2915248 ) utm=2781704 stm=1565231 core=6 HZ=100`这行其实能看到比较有用的信息但是可能很多人会忽略掉。

### state

线程状态,root的情况下我们也可以直接`cat /proc/{pid}/task/{tid}/stat`获取,它和`cat /proc/{pid}/stat`的进程状态类似,从[文档](https://man7.org/linux/man-pages/man5/proc_pid_stat.5.html)看有下面的下面的值:

- R      Running
- S      Sleeping in an interruptible wait
- D      Waiting in uninterruptible disk sleep
- Z      Zombie
- T      Stopped (on a signal) or (before Linux 2.6.33) trace stopped
- t      Tracing stop (Linux 2.6.33 onward)
- W      Paging (only before Linux 2.6.0)
- X      Dead (from Linux 2.6.0 onward)
- x      Dead (Linux 2.6.33 to 3.13 only)
- K      Wakekill (Linux 2.6.33 to 3.13 only)
- W      Waking (Linux 2.6.33 to 3.13 only)
- P      Parked (Linux 3.9 to 3.13 only)
- I      Idle (Linux 4.14 onward)



### schedstat

[schedstat](https://docs.kernel.org/translations/zh_CN/scheduler/sched-stats.html)显示了CPU的调度器统计数据,它有三个数字的含义分别是

1. 在CPU上运行花费的时间(单位是纳秒)
2. 在调度队列上等待的时间(单位是纳秒)
3. CPU调度切换次数

root的情况下我们也可以直接`cat /proc/{pid}/task/{tid}/schedstat`获取

所以`schedstat=( 43469364687935 414576037821 2915248 )`表示这个线程在CPU上运行了43469364687935ns, 等待了414576037821ns, 调度切换了2915248次。

### HZ

CPU时间或者说系统时间是以节拍(tick)为单位进行计算的,[维基百科](https://en.wikipedia.org/wiki/System_time)上是这么说的:

> The system clock is typically implemented as a programmable interval timer that periodically interrupts the CPU, which then starts executing a timer interrupt service routine. This routine typically adds one tick to the system clock (a simple counter) and handles other periodic housekeeping tasks (preemption, etc.) before returning to the task the CPU was executing before the interruption.

而这里的HZ代表的是[CFS Scheduler](https://docs.kernel.org/scheduler/sched-design-CFS.html)的CONFIG\_HZ配置,通过它可以计算出每个cpu节拍是多长,例如上面显示的`HZ=100`代表的是每个cpu节拍为`1s / 100 = 10ms`

### utm、stm

- utime:  用户态下使用了多少个节拍的CPU
- utime:  内核态下使用了多少个节拍的CPU

由于从前面的`HZ=100`的到cpu节拍为10ms,所以`utm=2781704 stm=1565231`代表这个线程在用户态下使用了`2781704 * 10 = `27817040ms`的CPU,在内核态下使用了`1565231 * 10 = 15652310ms`的CPU,

它总的CPU使用总时间就是`27817040ms + 15652310ms = 43469350ms`和前面schedstat算出来的`43469364687935ns = 43469364ms`是匹配的

### core

最后是CPU哪个核在执行这个线程


### 定位方法

有了上面这些信息之后我们就能比较快速的定位是哪个线程在不断占用cpu了:

1. 使用`kill -3 {pid}`命令输出第一份堆栈
2. 等待1分钟
3. 使用`kill -3 {pid}`命令输出第二份堆栈
4. 编写脚本对比两份堆栈里面每个线程的schedstat,看每个线程在这一分钟了使用了多久的cpu


这个脚本代码如下:

```python
import sys
import datetime

def readTrace(path):
	thread_run_time = {}
	with open(path) as f:
		for line in f.readlines():
			if line.startswith("----- pid "):
				trace_time = datetime.datetime.strptime(line.split(" at ")[1].split(".")[0], "%Y-%m-%d %H:%M:%S")
			elif " tid=" in line:
				thread_name = line.strip().split("\"")[1]
			elif "schedstat=(" in line:
				schedstat = line.split("schedstat=( ")[1].split(")")[0].split(" ")
				thread_run_time[thread_name] = (int(schedstat[0]) + int(schedstat[1])) / 1000000000.0
	return trace_time, thread_run_time


if __name__ == '__main__':
	trace_time_start, thread_run_time_start = readTrace(sys.argv[1])
	trace_time_end, thread_run_time_end = readTrace(sys.argv[2])
	duration = (trace_time_end - trace_time_start).total_seconds()
	result = []
	for thread, thread_run_time in thread_run_time_end.items():
		if thread in thread_run_time_start:
			thread_run_in_duration = thread_run_time_end[thread] -  thread_run_time_start[thread]
			result.append([thread, thread_run_in_duration, f"{thread_run_in_duration / duration:.{2}%}"])
	for it in sorted(result, key=lambda it: it[1], reverse=True):
		print(it[0], it[1], it[2])

```

执行结果如下:

```
python3 test.py trace_04 trace_05
Thread-6 100.00285464500485 99.01%
main 0.4229402489999998 0.42%
HeapTaskDaemon 0.1186862900000012 0.12%
Signal Catcher 0.113778791 0.11%
Thread-7 0.11453095000000069 0.11%
DefaultDispatcher-worker-5 0.1086997830000005 0.11%
...
```

最后找到`Thread-6`线程在`09:22:10.977080231`到`09:23:51.419991904`这100秒左右,使用了`43469364687935ns - 43402540633850ns = 66824054085ns`秒的cpu,然后还在调度队列上等待了`414576037821ns - 381397237261ns = 33178800560ns`,`66824054085ns + 33178800560ns = 100002854645ns = 100.002854645s`基本这100s它都在执行了:

```
----- pid 1834 at 2024-10-16 09:22:10.977080231+0800 -----
...
"Thread-6" prio=5 tid=3 Native
  | group="main" sCount=1 ucsCount=0 flags=1 obj=0x75c801b8 self=0xb400007140d4ca60
  | sysTid=5030 nice=0 cgrp=default sched=0/0 handle=0x6fb6d0bcb0
  | state=R schedstat=( 43402540633850 381397237261 2772868 ) utm=2777362 stm=1562891 core=5 HZ=100
  | stack=0x6fb650c000-0x6fb650e000 stackSize=8191KB
  | held mutexes=
  native: #00 pc 00000000000a22d8  /apex/com.android.runtime/lib64/bionic/libc.so (__ioctl+8) (BuildId: 058e3ec96fa600fb840a6a6956c6b64e)
  native: #01 pc 000000000005b40c  /apex/com.android.runtime/lib64/bionic/libc.so (ioctl+156) (BuildId: 058e3ec96fa600fb840a6a6956c6b64e)
  ...
```

```
----- pid 1834 at 2024-10-16 09:23:51.419991904+0800 -----
...
"Thread-6" prio=5 tid=3 Native
  | group="main" sCount=1 ucsCount=0 flags=1 obj=0x75c801b8 self=0xb400007140d4ca60
  | sysTid=5030 nice=0 cgrp=default sched=0/0 handle=0x6fb6d0bcb0
  | state=R schedstat=( 43469364687935 414576037821 2915248 ) utm=2781704 stm=1565231 core=6 HZ=100
  | stack=0x6fb650c000-0x6fb650e000 stackSize=8191KB
  | held mutexes=
  native: #00 pc 00000000000a22d8  /apex/com.android.runtime/lib64/bionic/libc.so (__ioctl+8) (BuildId: 058e3ec96fa600fb840a6a6956c6b64e)
  native: #01 pc 000000000005b40c  /apex/com.android.runtime/lib64/bionic/libc.so (ioctl+156) (BuildId: 058e3ec96fa600fb840a6a6956c6b64e)
```

然后就可以根据堆栈定位到底是哪里的代码逻辑异常不断占用cpu。