---
title: COM与RPC
date: 2020/6/1 11:11:11
categories:
- Dev
tags:
- Windows
---


# COM与RPC

研究windows安全的过程中对COM非常不熟。于是找资料学习了一会。

<!-- more -->

## 概述

COM，组件对象模型，它解决的问题是二进制间兼容性问题，并在此基础上实现了RPC。

主要学习资料： [COM编程攻略](https://zhuanlan.zhihu.com/c_1234485736897552384)



它的兼容思想是通过只暴露接口，不得出现跨边界的编译器相关行为，从而实现二进制的兼容。即不依赖结构体的布局，不依赖类型转换和new、delete的实现。

AddRef()：返回之后的引用计数。

Release()：一旦引用计数为0，实现者必须要释放此对象。

QueryInterface()

## 接口转换的实现原则

`HRESULT QueryInterface(REFIID iid, void** ppvObject);` 

1、如果可以成功拿出接口，返回S_OK。如果ppvObject为空，返回E_POINTER。如果不能拿出接口，那么返回E_NOINTERFACE。

2、QueryInterface(下面简称QI)是静态的，不是动态的。这说明，一个对象QI能否成功，和时间没有关系。如果某个特定的类的实例QI(A)->B（执行QueryInterface拿到B），那么任何时候都应该能拿到B。

3、QI是自反的（如果QI(A)->B，那么QI(B)->A。

4、QI是对称的。

5、QI是可传递的。

6、如果需要取的是IUnknown(IID_IUnknown)，那么必须要返回相同的指针。

## IUnknown 继承模型 聚合模型

继承模型：一个接口继承IUnknown，要用的时候转换成自己。

聚合模型：实现IUnknown的是套壳接口，QI的时候返回不同的接口。

结合聚合模型的特点和接口转换的实现原则，进行推理：用不同的地址代表不同接口的具体实现：

1. 根据自反性，必须能够一次任意转换。因此所有的聚合在同一个套壳接口的类型调用QI的时候必须调用套壳接口的QI。
2. 根据



## ATL实现的三层模型

`Wrapper -> YourClass -> Internal` 

![COM](COM与RPC/COM.jpg)

`CComObject` 对应继承模型，如下

```cpp
template <class Base>
class CComObject : 
	public Base
```

`CComAggObject` 则对应的是聚合模型，不再直接继承。

YourClass需要继承internal和各种需要的interface，并用宏指明转换规则。从而创建`_QueryInterface` 函数和静态与 `_GetEntries` 的Entries。由 `InternalQueryInterface` 来调用API遍历这个表。YourClass不只是一个分发器，而是把接口的实现都作为自己的成员函数。

Interface是带有很多虚函数的基类罢了。虚函数是父类声明时，用来告知编译器，希望即使把子类作为父类，调用同名方法的时候要调用子类的方法。

1. 调用QI（wrapper的）会调用到内部的YourClass分发器的QI。
2. 成功分发，转换类型后，再调用QI得调用回Wrapper的QI。

`CComObjectRootBase` 类型自身就有m_pOuterUnknown成员，和`OuterAddRef` 、 `OuterRelease` 函数，用来对聚合模型实现支持。它的QI就是总的QI，之后转换出去的COM接口都要调用回来这里的QI。

其实是COM手动实现了对与Interface类型的转换？YourClass注册Interface的时候，通过一个方法的静态数组成员来记录每个IID和对应的指针相对于this的偏移，转换的时候用到。但实际上，外围的CComAggObject持有的是通过模板生成的CComContainedObject。它通过模板继承上面写的类，重写了QueryInterface。通过CComAggObject拿到的都是继承自己的类之后的CComContainedObject了，此时构建的时候传入了原来的IUnokown的指针，通过继承和重新实现QI，把QI导向到了总的QI。导向方法是转调OuterQueryInterface，它调用了CComObjectRootBase的m_pOuterUnknown->QueryInterface

由于拿到的总是被`CComContainedObject ` 包围的QI，这里是调用`CComContainedObject ` 的QI。此时则调用的之前保存的OuterUnknown的QI。

QI，的时候，是把

## Example

1. 首先vs2019选ATL项目模板，创建ATLMessageBox项目，选择服务exe

   此时的解决方案里面有ATLMessgaeBox和ATLMessageBoxPS项目，后者是ProxyStub代理桩，给享受服务的客户端用的，客户端调用对应服务的时候由它来处理序列化，通讯等事情。

2. **uuidgen /i /ohello.idl** 创建带有UUID的IDL文件
3. IDL 文件描述接口，填写创建
4. rgs注册表消息，填写创建

5. 创建MessageBox.cpp MessageBox.h

6. 注册：C:\Users\warren\source\repos\ATLMessageBox\x64\Debug\ATLMessageBox.exe /RegServer



## RPC

[Remote Procedure Call](https://docs.microsoft.com/en-us/windows/win32/rpc/rpc-start-page) 

主要分析的是 [这个微软的例子RPCHello](https://github.com/microsoft/Windows-classic-samples/blob/master/Samples/Win7Samples/netds/rpc/hello/Hellop.c) 

TODO rpc的跨平台，是否支持linux或者Unix / Apple

RPC的环境内置在windows中，而RPC的开发环境在windows sdk中。

Microsoft Interface Definition Language MIDL，用来描述调用的接口。

客户端程序调用的服务端的函数，实际上不是真正的实现函数，而是一个stub函数，负责把参数转换成标准的NDR格式，通过网络传输请求。

服务器的运行时函数接受请求，转换参数，最后再调用服务端的stub函数，返回值数据的时候也是类似的方法传输回去。

RPC有如下组件：MIDL编译器，运行时的lib和头文件，Name service provider和Endpoint mapper。还有uuidgen工具。

承载RPC的dll有通过命名管道的、tcp/ip、NetBIOS、SPX、IPX、UDP的等等。

开发的过程包括：开发接口->开发服务端->开发客户端。

接口的定义主要包括的是IDL文件和ACF文件。编写后用MIDL编译器得到服务端和客户端的stub。VS1029中idl文件属于源文件，而acf文件属于资源文件。编译时的选项在项目的属性中多出来的MIDL项里面配置。

- Hello_c.c 客户端stub

- Hello.h 两边都包括的头文件
- Hello_s.c 服务端的stub

Hellop.c 这个文件不是生成的，（example里面的）包含对server的procedure的实现。

Hellos.c和Helloc.c里面就是真正的RPC代码了。这一块才是重点关注的部分。

### MIDL



服务端和客户端代码容易混在一起，在同一个项目里建立两个文件夹。

默认情况下，客户端和服务端的stub函数名字相同，导致不能同时链接服务端和客户端的stub，编译的时候加上 `/prefix` 参数可以避免这种情况。

如果编译的时候不加上 [`/osf`](https://docs.microsoft.com/en-us/windows/win32/midl/-osf) (Open Software Foundation compatibility mode)，就需要提供一个函数分配和回收内存。开启这个模式会失去很多功能特性。





### server

API调用序列大致如下

RpcServerUseProtseqEp

RpcServerRegisterAuthInfo (增加安全机制)

RpcServerRegisterIfEx

RpcServerListen

RpcMgmtWaitServerListen 循环等待

RpcMgmtStopServerListening

RpcServerUnregisterIf



applications must specify a string that represents a combination of 

1. an RPC protocol, 

   1. Network Computing Architecture connection-oriented protocol (NCACN)
   2. Network Computing Architecture datagram protocol (NCADG)
   3. Network Computing Architecture local remote procedure call (NCALRPC) 

   一般都选这个NCALRPC ？

2. a transport protocol and a network protocol. TCP/IP. IPX/SPX, NetBIOS, AppleTalk DSP什么的。肯定选tcp/ip 

**ncalrpc** for local communications and **ncacn_ip_tcp** or **ncacn_http** for remote communications are recommended

选好了就可以通过the [**RpcStringBindingCompose**](https://docs.microsoft.com/en-us/windows/desktop/api/Rpcdce/nf-rpcdce-rpcstringbindingcompose) and [**RpcBindingFromStringBinding**](https://docs.microsoft.com/en-us/windows/desktop/api/Rpcdce/nf-rpcdce-rpcbindingfromstringbinding) functions创建binding的handle了。



另外需要实现 the [**midl_user_allocate**](https://docs.microsoft.com/en-us/windows/desktop/Rpc/the-midl-user-allocate-function) and [**midl_user_free**](https://docs.microsoft.com/en-us/windows/desktop/Rpc/the-midl-user-free-function) 这两个函数。

```c
void __RPC_FAR * __RPC_USER midl_user_allocate(size_t len)
{
    return(malloc(len));
}

void __RPC_USER midl_user_free(void __RPC_FAR * ptr)
{
    free(ptr);
}
```





### client

源文件添加上生成的 _c.c后缀的文件。此外要加上任何可能需要的lib文件

API调用序列如下

RpcStringBindingCompose

RpcBindingFromStringBinding

RpcBindingSetAuthInfoEx (增加安全机制)



HelloProc

RpcStringFree



RpcBindingFree



#### spn

*Service Principal Name* is a concept from Kerberos

实现安全机制的时候用的，所以目前可以暂时不管。