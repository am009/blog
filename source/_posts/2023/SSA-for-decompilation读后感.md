---
title: SSA-for-decompilation读后感
date: 2023/5/12 11:11:12
categories:
- Read
tags:
- Decompile
---

SSA-for-decompilation 读后感。反编译这一块不是很成体系。在这里整理一下。

<!-- more -->

### 总体

1. 静态分析和人分析还是不一样的。部分增加可读性的操作可以忽略。
1. 分类法：按类型信息的来源-传递（运算）。
1. 从另一头看，需要学习和观察静态分析的问题。静态分析如何处理引用。

### 结构体分析

> 聚合类型通常只能从基本指令的上下文中发现，例如在循环中执行的基本操作，或在指针的固定偏移处执行的一些基本操作。

聚合类型，比如结构体和数组，确实，似乎只能从循环中识别出来。而且这相关的似乎完全没有被retypd提到。似乎retypd直接使用了ghidra的结构体划分。这一块的识别需要单独做。


> 用索引编写的程序可能会被优化编译器转换为操作指针的等效代码

是否可以用一个分析把这个过程逆向转回去。不过我们静态分析不知道更需要哪种？

1. 用运行指针遍历数组
1. 用运行指针，复杂终止条件遍历数组：无法预知数组大小
1. 用运行指针，访问结构体数组。

> 类型分析是少数可以用数据流术语表达的问题之一，并且是真正双向的。
1. 定义的类型影响使用的类型
1. 使用的类型影响定义的类型：
1. 库函数调用（看作使用）影响之前的定义，已知返回值类型的赋值影响之前的定义。引用参数调用影响之前的定义。

> 仅考虑单个指令本身（至少在某些情况下）不足以分析聚合类型。要么必须在约束系统之外添加一些辅助规则，要么可以将表达式传播与高级类型模式结合使用。

如何判断数组？可能可以通过，取数组下标的是变量，来表示。


### 多种可能的解

反编译可行解过多怎么办？用更高层次的lattice值表示？

例如下面这个很难的例子

> 
> $m[1000_1] := m[1000_2] + 1000_3$
> 
> 三种可能的结果是：
> 
> -  $T (1000_1) = int*$ and $T (1000_2) = int*$ and $T (1000_3) = int$
> - $T (1000_1) = α**$ and $T (1000_2) = α**$ and $T (1000_3) = int$
> - $T (1000_1) = α**$ and $T (1000_2) = int*$ and $T (1000_3) = α*$

对于这种不知道什么类型的指针，可以在lattice上用void*临时表示一下？

约束求解是否不可避免？即如何让加法，减法做出三种选择（数字+指针，指针+数字，数字+数字）中的一种？

