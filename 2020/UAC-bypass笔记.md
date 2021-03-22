---
title: UAC-bypass 笔记
date: 2020/12/8 11:11:11
categories:
- Hack
tags:
- Windows
---

# UAC-bypass 笔记

这是之前课程中调研UAC bypass时自己一些零散的记录。

<!-- more -->

[toc]

### UAC的防御

越考虑UAC如何防御，越感觉应该直接把UAC slide拉到最上面

但是确实有几个能绕过Always Notify的方法，这方面确实要防御防御



1. 做攻击demo绕WD, 但是我们能检测
2. dismcore.dll 劫持，直接利用win defender
3. 做一些定向的提前防御手段。做成一个程序。这样反而不用监测了。



能讲的：

1. 比Windows Defender能做得更好的地方。因为讲Win Defender的也就那一篇文章，研究防御的人还是少的，而我们就是研究了Windows defender的检测的人。搞搞花式绕过。目前有注册表跟随符号链接和.local的DLL劫持方面能检测绕过WD的攻击。

2. FIleless attack 只留在注册表的那个攻击。 这就可能会避开windows defender的扫描

   https://github.com/bytecode77/living-off-the-land

   https://enigma0x3.net/2016/08/15/fileless-uac-bypass-using-eventvwr-exe-and-registry-hijacking/

   https://www.cybereason.com/blog/fileless-malware 现在的不少最新恶意软件在用

3. Living off the land 相关攻击

4. 搞一些新的攻击检测，Process reimage attack？

5. 锁屏界面下的键盘启动cmd的后门的检测 [here](https://github.com/hak5darren/USB-Rubber-Ducky/wiki/Attacking-Windows-At-The-Logon-Screen,---Gaining-Access-To-CMD-With-System-Privileges.) 

有些部分windows用这种方法修复了，但类似的部分却没修复。



### 值得讲的大块内容：

注册表劫持的提权攻击-一系列。

COM组件提权 新博客内容。 COM组件提权的历史与修复

winsxs和.local机制 https://www.kernelmode.info/forum/viewtopicb857.html?t=3643&start=90#p28579 ，高权限移动文件的基础方法，sysprep.exe系列的dll劫持，UAC攻击的起源

wow64log机制, .net的profiling dll

UIAccess的消息注入

Manifest的路径漏洞





### 资源

https://www.kernelmode.info/forum/viewtopice732.html?f=11&t=3643 能补充UACMe



### UAC 绕过的历史

 2014年12月：有63种UAC绕过的方法。20%的方法是其他方法的结合。**Win32**/**Simda**. B 这个之前广泛传播的木马，它利用的**ISecurityEditor** 这个方法让微软限制它只能用于文件后，不仅原来的方法失效了，*Application Verifier* dll planting（later this method was also used in some ITW malware）相关的方法也受到了影响。

微软只在有很大负面新闻的时候和大版本更新的时候才搞搞UAC。

*Sysprep.exe*、*inetmgr.exe* 通过Manifest 的***loadFrom*** 加固，unfortunately this was incomplete fix and it took them few more years to finally harden sysprep from dll hijacking。这里面的故事可以讲讲 TODO

***Software\Classess\mscfile\shell\open\command*** 被动了，WD就会报警

*sdclt.exe* ***/kickoffelev\*** 的报警



### UAC 原理

[微软找借口文章1-](https://docs.microsoft.com/en-us/previous-versions/technet-magazine/dd822916(v=msdn.10)) 

[How UAC Works](https://docs.microsoft.com/en-us/previous-versions/bb757008(v=msdn.10))

[UAC Architecture](https://docs.microsoft.com/en-us/previous-versions/bb756945(v=msdn.10)) 

在启动新进程的时候，通过一系列的流程决定是给程序受限令牌还是完整令牌。

通过安全桌面询问用户。

UAC slide UAC提醒级别控制条。AlwaysNotify

 

默认的级别运行可以看到，允许部分系统设置更改的时候提醒。



### UAC自动提升总结

Security through obscurity？

RPC call is made to AIS 。appinfo.dll逆向得到[自动提取的规则](https://medium.com/tenable-techblog/uac-bypass-by-mocking-trusted-directories-24a96675f6e) 

阶段1：Manifest内的AutoElevate/**g_lpAutoApproveEXEList** 

阶段2 微软签名

阶段3 受信任的目录下 C:\Windows\System32 、Program Files等

不能把微软的不提权程序通过修改Manifest成为提权程序，因为 如果自己去更改Manifest，会破坏签名

Manifest中的自动提升 https://technet.microsoft.com/en-us/magazine/2009.07.uac.aspx

1. 被（微软）签名
2. 位于如 C:\Windows\System32 的安全目录**g_lpIncludePFDirs** 中

**g_lpAutoApproveEXEList** 内部的approve表，直接提权

**g_lpIncludePFDirs** 内部的可信文件夹列表



COM Approval List [elevated com moniker](https://docs.microsoft.com/en-us/windows/win32/com/the-com-elevation-moniker) https://swapcontext.blogspot.com/

1. 需要首先在HKEY_CLASSES_ROOT\CLSID\ {CLSID} \Elevation\Enabled = 1 这里启用

1. （RS1）HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\UAC\COMAutoApprovalList 里面也有。这次更新打掉了不少的Bypass的



## 基本操作- Explorer + IFileOperation

注入到Explorer.exe然后使用IFileOperation移动文件。



拓展：[伪装成explorer.EXE](https://github.com/AV1080p/Mottoin-SecPaper/blob/master/UAC%E6%94%BB%E5%87%BB%E5%89%96%E6%9E%90.md#0x03-%E6%8F%90%E6%9D%83%E6%96%87%E4%BB%B6%E6%93%8D%E4%BD%9C) 



## DLL劫持

https://www.freebuf.com/articles/system/83369.html

程序可以通过DLL实现拓展功能。由于DLL的灵活性，程序可以动态判断是否有某个DLL，如果有则加载，并使用相关的功能，没有则不加载，从而实现插件等功能。但这也为DLL劫持留下了机会。

可能是debug用途的劫持，可能是搜索路径靠前的劫持。

大多数的DLL劫持的方法需要高权限移动文件。这是第一道防线

接下来需要防住劫持的点，这是第二道防线。

最后是一些通用的劫持防御，比如system32文件夹下的可执行文件增加的检测，system32文件夹下的.local劫持检测，这是第三道防线。

通常情况下是不会有这些文件的，而且也不可能有。

然而sysmon只支持检测文件的创建：

1. 检测创建文件时的路径里是否含有.exe.local\
2. ImageLoad里面增加Include，判断路径里是否含.exe.local\

缺点：文件剪切粘贴会被绕过、无法检测文件夹创建

winsxs的dot-local为DLL劫持提供了更多的机会 

[DLL搜索路径](https://docs.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order) 

一些可劫持的exe和dll组合的统计：

```
C:\windows\System32\sysprep\sysprep.exe
C:\Windows\System32\Sysprep\SHCORE.dll
C:\Windows\System32\Sysprep\OLEACC.DLL

C:\windows\System32\cliconfg.exe
C:\Windows\System32\NTWDBLIB.DLL

C:\windows\System32\pwcreator.exe
C:\Windows\System32\vds.exe
C:\Program Files\Common Files\microsoft shared\ink\CRYPTBASE.dll
C:\Program Files\Common Files\microsoft shared\ink\CRYPTSP.dll
C:\Program Files\Common Files\microsoft shared\ink\dwmapi.dll
C:\Program Files\Common Files\microsoft shared\ink\USERENV.dll
C:\Program Files\Common Files\microsoft shared\ink\OLEACC.dll

C:\windows\System32\cliconfg.exe
C:\Windows\System32\NTWDBLIB.DLL

C:\windows\System32\pwcreator.exe
C:\Windows\System32\vds.exe
C:\Windows\System32\UReFS.DLL

C:\windows\ehome\Mcx2Prov.exe
C:\Windows\ehome\CRYPTBASE.dll

C:\windows\System32\sysprep\sysprep.exe
C:\Windows\System32\sysprep\CRYPTSP.dll
C:\windows\System32\sysprep\CRYPTBASE.dll
C:\Windows\System32\sysprep\RpcRtRemote.dll
C:\Windows\System32\sysprep\UxTheme.dll

C:\windows\System32\cliconfg.exe
C:\Windows\System32\NTWDBLIB.DLL
```

首先第一层防线是防止高权限移动文件。

dll劫持一方面攻击者可能发现各种各样的新dll可以劫持，拦截特定路径下的特定dll不太好

需要更好的办法

## winsxs

https://www.kernelmode.info/forum/viewtopic2782.html?t=3643&start=110#p28833

https://docs.microsoft.com/zh-cn/archive/blogs/junfeng/dotlocal-local-dll-redirection

http://www.hexacorn.com/blog/2015/01/09/beyond-good-ol-run-key-part-23/

没有manifest就会启用.local，这样的程序不算多

设置注册表可以全局启用

当前目录下的exe名字+.local文件夹会优先成为dll搜索路径

dccw.exe屏幕校准和consent.exe，GoogleUpdate.exe没有menifest会找.local





### SetProcessMitigationPolicy

[API文档](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setprocessmitigationpolicy) 

ProcessSignaturePolicy 防止加载非微软签名的dll。相关的论文不少。

怎样让系统exe能启动的时候调用？内核态驱动层设置？





### 已经收集的攻击方法分类



| 类型       | 未修复的攻击        | 已修复      |
| ---------- | ------------------- | ----------- |
| 注册表劫持 | #33 #62 #53 #56 #61 | #29 #25 #31 |
| DLL劫持    | #22 #23 #30 #37 #39 | #18 #26     |
| COM组件    | #38 #59 #41 #43     |             |
| UIAccess   | #32 #55             |             |
| 其他       | #36 #35 #52 #63     |             |
| 环境变量   | #34 #58             |             |



其中也有UAC提醒等级为Always Notify下可以绕过的方法: #26 #34



### Windows 中的动词机制

https://docs.microsoft.com/en-us/windows/win32/shell/fa-verbs

**eg : Print**, **Edit** and **Open with**. 这个动词对应的恰好是ShellExecute的lpOperation参数。系统拿着动词去注册表找。

HKEY_CLASSES_ROOT 是*HKEY_LOCAL_MACHINE\Software\Classes*的链接。内保存了各种不同类型的文件拓展对应的打开方式。如HKCR\\.py\\@ 保存的是Python.File。而HKCR\\Python.File\Shell\Open\Command 就保存了打开python文件的命令行。

但是HKEY_LOCAL_MACHINE是系统的总体打开方式。每个用户也有自己的HKEY_Current_User\Software\Classes可以设置自己的打开方式。

用户自己设置的打开方式理论上是覆盖系统的打开方式，就像局部变量的名字覆盖全局变量一样。

一般HKCU里面查不到是正常的，在HKLM里有，但是会去HKCU里面找。



## #25 EventVwr.exe的注册表劫持 - 失效

发现者：enigma0x3，时间：20160815。https://enigma0x3.net/2016/08/15/fileless-uac-bypass-using-eventvwr-exe-and-registry-hijacking/ 

sdclt.exe /KickOffElev 

**HKCR\mscfile\shell\open\command** 

EventVwr.exe redesigned, CompMgmtLauncher.exe autoelevation removed

https://github.com/turbo/zero2hero

- Works from: Windows 7 (7600)
- Fixed in: Windows 10 RS2 (15031)
  - How: EventVwr.exe redesigned, CompMgmtLauncher.exe autoelevation removed



## #29 sdclt.exe AppPathMethod - fixed

发现者：enigma0x3，时间：20170314。链接：https://enigma0x3.net/2017/03/14/bypassing-uac-using-app-paths/ 

sdclt.exe（打开跳到控制面板的backup）通过查看低权限的注册表**HKCU:\Software\Microsoft\Windows\CurrentVersion\App Paths\control.exe** 启动控制面板。

- Works from: Windows 10 TH1 (10240)
- Fixed in: Windows 10 RS3 (16215)
  - How: Shell API update



## #31 sdclt.exe IsolatedCommandMethod - fixed

发现者：enigma0x3，时间：20170317。https://enigma0x3.net/2017/03/17/fileless-uac-bypass-using-sdclt-exe/

sdclt.exe /KickOffElev

- Works from: Windows 10 TH1 (10240)
- Fixed in: Windows 10 RS4 (17025)
  - How: Shell API / Windows components update



## #33（#62） 注册表劫持

有点像infamous Enigma0x3 "***mscfile fileless***" bypass，但利用的是不同的注册表和不同的程序。widely used ITW by malware。

https://cqureacademy.com/cqure-labs/cqlabs-how-uac-bypass-methods-really-work-by-adrian-denkiewicz

fodhelper.exe computerdefaults.exe 加载注册表中的 exe

```
New-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
New-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Value "" -Force
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "(default)" -Value cmd.exe -Force
Start-Process "C:\Windows\System32\fodhelper.exe"
Remove-Item "HKCU:\Software\Classes\ms-settings\" -Recurse -Force
```



https://devblogs.microsoft.com/oldnewthing/?p=14623

其中#62就只是computerdefaults.exe触发

WD的检测与绕过：

- 符号链接写入



## #53 注册表劫持



Target key here is ***HKCU***\\***Software\Classes\Folder\shell\open\command\\*** @*Default* value (+*DeletegateExecute* as usual) and trigger is *sdclt.exe*。

WD会检测。修改那个@default就会触发。

绕过方法同上。

## #56 注册表劫持

Target key here is ***HKCU\Software\Classes\AppX82a6gwre4fdg3bt635tn5ctqjf8msdd2\shell\open\command*** @*Default* value (+*DeletegateExecute* as usual) and trigger is *wsreset.exe*

WD会检测。修改那个@default就会触发。

绕过方法：准备好注册表相关的结构，再重命名过去。

## #61 注册表劫持

Target key here is ***HKCU\Software\Classes\******Launcher.SystemSettings\shell\open\command*** @*Default* value (+*DeletegateExecute* as usual) and trigger is *slui.exe* which is started with ***runas\*** verb*.*

WD会检测。修改那个@default就会触发。





## #58 COM组件的环境变量劫持 AlwaysNofity ok

**EditionUpgradeManager** 这个提权的接口里的*AcquireModernLicenseWithPreviousId* 函数会通过%windir%去组合路径调用*Clipup.exe*。设置windir环境变量去劫持。

Cheap and easy fix - get rid of environment variables from Win95 era and build a proper path to *clipup.exe* by using, surprise - surprise! ***GetSystemDirectory***.



## #34 DiskCleanup计划任务 - unfixed - AlwaysNofity ok

20170515 [James Forshaw](https://twitter.com/tiraniddo)  https://www.tiraniddo.dev/2017/05/exploiting-environment-variables-in.html

https://cqureacademy.com/cqure-labs/cqlabs-how-uac-bypass-methods-really-work-by-adrian-denkiewicz

*Microsoft\Windows\DiskCleanup* 会启动 %windir%\system32\cleanmgr.exe 

```
New-ItemProperty "HKCU:\Environment" -Name "windir" -Value "cmd.exe /k cmd.exe" -PropertyType String -Force
schtasks.exe /Run /TN \Microsoft\Windows\DiskCleanup\SilentCleanup /I
```

WD的检测：*schtaks.exe* 的命令行是否包含“Microsoft\Windows\DiskCleanup\SilentCleanup”，有则警报 （**Behavior:Win32/SilentCleanupUACBypass**）

检查**\Microsoft\Windows** 子串（***PossibleSchedTasksUACBypass*** ）

绕过：不用schtaks，用***ITaskService***, ***ITaskFolder*** and ***IRegisteredTask*** 接口自己去启动

真正应该：检测windir 环境变量改变。

windows中的环境变量直接反映在注册表里。





## Method #18 -fixed



参考资料：

https://www.reddit.com/r/lowlevel/comments/4lktyw/sxs_backdoor_and_story_of_windows_uac_ridiculous/

https://www.kernelmode.info/forum/viewtopicb857.html?t=3643&start=90#p28579

背景，自从SxS+sysprep.exe的劫持，已经烂大街，微软拿出了自己的后门进行修复：

**malware** friendly的微软对应用的垃圾Manifest文件的undocumented 项

`<file loadFrom="%systemroot%\system32\NAME.dll" name="NAME.dll" />`

只要找到没有manifest的文件，通过外置manifest文件的方法，`ExeName.exe.manifest` 通过cryptbase.dll依然在sysprep文件夹内做dll劫持。

`<file loadFrom="%systemroot%\system32\sysprep\cryptbase.DLL" name="cryptbase.DLL"      />`

当前该方法已经失效，因为外置manifest文件不再支持这个项了。



## #22

*comctrl32.dll* 植入到system32 （利用.local?） 劫持*consent.exe* 。

WD只检测了子文件夹创建，和文件移动到对应的文件夹内（移动？）。不移动，直接写入文件夹可以绕过？

首先通过***IFileOperation*** with **ISecurityEditor** 创建文件夹并降权？

创建好对应名字的文件再降权再写入。

创建好其他名字的文件后再降权？再重命名，写入



微软应该这样修复： process mitigation policy  的 **ProcessSignaturePolicy** 设置为只有微软签名的

ProcessSignaturePolicy restrict image loading to those images that are either signed by Microsoft, by the Windows Store, or by Microsoft, the Windows Store and the Windows Hardware Quality Labs



## #23 PkgMgr-Dism劫持

https://cqureacademy.com/cqure-labs/cqlabs-how-uac-bypass-methods-really-work-by-adrian-denkiewicz

+博客https://swapcontext.blogspot.com/

*PkgMgr* 被放弃支持了，但还在自动提权的exe列表（g_lpAutoApproveEXEList）里，是dism的启动器。

dism搜索***dismcore.dll*** 的时候会先搜索system32文件夹，再搜索当前路径，然后找到。

1. 创建*oemsetup.xml*文件，用来调用pkgmgr程序

   ```
   <?xml version="1.0" encoding="utf-8"?>
   <unattend xmlns="urn:schemas-microsoft-com:unattend">
   <servicing>
   <package action="install">
   <assemblyIdentity name="Package_1_for_KB929761" version="6.0.1.1" language="neutral" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35"/>
   <source location="%configsetroot%\Windows6.0-KB929761-x86.CAB" />
   </package>
   </servicing>
   </unattend>
   ```

2. IFileOperation 移动DismCore.dll到*C:\windows\system32*. 

3. `"C:\WINDOWS\system32\pkgmgr.exe" /n:C:\Users\<user>\AppData\Local\Temp\oemsetup.xml`

   随后pkgmgr调用`"C:\WINDOWS\system32\dism.exe" /online /norestart /apply-unattend:"C:\Users\<user>\AppData\Local\Temp\oemsetup.xml"` 

WD叫***Win32/Disemer***，首先会检测system32下的 DismCore.dll（平常系统不可能发生），其次会检测*pkgmgr.exe* 的参数里特定的Token

绕过方法：通过其他的dll去劫持。pkgmgr的参数（xml文件）也不用加什么？？因为没有意义。





## #30 Wow64 subsystem logger dll

subsystem logger dll是无文档内部的debug工具，十年前[lhc645](https://lhc645.wordpress.com/tag/wow64log-dll/)就发现了。wow64log.dll 源码骨架都有。

每个wow64 application（32位）启动的时候都会加载它（64位），

This method has a lot of disadvantages and not that much room to maneuver in case of active counteraction from WD.

UACMe首先移植非native的wow64log.dll到system32，之后相关程序启动之后就会崩溃，然后启动*WerFault* 实例，此时wow64log.dll才执行payload。

最好能默认关闭这个debug功能。







## #26 dismhost.exe劫持 - fixed

发现者：enigma0x3，时间：20160722。https://enigma0x3.net/2016/07/22/bypassing-uac-on-windows-10-using-disk-cleanup/

cleanmgr.exe 系统的计划任务中的磁盘清理程序，是提权的计划任务。它运行的时候会把dismhost.exe和一些其他dll拷到%temp%\GUID\dismhost.exe再运行。通过复制LogProvider.dll劫持。需要一直监控着temp文件夹然后在他们运行之前拷好。有点算条件竞争吧。

- Works from: Windows 10 TH1 (10240)
- AlwaysNotify compatible

- Fixed in: Windows 10 RS2 (15031)
  - How: File security permissions altered



## #36

infamous *SandboxEscaper* 利用Wusa在当前盘符根目录下创建的临时目录的安全性不对，利用NTFS reparse point feature。条件竞争设置reparse point到那个目录上就可以重定向写入到其他地方。

## #37 

使用#35去高权限移动文件。

使用GdiPlus with target set to *DCCW.exe* 进行dotLocal的dll劫持。

dccw.exe.local的防御

## #38 没看懂

This method uses *appinfo* command line parsing spoofing. This requires talking to *appinfo* via *RPC* as Windows Shell always build correct command line request when used *ShellExecute(Ex)*. This can be achieved through ***AicLaunchAdminProcess*** hook or by direct *RPC* call. You have to prepare your *MMC* snap-in and embed in it *Shockwave Flash Object* which will execute supplied *HTML* script located somewhere on disk. This *HTML* code will eventually run your payload through *ExecuteShellCommand* method.

## #59 

*appinfo* direct talking 的debug object去提权，非常复杂。和#38利用的原理不同



## #39 C# profile - unfixed

https://www.freebuf.com/vuls/183914.html

https://offsec.almond.consulting/UAC-bypass-dotnet.html

https://3gstudent.github.io/3gstudent.github.io/Use-CLR-to-bypass-UAC/

双击任一msc文件，通过Procmon监控发现最终运行的都是mmc.exe文件

CLR(Common Language Runtime)，是微软为他们的.NET的虚拟机所选用的名称

CLR会先检查环境变量中COR_ENABLE_PROFILING是否为1，若检查通过，则根据.NET版本不同，查找DLL位置的方法也不同，对于低于4.0的则去注册表中查找CLSID或ProgID项，找到其指定的dll文件路径。从.NET4.0版本开始则先查找环境变量COR_PROFILER_PATH是否指定dll文件路径，没有再去注册表中查找对应的CLSID项。所以这里我们就不设置COR_PROFILER_PATH了

```
COR_ENABLE_PROFILING = 1

COR_PROFILER={CLSIDor ProgID}
```

找到HKEY_CURRENT_USER\Software\Classes\CLSID项，分别添加以下新项：

```
{11111111-1111-1111-1111-111111111111}和InprocServer32
```

该dll中只负责运行cmd.exe，并退出主进程：现在我们尝试运行gpedit.msc，看能否运行管理员权限的cmd程序：

WD：无法检测

检测方法：检测环境变量。

真正的修复：mmc.exe在高权限允许时应该不允许加载非微软签名的程序。

## #41 ICMLuaUtil

应该直接从COMAutoApprovalList移除。

## #43 IColorDataProxy

先用#41的任意注册表写，写入**HKLM\Software\Microsoft\Windows NT\CurrentVersion\ICM\Calibration** @*DisplayCalibrator* 然后调用 **IColorDataProxy** 通过高完整度**ShellExecuteEx** 运行指定的程序。

## #52 appinfo里利用GetLongPathName缺陷

恶意软件非常喜欢的方法。简单。Win32 API path conversion

特殊方法创建一个后面带空格的"C:\Windows "目录，然后在appinfo.dll里面UAC检查的时候经过***GetLongPathName\*** 这个API会自作主张修复这个不合法的路径，导致之后比对的时候成为可信路径。

然后从system32里面拷可信exe出来做dll劫持。

UACME的作者改进了，增加了reparse points。



## #32 osk的劫持+UIAccess消息注入

复制osk.exe到`%ProgramFiles%\Windows Media Player` 文件夹，然后通过该文件夹下的*duser.dll* or *osksupport.dll* 进行劫持，最终得到Medium+的带UIAccess的权限。可以通过MessageHook注入高权限进程。UacMe注入了*eventvwr.exe* 。

微软应该：

1. 加固Manifest的LoadFrom项
2. 禁止*UIAccess* 权限的进程从Medium+注入到High



## Method #55

有意思的攻击，虽然是自动化的，但是涉及到了UI的操作，会被用户看到。

> 参考资料：
>
> https://github.com/rootm0s/UUB
>
> https://swapcontext.blogspot.com/ 
>
> https://cqureacademy.com/cqure-labs/cqlabs-how-uac-bypass-methods-really-work-by-adrian-denkiewicz

获取UIAccess权限 -> 利用它提权。

#### 阶段一 UIPI token窃取

从msra.exe/osk.exe(on-screen keyboard) 窃取有UIAccess标志的令牌，然后修改完整度从Medium+到Medium，再启动自己的程序，这样就能操作高完整度的UI窗口了。

#### 阶段二 GUI操作

启动msconfig打开cmd，cmd输入命令任意执行。



## #63 

相对较新的方法。dll injection to the Windows *Native Image Cache (NIC)*



## Always Notify下可以绕过的方法



## 结语

While maybe not that sophisticated and complicated UAC is still part of Windows and integrated to the system. It is not about OpenDialog. The better you understand Windows internals the better you understand how UAC works and it weakness (and the more you laugh at yet another idiot with OpenDialog). Aside of this some of these bypasses are usually accompaned by various different mechanics, algorithms and internals that need to be researched. 

There is a still a lot of possible real UAC bypasses undiscovered since this system mechanics is entirely broken in many ways and these methods are just waiting to be exploited.



