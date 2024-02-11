title: Windows Dump文件分析
date: 2024-02-11 10:36:39
tags:
    - 技术相关
    - Windows
---
# dump文件获取

Windows上处理程序crash的问题可以通过分析dump文件来定位问题。那怎么拿到dump文件呢?有几种方式可以获取。

## 注册表配置dump文件生成目录

像我司生产的整机可以根据[Windows官方文档](https://learn.microsoft.com/zh-cn/windows/win32/wer/collecting-user-mode-dumps)在出厂的时候就预埋注册表`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps`配置dump文件生成的路径:

{% img /WindowsDump文件分析/1.png %}

像这里配置到了`%localappdata%\CrashDumps`由于`%localappdata%`是个环境变量,所以当程序崩溃的时候会不同的用户会在存在不同的目录下产生dump文件,系统服务和普通应用的目录也不一样:

- 普通应用 : `C:\Users\你的用户名\AppData\Local\CrashDumps`
- 系统服务 : `C:\Windows\system32\config\systemprofile\AppData\Local\CrashDumps`

它也支持在`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps`下添加子项,为指定的exe进行单独配置:

{% img /WindowsDump文件分析/2.png %}

像这样的Demo2.exe就会在`D:\dump2`下生成dump文件。

而像windows出现蓝屏的情况重启之后就可以看`C:\WINDOWS\MEMORY.DMP`这个dump文件了,像我们这段时间做的项目,有个驱动引发蓝屏的问题就是看这个dump文件去定位分析的。

## 使用MiniDumpWriteDump生成dump文件

我们也可以使用[SetUnhandledExceptionFilter](https://learn.microsoft.com/zh-cn/windows/win32/api/errhandlingapi/nf-errhandlingapi-setunhandledexceptionfilter)注册异常处理函数,在函数里面调用[MiniDumpWriteDump](https://learn.microsoft.com/zh-cn/windows/win32/api/minidumpapiset/nf-minidumpapiset-minidumpwritedump)生成dump文件:

```c++
#include <windows.h>
#include<DbgHelp.h> 
#pragma comment(lib,"DbgHelp.lib")  

LONG ApplicationCrashHandler(EXCEPTION_POINTERS* pException) {
    HANDLE hDumpFile = CreateFileW(L"DemoDump.dmp", GENERIC_WRITE, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
    MINIDUMP_EXCEPTION_INFORMATION dumpInfo;
    dumpInfo.ExceptionPointers = pException;
    dumpInfo.ThreadId = GetCurrentThreadId();
    dumpInfo.ClientPointers = TRUE;
    MiniDumpWriteDump(GetCurrentProcess(), GetCurrentProcessId(), hDumpFile, MiniDumpNormal, &dumpInfo, NULL, NULL);
    CloseHandle(hDumpFile);
    return EXCEPTION_EXECUTE_HANDLER;
}

int main()
{
    //注册异常处理函数  
    SetUnhandledExceptionFilter((LPTOP_LEVEL_EXCEPTION_FILTER)ApplicationCrashHandler);

    ... // 业务代码
    return 0;
}
```

上面的代码虽然可以打出dump文件,但是里面的信息比较少,如果想要打出更多的信息可以参考[crashdump](https://github.com/Arnavion/crashdump)

# dump文件分析

拿到dump文件之后就要看看如何分析了,这里写一个简单的demo:

```c++
#include <iostream>

using namespace std;

struct Data {
    char* str;
};

void foo(Data* data) {
    cout << std::strlen(data->str)<<endl;
}

int main()
{
    Data data = { nullptr };
    foo(&data);
    return 0;
}
```

我们可以使用WinDbg去打开dump文件:

{% img /WindowsDump文件分析/3.png %}

注意将pdb文件放到符号目录下,例如我这里配置的是D盘:

{% img /WindowsDump文件分析/4.png %}

然后就可以用`!analyze -v`命令先让WinDbg简单分析一下:

```
...
ExceptionAddress: 00007ffdcf587cc1 (ucrtbased!strlen+0x0000000000000031)
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 0000000000000000
   Parameter[1]: 0000000000000000
Attempt to read from address 0000000000000000
...
```

例如从上面的信息我们可以看到奔溃的原因是strlen里面读取了空指针。

然后使用`kb`命令调用堆栈查看完整调用链路:

```
0:000> kb
 # RetAddr               : Args to Child                                                           : Call Site
00 00007ffe`aaf9ffe9     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!NtWaitForMultipleObjects+0x14
01 00007ffe`aaf9feee     : 00000000`000000bc 00000000`00000096 00000000`d000022d 00000000`d000022d : KERNELBASE!WaitForMultipleObjectsEx+0xe9
02 00007ffe`acc527f7     : 00000000`00000000 00000000`00000001 0000008c`ec2fee00 00000000`00000096 : KERNELBASE!WaitForMultipleObjects+0xe
03 00007ffe`acc52236     : 00000000`00000000 00000000`00000000 00000000`00000003 00007ffe`acbe0000 : kernel32!WerpReportFaultInternal+0x587
04 00007ffe`ab09cafb     : 00000000`00000000 0000008c`ec2ffec0 00000000`00000001 00007ffd`cf512e59 : kernel32!WerpReportFault+0xbe
05 00007ffe`adaf8abd     : 00000000`0005aa79 00007ffe`adbb2b34 00000000`00000000 00007ffe`ada70b5a : KERNELBASE!UnhandledExceptionFilter+0x3db
06 00007ffe`adadf197     : 0000008c`ec2feed0 00000000`00000000 0000008c`ec2fee88 0000008c`ec2ff470 : ntdll!RtlUserThreadStart$filt$0+0xac
07 00007ffe`adaf441f     : 00000000`00000000 0000008c`ec2ff3d0 0000008c`ec2ffab0 0000008c`ec2ffab0 : ntdll!_C_specific_handler+0x97
08 00007ffe`ada6e466     : 0000008c`ec2ffab0 00007ffe`ada50000 00007ffe`adaaaa58 00007ffe`adbdebf8 : ntdll!RtlpExecuteHandlerForException+0xf
09 00007ffe`adaf340e     : 000001b5`00250000 00000000`00000000 00000000`00000000 00007ffd`00000000 : ntdll!RtlDispatchException+0x286
0a 00007ffd`cf587cc1     : 00007ff7`8d053837 cccccccc`cccccccc 00007ffd`cf4b0dab 00000000`00000000 : ntdll!KiUserExceptionDispatch+0x2e
0b 00007ff7`8d053837     : cccccccc`cccccccc 00007ffd`cf4b0dab 00000000`00000000 00007ff7`8d052367 : ucrtbased!strlen+0x31 [minkernel\crts\ucrt\src\appcrt\string\amd64\strlen.asm @ 70] 
0c 00007ff7`8d05389a     : 0000008c`ec2ffd48 00000000`00000000 00000000`00000000 00007ffd`cf4b0769 : Demo!foo+0x17 [C:\Users\user\workspace\CppAutoRegisterDemo\main.cpp @ 11] 
0d 00007ff7`8d05c7c9     : 00000000`00000001 00007ffd`cf5322a3 00000000`00000000 00007ff7`8d05bc4d : Demo!main+0x2a [C:\Users\user\workspace\CppAutoRegisterDemo\main.cpp @ 18] 
0e 00007ff7`8d05c6ae     : 00007ff7`8d065000 00007ff7`8d065350 00000000`00000000 00000000`00000000 : Demo!invoke_main+0x39 [D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 79] 
0f 00007ff7`8d05c56e     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : Demo!__scrt_common_main_seh+0x12e [D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288] 
10 00007ff7`8d05c85e     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : Demo!__scrt_common_main+0xe [D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 331] 
11 00007ffe`acbf257d     : 0000008c`ec07d000 00000000`00000000 00000000`00000000 00000000`00000000 : Demo!mainCRTStartup+0xe [D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_main.cpp @ 17] 
12 00007ffe`adaaaa58     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : kernel32!BaseThreadInitThunk+0x1d
13 00000000`00000000     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!RtlUserThreadStart+0x28
```

从这一行可以看到foo的frame id是0c:

> 0c 00007ff7`8d05389a     : 0000008c`ec2ffd48 00000000`00000000 00000000`00000000 00007ffd`cf4b0769 : Demo!foo+0x17 [C:\Users\user\workspace\CppAutoRegisterDemo\main.cpp @ 11] 

所以我们可以用`.frame 0c`查看这个frame的信息:

{% img /WindowsDump文件分析/5.png %}

从Locals信息里面可以看到data这个指针的值是0x0000008cec2ffd48,而data->str的值则为0x0，所以我们就能定位出是data->str空指针导致strlen出现异常。

有时候奔溃也可能和多线程还有锁相关，使用到的WinDbg命令可以参考我之前的[博客](https://blog.islinjw.cn/2023/08/31/Windows%E8%B0%83%E8%AF%95%E6%8A%80%E5%B7%A7%E6%A1%88%E4%BE%8B-ffmpeg-wave%E6%92%AD%E6%94%BE%E9%9F%B3%E9%A2%91%E5%8D%A1%E6%AD%BB%E9%97%AE%E9%A2%98/)