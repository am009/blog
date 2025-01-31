---
title: Ghidra反编译器原理与代码架构解析
date: 2025/01/31 11:11:12
categories:
- Dev
tags:
- Decompile
- CTF
---

Ghidra反编译器原理与架构解析

Ghidra Sleigh Decompiler Internals and Code Architectures

<!-- more -->

**相关资源**

{% pdf /2023.assets/D1T2_Taking_Ghidra_to_The_Next_Level_Zhanzhao_Ding.pdf %}

- [Ghidra反编译器的C++源码](https://github.com/NationalSecurityAgency/ghidra/blob/5604178194cb55ddb8526f843466d485ed8a36e6/Ghidra/Features/Decompiler/src/decompile/cpp/coreaction.cc#L5363) 在这层目录执行make doc会使用doxygen生成HTML格式的[文档](https://lemuellew.github.io/Ghidra-Decompiler-Analysis-Engine-Document/)。
- [一个Ghidra的使用介绍](https://github.com/dannyquist/re/blob/master/ghidra/ghidra-getting-started.md)
- [一个Ghidra PCode转LLVM的论文](https://uwspace.uwaterloo.ca/bitstream/handle/10012/17976/Toor_Tejvinder.pdf?sequence=3&isAllowed=y)和[代码](https://github.com/toor-de-force/Ghidra-to-LLVM)
- [Ghidra上的ASI结构体类型恢复](https://blog.grimm-co.com/2020/11/automated-struct-identification-with.html)

### Ghidra不是Java写的吗？

![Ghidra的C++和Java代码关系](Ghidra的C++和Java代码关系.png)

是的，但是反编译器是用C++写的。Java主要负责图形界面，反汇编器等外围数据结构，比如存储各种图形界面展示的信息。核心的反编译算法是用C++写的，并又Java进程启动子进程，中间似乎是通过一种特殊的XML格式通信（TODO）。

但是这个反编译器也自带一个简单的命令行，用于调试。Ghidra图形界面反编译结果右上角也可以生成一个xml格式的dump，包含了输入给C++反编译器进程的所有信息，用它可以复现某次反编译过程，便于调试。

### Ghidra 反编译器简介

![Ghidra反编译器内部模块](Ghidra反编译器内部模块.png)

反编译器内部，Protocol负责和Java进程通信，比如获取用户设置的变量名等。最关键的是Action，和LLVM的Pass类似。

![反编译器内部流程](反编译器内部流程.png)

这里参照[文档](https://lemuellew.github.io/Ghidra-Decompiler-Analysis-Engine-Document/)阅读，现有的反编译器都需要结合编译优化技术，因此往往也基于SSA的IR。然后就是一些特有的，比如逆向处理一些适合执行但是不适合分析的优化。最后是转表达式，然后CFG转if-else/while/for等AST控制流的结构恢复。

### 阅读与调试

**代码阅读环境**：尝试编译，并用bear等工具生成compile_commands.json。然后用VSCode打开有json的那层文件夹，然后使用clangd插件或者默认的C++插件，从而获得代码提示和跳转。

**代码调试环境**：使用`make decomp_dbg`编译后，找到`cpp/decomp_dbg`可执行文件进行调试。[这里](https://www.nccgroup.com/sg/research-blog/earlyremoval-in-the-conservatory-with-the-wrench-exploring-ghidra-s-decompiler-internals-to-make-automatic-p-code-analysis-scripts/)也有相关的介绍

```
{
    "type": "lldb",
    "request": "launch",
    "name": "Debug",
    "program": "${workspaceFolder}/cpp/decomp_dbg",
    "args": [],
    "cwd": "${workspaceFolder}",
    "env": {"SLEIGHHOME": "/home/ubuntu/ghidra"}
}
```

### 代码结构

最关键的分析在[`universalAction`](https://github.com/NationalSecurityAgency/ghidra/blob/5604178194cb55ddb8526f843466d485ed8a36e6/Ghidra/Features/Decompiler/src/decompile/cpp/coreaction.cc#L5363)函数注册。上面的`buildDefaultGroups`给分析分组。

在启动的时候，会调用`ActionDatabase::resetDefaults`

### SLEIGH 汇编转PCode IR

TODO

### 中端优化与类型恢复

TODO

### 打印到语法树

数据结构上，Ghidra没有使用完善的C语法树节点，而是直接基于结构恢复算法的数据结构，以及PCode。直接在结构分析结束的时候打印为C代码。

结构分析是一个CFG转IF-ELSE/WHILE这种语法树形式的控制流的过程。使用的数据结构在最初的时候就是CFG，每个基本块里面的指令可以提前转为C代码，也可以之后转。在结构恢复过程中，会使用模式匹配，匹配IF-ELSE结构或者While循环，然后把相关的块折叠为一个线性执行的复合块（[`FlowBlock`](https://github.com/NationalSecurityAgency/ghidra/blob/784540f1c0fc7df122f1c6591557edf608a8c1d1/Ghidra/Features/Decompiler/src/decompile/cpp/block.hh#L77)），或者说直接折叠为一个Statement，包含在其他基本块中。在分析结束的时候，整个函数就只有一个函数体的单个基本块。

这里Ghidra在结构分析结束的时候直接基于这个复合块打印。Ghidra对每种特殊的块都实现同一个emit虚函数，然后[转调](https://github.com/NationalSecurityAgency/ghidra/blob/784540f1c0fc7df122f1c6591557edf608a8c1d1/Ghidra/Features/Decompiler/src/decompile/cpp/block.hh#L699)`printc.cc`里面`PrintC`的方法。

具体每个语句，即内部的表达式是[直接基于PCode打印](https://github.com/NationalSecurityAgency/ghidra/blob/e5969a613cdca59252cc0366254d005597c313a2/Ghidra/Features/Decompiler/src/decompile/cpp/printc.cc#L2716)的。具体是[`PrintC::emitExpression`](https://github.com/NationalSecurityAgency/ghidra/blob/e5969a613cdca59252cc0366254d005597c313a2/Ghidra/Features/Decompiler/src/decompile/cpp/printc.cc#L2464)函数。这里有一个`RPN stack`逆波兰表示法。

其他反编译器是怎么做的？[reko的做法](https://github.com/uxmal/reko/blob/f9ff640ea95cbde1f2f4ce948339d42e392d5dab/src/Decompiler/Decompiler.cs#L315)类似

### decomp_dbg

[这里](https://www.nccgroup.com/sg/research-blog/earlyremoval-in-the-conservatory-with-the-wrench-exploring-ghidra-s-decompiler-internals-to-make-automatic-p-code-analysis-scripts/)的执行实例，可以总结出大致的流程：

- 首先使用`SLEIGHHOME=/opt/ghidra_10.1.2 ./decomp_dbg`启动
- 使用`restore /tmp/mystery_printf_main.xml`命令加载某次反编译信息。
  - 这里也可以用binary命令直接加载某个二进制？。
- 然后执行`load function main.main`
- `trace address 0x48e7c4` 追踪某个指令被每个优化怎么修改了。
- `decompile` 启动反编译


decomp_dbg命令行处按Tab有如下的命令：
- `# % //` 表示注释
- addpath
- adjust vma
- analyze range
- binary
- break action
- break jumptable
- break start
- callgraph build
- callgraph build quick
- callgraph dump
- callgraph list
- callgraph load
- clear architecture
- closefile
- codedata dump crossrefs
- codedata dump hits
- codedata dump starts
- codedata dump targethits
- codedata dump unlinked
- codedata init
- codedata run
- codedata target
- comment instruction
- continue
- count pcode
- deadcode delay
- debug action
- decompile
- disassemble
- dump
- duplicate hash
- echo
- execute test command
- fixup apply
- fixup call
- fixup callother
- force datatype
- force goto
- force varnode
- global add
- global registers
- global remove
- global spaces
- graph controlflow
- graph dataflow
- graph dom
- history
- isolate
- list action
- list override
- list prototypes
- list test commands
- load addr
- load file
- load function
- load test file
- map address
- map convert
- map externalref
- map function
- map hash
- map label
- map unionfacet
- name varnode
- openfile append
- openfile write
- option
- override flow
- override jumptable
- override prototype
- parse file
- parse line
- pointer setting
- prefersplit
- print C
- print C flat
- print C globals
- print C types
- print C xml
- print actionstats
- print cover high
- print cover varnode
- print cover varnodehigh
- print extrapop
- print high
- print inputs
- print inputs all
- print language
- print localrange
- print map
- print parammeasures
- print raw
- print spaces
- print tree block
- print tree varnode
- print varnode
- produce C
- produce prototypes
- prototype lock
- prototype unlock
- quit
- read symbols
- readonly
- remove
- rename
- reset actionstats
- restore：这个命令后面加导出的XML文件，就可以复现一次反编译过程了
- retype
- save
- set context
- set track
- source
- structure blocks
- trace address
- trace break
- trace clear
- trace disable
- trace enable
- trace list
- type varnode
- volatile
