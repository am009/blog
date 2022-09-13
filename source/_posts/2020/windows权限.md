---
title: Windows权限与UAC
date: 2020/6/1 11:11:11
categories:
- Dev
tags:
- Windows
---

# Windows权限与UAC

这篇文章是课程中被逼无奈（被一门2学分的课的老师疯狂压榨时间）调研Windows相关的攻击，UAC绕过中总结的一些笔记

<!-- more -->

## 综述

在windows中有两种访问控制，首先是强制完整性控制，它蕴含在SACL中。其次是自主访问控制DACL。

SACL原本的用途是类似自主访问控制一样自主控制系统日志，但完整性控制加入到了它的范围内。

UAC提权不仅完整性从medium到了high，而且多了特权。特权是另外的资源。

UAC绕过的例外方法：提升的COM名字对象，UIAccess

从admin到system的方法：令牌窃取。

### 强制完整性控制（MIC）

windows 首先是自主访问控制, 只有很简单的强制访问控制, 在原本只是打印日志的SACL的完整性字段里. 实现的方式是"可上读, 可下写". 为了保证完整性, 只需要严格控制写权限. TODO: 能不能下读.

保护完整性的强制访问控制: 低级的不可以（发送消息）影响高级的程序。

与之对应的是保护机密性的强制访问控制：低级的可以发消息给高级程序，而高级程序无法发消息给低级程序（泄露机密。）

windows中的完整性等级：

```
SeUntrustedMandatorySid
SeLowMandatorySid
SeMediumMandatorySid
SeHighMandatorySid
SeSystemMandatorySid
```

主体的缺省完整性级别是SeUntrustedMandatorySid，而客体的缺省完整性级别是SeMediumMandatorySid

> 进程完整性级别是为了保证不同标签的进程（TOKEN) 和对象（SACL)之间的访问安全
> https://zh.wikipedia.org/wiki/%E5%BC%BA%E5%88%B6%E5%AE%8C%E6%95%B4%E6%80%A7%E6%8E%A7%E5%88%B6
>
> [SID](https://en.wikipedia.org/wiki/Security_Identifier)
>
> 可以看到标准用户和经过权限提升的UAC用户信息的差别。用户名项中组信息和sid均相同，区别就是UAC用户是经过权限提升的，最终体现在权限上的不同如下：
>
> 1.组信息项中主要是Integrity levels (IL)【进程完整性级别不同】。标准用户是Medium Mandatory Level，UAC用户是High Mandatory Level,它包括Untrust， Low， Medium， Hight， System等， 级别越低，权限也就越低。我们可以通过GetTokenInformation的TokenIntegrityLevel来进行查询。
> 2.体现在Privilege中的就是UAC用户拥有很多Privilege，比如最常用的SeDebugPrivilege 。
> 注释：
> 进程完整性级别是为了保证不同标签的进程（TOKEN) 和对象（SACL)之间的访问安全，如果当前进程的TOKEN 是Low Mandatory Level， 它就不能修改具有Medium Mandatory Level的对象，即使我们对象的DACL赋予完全读写的权限。当每个进程打开对象， 我们会进行SACL和DACL检查，这个检查通过核心态函数
> SeAccessCheck . 只有当前进程TOKEN的　完整性标签　高于或者等于　对象的完整性标，　我们才会进一步进行 DACL 检查。如果完整性标签验证通不过。 即使DACL给予再高权限都无济于事。


### DACL 自主访问控制列表
文件/注册表方面自然是还有自主访问控制. 进程没想到!!和文件一样的, 有安全描述符
主体是访问令牌. 每个进程都有一个基本令牌 (Primary Token)，可以被进程中的每个线程所共享, 后面线程可以获得其他令牌. 令牌里有用户sid, 组sid, 受限sid, 特权, 身份模拟级别, 完整度级别.
客体是对象: 文件/注册表/进程. 客体关联了安全描述符, 安全描述符包括所有者SID, 组SID, DACL, SACL(日志)

“属性” => “安全”, “权限”选项卡就是DACL，”审核”选项卡是SACL，“所有者”是Owner、Group。

每个DACL内有很多ACE, 访问控制表项, 可以接受也可以拒绝, 先找到的生效. 因此(有时?)一般拒绝的放在接受的前面.


impersonation 身份模拟 传输层, 被rpc使用的, 服务端可以使用客户端的令牌.

令牌里的权限有的没有enable, 要单独开启
```
+0x000 Present          : Uint8B
+0x008 Enabled          : Uint8B
+0x010 EnabledByDefault : Uint8B
```
特权Privilege在访问某个具体的安全对象时并没有作用，Privilege是表示进程是否能够进行特定的系统操作，如关闭系统、修改系统时间、加载设备驱动等。

### UAC

当用户登录Windows时，操作系统会为用户生成一对初始令牌，分别是代表着用户所拥有的全部权限的完整版本令牌(即管理员权限令牌)，以及被限制管理员权限后的普通令牌，二者互为关联令牌;此后，代表用户的进程所使用的令牌都是由普通令牌继承而来，用来进行常规的、非敏感的操作;当用户需要进行一些需要管理员权限的操作时，比如安装软件、修改重要的系统设置时，都会通过弹出提权对话框的形式提示用户面临的风险，征求用户的同意，一旦用户同意，将会切换到当前普通令牌关联的管理员权限令牌，来进行敏感操作。通过这种与用户交互的方式，避免一些恶意程序在后台稍稍执行敏感操作。
[来自](http://blog.nsfocus.net/analysis-windows-access-authority-inspection-mechanism/)


> Access Token：是一个包含了登陆会话安全信息的 Windows 软件对象，用于指名一个用户以及他所在组以及相应的特权。
> UAC Token：定义了Windows Vista 用户在UAC支持开启的时候的默认交互式登陆特权。一个 UAC Token 定义了最小的运行特权。
> Full Token：给账户提供了最大的经过授权的特权。Full Token 实际上是由该用户隶属于的用户组决定的。

```
通过whoami -all查看当前用户所拥有的Privilege。
```

### 其他


部分受保护进程难以获得debug, 注入等权限. 保护进程的Protection成员不为0. 两种保护类型：Protected Process(PP)，Protected Process Lite(PPL). 对于Signer为PsProtectedSignerWindows(5)和PsProtectedSignerTcb(6)的保护进程, 其Type和Signer信息会被抽取出来, 组合成sid, 保存到基本令牌中的TrustLevelSid成员中

通过创建受限令牌，可以获得一个普通令牌所有拥有的权限集合的一个子集，用来进行一些低权限操作，降低安全风险。

[权限编程需要注意的](http://www.youngroe.com/2015/08/14/Windows/Windows-Permissions-Privilege/)
[权限编程2](http://www.cppblog.com/weiym/archive/2013/08/25/202751.html?opt=admin)

## 用户界面特权隔离 User Interface Privilege Isolation, UIPI

通过结合强制完整性控制，用户界面特权隔离阻止较低等完整性级别（Integrity level）的进程向较高等完整性级别进程的窗口发送消息或者安装钩子，但也有一些消息不被阻止。

[UIAccess](https://docs.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/user-account-control-allow-uiaccess-applications-to-prompt-for-elevation-without-using-the-secure-desktop) 选项The ability to bypass UIPI restrictions across privilege levels is available for UI automation programs by using UIAccess. 但是必须签名并且安装在指定地点.

UAC level: asInvoker 不询问权限, 但是用户可以右键以管理员权限运行. highestAvailable时如果用户在admin用户组则和requireAdministrator一样, 必须以管理员权限运行.

> A lower privilege process cannot:

> - Perform a window handle validation of higher process privilege.
> - SendMessage or PostMessage to higher privilege application windows. These application programming interfaces (APIs) return success but silently drop the window message.
> - Use thread hooks to attach to a higher privilege process.
> - Use Journal hooks to monitor a higher privilege process.
> - Perform dynamic link library (DLL)–injection to a higher privilege process.

## 键盘监控的思路


| API | 适用范围 | 备注 |
|-----|--------|--|
| GetAsyncKeyState |  | 每次获取单个按键的状态, 轮询每个键状态, 效率略低|
|GetKeyboardState| | 一次获取所有的键的状态, 和消息队列相关 |
|SetWindowsHookEx 指定键盘| | |

### SetWindowsHookEx
在回调函数中，我们将接收KeyboardProc的wParam中的虚拟键码和LowLevelKeyboardProc的KBDLLHOOKSTRUCT.vkCode（wParam指向KBDLLHOOKSTRUCT）。

如果m_ThreadId = 0，则消息钩子是全局消息钩子。针对全局消息钩子，你必须将回调函数置于dll中，并且需要编写2个dll来分别处理x86/x64进程。

针对底层键盘钩子，SetWindowsHookEx的HMod参数可以为NULL或者本进程加载的模块（我测试了user32，ntdll）。

WH_KEYBOARD_LL不需要dll中的回调函数，并且能适应x86/x64进程。

WH_KEYBOARD需要两个版本的dll，分别处理x86/x64。但是如果使用x86版本的全局消息钩子，所有的x64线程仍被标记为“hooked”，并且系统在钩子应用上下文执行钩子。类似的，如果是x64，所有的32位的进程将使用x64钩子应用的回调函数。这就是为什么安装钩子的线程必须要有一个消息循环。

hook是在整个桌面环境(desktop)内的. uac是safe desktop, 另外一个桌面

[来自](https://www.anquanke.com/post/id/86403)
[专业文章](https://securelist.com/keyloggers-implementing-keyloggers-in-windows-part-two/36358/)

已知设置钩子是不能跨越完整度保护的. 猜测至少需要最高的完整度级别才能设置全局钩子

> "Process isolation provides a way to extend the authorization model to common extension points for inter-process communication. For example, if an application running at medium integrity were to register a hook to process Windows messages, this hook would not be active in a process running at the high integrity level."

所以可能可以安装但是其实不是全局?


### subsystem: console
是一个过时的东西, 直接用subsystem: windows??
[看这](https://www.devever.net/~hl/win32con)

## 自启动技术
[自启动技术](https://www.jianshu.com/p/cf01fee50fb4)

## bypass uac

[概述](https://cqureacademy.com/cqure-labs/cqlabs-how-uac-bypass-methods-really-work-by-adrian-denkiewicz)

这里研究的是COM接口绕过
[详述](https://y4er.com/post/bypassuac-with-icmluautil/)

## 提权

1. 权限窃取
2. 使用服务的impersonation
真正的关键是分析其中的权限问题

首先是bypass UAC, 得到SeDebugPrivilege, 然后就可以直接用窃取的方式得到System
没有UAC时特权很少, 完整度级别为Medium, 过了UAC就是High

现在就看创建服务了. 如果创建服务获取system不需要高完整度, 那就神了. 否则还是要bypass uac, 那还不如上面的方法.
目前我猜测大概是要高完整度的.


## 远程线程注入
远程线程注入首当其冲就是权限问题, 需要debug权限?? !!
meterpreter 怎么migrate的

打开一个同级别的进程, Medium的完整度的, OpenProcess的时候指定什么权限??

## raw_input方法的主函数的消息循环
发现主函数其实消息非常少... 平时只有一个消息, 开了hook之后每按下和松开都有一个事件.
理解为是winmain的消息传过去的吗... 或者要先给到这边, 才能给到那边winProc
发现如果没有getMessage的循环，无论是Hook的keylog还是rawInput的keylog都不行。创建一个子进程执行而不是在dllmain里面执行好像没什么区别？？？ notepad.exe正常使用？？
我的问题是，notepad.exe这种桌面程序难道不是应该winmain里有自己的Getmessage吗。。。为什么即使是注入也需要自己有getMessage的循环？？