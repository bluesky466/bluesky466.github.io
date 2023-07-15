title: Windows发送模拟按键的坑
date: 2023-07-15 17:14:42
tags: 
    - 技术相关
    - Windows
---

最近支援C++兄弟的项目里面有在windows下发送模拟按键的需求,整个功能做下来发现了不少的坑,这里记录下来。

首先Windows上发送模拟按键可以用[SendInput](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput)或者[keybd_event](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-keybd_event)去实现,而keyb\_event文档里面也说更推荐用SendInput,所以我也选用了它:

> Note  This function has been superseded. Use SendInput instead.

SendInput的用法也很简单,下面是官方文档提供的Demo:

```c++
// https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput#example
void ShowDesktop()
{
    OutputString(L"Sending 'Win-D'\r\n");
    INPUT inputs[4] = {};
    ZeroMemory(inputs, sizeof(inputs));

    inputs[0].type = INPUT_KEYBOARD;
    inputs[0].ki.wVk = VK_LWIN;
   
    inputs[1].type = INPUT_KEYBOARD;
    inputs[1].ki.wVk = VK_D;

    inputs[2].type = INPUT_KEYBOARD;
    inputs[2].ki.wVk = VK_D;
    inputs[2].ki.dwFlags = KEYEVENTF_KEYUP;

    inputs[3].type = INPUT_KEYBOARD;
    inputs[3].ki.wVk = VK_LWIN;
    inputs[3].ki.dwFlags = KEYEVENTF_KEYUP;

    UINT uSent = SendInput(ARRAYSIZE(inputs), inputs, sizeof(INPUT));
    if (uSent != ARRAYSIZE(inputs))
    {
        OutputString(L"SendInput failed: 0x%x\n", HRESULT_FROM_WIN32(GetLastError()));
    } 
}

```

# 服务中无法发送按键

代码写完之后在本地调试下工作正常,但是部署到正式环境之后没有效果。经过对比发现本机调试时候是直接运行exe,而正式环境下我们的程序实际是作为Windows的一个服务在运行,在服务中调用SendInput没有作用。

原因是Windows服务运行在非交互会话中，这意味着服务所使用的窗口会话并非是当前正在登录用户的会话或默认会话。在这种情况下使用SendInput函数也会失败。因为SendInput函数必须将输入消息发送到活动窗口或当前焦点所在的窗口，而在非交互会话中默认没有活动窗口或当前焦点。

其实这个在服务开发里面还是挺经常遇到的,常规的方式就是使用[CreateProcessAsUserW](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessasuserw)在当前登录用户的会话中创建子进程去调用SendInput(代码见[Github](https://github.com/bluesky466/WindowsCreateProcess),参考CreateProcessAsCurrentUser(cmd, false))。

# 任务管理器接收不到模拟按键

加上子进程之后正常情况功能是正常了,但是在部分场景下发送模拟按键还是没有效果,例如焦点在任务管理器的情况下。这个问题在[文档](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput)里面其实也有提及:

>  This function is subject to UIPI. Applications are permitted to inject input only into applications that are at an equal or lesser integrity level.

只能往和自己同样或者低权限的程序发送按键。所以解决的思路就是让子进程以管理员权限运行,搜索之后发现的确是可行的,在系统服务中可以用GetTokenInformation获得一个TokenLinkedToken去启动一个管理员身份的进程:

```
我们用GetTokenInformation可以获得一个TokenLinkedToken，简单的说就是要获得与我们进程token关联的token。

接下来就有趣了，如果当你的进程是一个提权的管理员权限的进程，那么你获得的token会是一个标准用户进程的token，也就是一个提权之前进程。那么这有什么用呢？比如我们的子程序需要运行其他开发者开发的插件，而我们不想给予他们过高的权限，那么这个就有用了。当然，如果你更谨慎一些，你希望给予他更低的权限，那就得实用CreateRestrictedToken来创建一个新的token了。

聪明的程序员看到这里肯定就会想，既然管理员权限下的进程获得的TokenLinkedToken是一个标准用户权限的token，那么标准用户权限环境下的进程能不能获得一个管理员权限的TokenLinkedToken呢？没错，答案是可以。更聪明的程序员肯定会惊讶，那这个不是安全漏洞么？答案是并不是，因为虽然可以获得一个管理员权限的token，但是这个token只是一个IDENTIFY level token，这是一个token的_SECURITY_IMPERSONATION_LEVEL，不同的模仿等级，对应于不同的功能。比如SecurityIdentification，这个等级就只能用来查询token的信息。比方说有外部一个进程访问我们的进程，我们可以让他提供token验证其身份。但是外部进程为了防止我们用他的token干坏事，所以只给我们一个IDENTIFY level token，这样一来，我们就只能验证身份而无法做其他事情了。

我们真的没办法通过TokenLinkedToken获得可以使用的管理员身份的token了么？也不是，我们确实有办法获得能够使用的管理员身份的token。但是有个前提，我们的进程必须有SeTcbPrivilege权限。那这不也是个安全漏洞么？不是，因为SeTcbPrivilege是SYSTEM用户的权限，简单的说，这个用户的权限比管理员还要高。那这玩意不是也没什么用么？也有用，当你想在系统服务中启动一个管理员身份的进程的时候，可以先获得标准用户权限的token，然后获得其TokenLinkedToken，最后CreateProcessAsUser来创建进程。

--- 来自https://0cch.com/2018/08/24/tokenlinkedtoken-tip/
```

PS: 代码见[Github](https://github.com/bluesky466/WindowsCreateProcess),参考CreateProcessAsCurrentUser(cmd, true)

# 方向键被识别成小键盘数字键

本来以为已经没有问题了，但是测试发现在某些场景下,上下作用的方向键会被识别成2468的数字键。例如开始菜单的搜索栏会概率出现，而我在debug的时候发现必现的场景是快捷方式的快捷键设置那里,而且windows自动的虚拟键盘也是有同样的问题:

{% img /Windows发送模拟按键的坑/1.png %}

会将 VK\_LEFT、VK\_UP、VK\_RIGHT、VK\_DOWN 识别成 VK\_NUMPAD4、VK\_NUMPAD8、VK\_NUMPAD6、VK\_NUMPAD2。

用[键盘检测工具](https://myclickspeed.com/keyboard-test-utility/)检测按键码,发现发送的的确是VK\_LEFT、VK\_UP、VK\_RIGHT、VK\_DOWN:

{% img /Windows发送模拟按键的坑/3.png %}


后面发现[第三方的虚拟键盘](https://freevirtualkeyboard.com/)没有这个问题:

{% img /Windows发送模拟按键的坑/2.png %}


用键盘检测工具去检查,发现这两者的区别在于这个第三方键盘会设置按键的扫描码,而且在发送方向键的时候还会设置KEYEVENTF\_EXTENDEDKEY:

{% img /Windows发送模拟按键的坑/4.png %}


我尝试了下,实际上不需要设置扫描码,只需要把KEYEVENTF\_EXTENDEDKEY这个flag加上问题也解决了:

```C++
INPUT ip = {0};
ip.type = INPUT_KEYBOARD;
ip.ki.wVk = VK_LEFT;
ip.ki.dwFlags = flag | KEYEVENTF_EXTENDEDKEY;
SendInput(1, &ip, sizeof(INPUT));
```

从[文档](https://learn.microsoft.com/en-us/windows/win32/inputdev/about-keyboard-input#extended-key-flag)来看,像方向键和INS, DEL, HOME, END, PAGE UP, PAGE DOWN这些按键,作为拓展键,他们的扫描码会在在数据前拼上0xE0:

```
The extended-key flag indicates whether the keystroke message originated from one of the additional keys on the Enhanced 101/102-key keyboard. The extended keys consist of the ALT and CTRL keys on the right-hand side of the keyboard; the INS, DEL, HOME, END, PAGE UP, PAGE DOWN, and arrow keys in the clusters to the left of the numeric keypad; the NUM LOCK key; the BREAK (CTRL+PAUSE) key; the PRINT SCRN key; and the divide (/) and ENTER keys in the numeric keypad. The right-hand SHIFT key is not considered an extended-key, it has a separate scan code instead.

If specified, the scan code consists of a sequence of two bytes, where the first byte has a value of 0xE0.
```

从[扫描码对照表](https://learn.microsoft.com/en-us/windows/win32/inputdev/about-keyboard-input#scan-codes)里面也可以看出来。

方向键的扫描码是0xE048、0xE050、0xE04B、0xE04D, 如果没有前面的0xE0，变成0x48、0x50、0x4B、0x4D就可能被识别成方向键或者小键盘的数字键:

|HID Usage Page | HID Usage ID | HID Usage Name  | Key Location  |  Scan 1 Make |
|-|-|-|-|-|
|07|52|Keyboard UpArrow|83|E0 48|
|07|51|Keyboard DownArrow|84|E0 50|
|07|50|Keyboard LeftArrow|79|E0 4B|
|07|4F|Keyboard RightArrow|89|E0 4D|
|07|60|Keypad 8 and Up Arrow|96|48|
|07|5A|Keypad 2 and Down Arrow|98|50|
|07|5C|Keypad 4 and Left Arrow|92|4B|
|07|5E|Keypad 6 and Right Arrow |102|4D|

于是猜测某些程序会将虚拟按键码转换成扫描码去做处理，在没有设置KEYEVENTF\_EXTENDEDKEY的时候可能转换出来就识别成了小键盘数字键。

