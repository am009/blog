---
title: SVF-LLVM指针分析框架
date: 2025/09/21 11:11:12
categories:
- Dev
tags:
- StaticAnalysis
---

本文解析SVF的代码框架。

<!-- more -->

## SVF

SVF是一个C++指针分析框架。

## Whole Program Analysis

- 构建了LLVMModuleSet，包含了一系列Module，以及函数集合等。
- [构建SVFIR](https://github.com/SVF-tools/SVF/blob/45597fc4c7fe44ff798db73dde5e7097cb3af444/svf-llvm/lib/SVFIRBuilder.cpp#L54)：
    - 处理外部函数节点，全局变量节点
    - 转换类型：LLVM的类型被转换为了SVFType。
    - 构建SVFIRCallGraph
    - [处理不同的指令](https://github.com/SVF-tools/SVF/blob/45597fc4c7fe44ff798db73dde5e7097cb3af444/svf-llvm/lib/SVFIRBuilder.cpp#L1004)
