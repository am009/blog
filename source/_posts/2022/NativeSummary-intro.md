---
title: NativeSummary简介
date: 2022/9/15 11:11:11
categories:
- Project
tags:
- StaticAnalysis
- Android
---

本文是对NativeSummary的介绍。配有配套slides。

<!-- more -->


直觉是，很多JNI接口很多对应着Java语句的操作。支持的JNI调用：

```
Throw,  ThrowNew
NewObject、Call[Nonvirtual/Static]XXXMethod
Get/Set[Static]ObjectField
不太好处理，先不管的：
NewXXXArray、Get/ReleaseXXXArrayElements
ExceptionCheck
```

当拿到一个APK文件，解包得到dex文件-Java代码，`.so`文件代表C语言编译后的二进制代码。

1. JNI绑定部分：找到分析入口

    - 最简单的模式，静态块内的System.LoadLibrary，so侧通过导出函数直接绑定。我们则扫描so侧的导出函数作为入口。这些都发生在大部分代码之前。

    - 动态绑定与JNI_OnLoad：System.LoadLibrary -> JNI_OnLoad(.so库内的C语言函数) -> DynamicRegister函数。也就是说依赖于对函数的分析，如果没有分析成功，直接无法找到函数入口点。但是一般的应用这个函数都是一个比较简单的编程模式的。

    - Java动态选择二进制库：Java类在静态块内往往调用System.loadLibrary，去加载不同的库。例如，有的APK会有两个名字类似，功能重复但稍有差别的二进制库，如`libxxx.so`和`libxxx_ssl.so`。这个有点麻烦，目前直接假设都会加载。

    还有各种其他小问题：

    - 分析的函数调用了其他so库内的函数：需要给分析器加上动态链接功能，加载并解析好。

2. 分析部分

    - 现在分析的现状：VSA的开源实现效果都不佳。介绍一下心路历程。

    - 介绍BAI的原理。用一个实例。讲得复杂点。假设一个函数，被不同的context调用。然后有混合的情况。

    - 介绍如何从AbsEnv里面提取对应到Java语句。同时介绍IR定义。

    - 介绍如何才能

3. 更好地集成：
    - 架构设计：（其实之所以分开是因为上一个是Python的Angr和Soot的Java语言不一样。但是现在都是Java，但由于是插件的架构，虽然合并还是有点麻烦，但是理论上是可以的。而且可以更多地合作交互。
    - 集成FlowDroid其实也行，但是不能把Flowdroid作为一个maven依赖使用。。。
    - 最后看来重打包还是挺好的，而且有优点。

### callee saved register

打一个形象的比方：假如有两个人（函数）洗衣服和修洗衣机，洗衣服函数会先把衣服放在旁边，然后操作洗衣机。首先判断是否洗衣机是坏的，是则要调用修洗衣机的函数。但是修洗衣机的那个人（函数）需要很大的位置（寄存器），所以需要把衣服挪开，修洗衣机，然后再把衣服放回去。有人可能问，为什么不让洗衣服函数在调用函数前把位置都空出来呢？因为有可能调用的函数是不太需要位置的，比如买洗衣粉函数。所以一般都会有部分位置作为随便用的，部分位置规定是要恢复的。比如这里采取了折中的方案，位置1是随便用的，位置2是要给原来的人保留的。

```
约定：位置1是随便用的(caller saved)，位置2的要给原来的人放回去（callee saved）

func 洗衣服
  衣服放到位置1，衣服放到位置2
  if 洗衣机坏了
    把位置1的东西挪到自己的仓库位置1
    修洗衣机()
    把自己的仓库位置1的东西放回位置1
  if 没有洗衣服
    买洗衣粉()

func 修洗衣机 (需要两个位置)
  把位置2的东西挪到自己的仓库位置1
  在位置1，位置2修洗衣机
  把自己的仓库位置1的东西放回位置2

func 买洗衣服 (不需要位置)
  买来洗衣粉
  返回
```
