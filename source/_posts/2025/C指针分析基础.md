---
title: C指针分析基础
date: 2025/09/01 11:11:12
categories:
- Read
tags:
- PL
- StaticAnalysis
---

C指针分析基础。

<!-- more -->

## 什么是C语言指针分析？

首先你要比较了解C语言的指针。


## 稀疏指针分析

我们都知道LLVM那种SSA形式是半-SSA，因为还是有内存中的变量。其中的道理是，内存通过指针访问，指针存储导致的定义点不明确。

如果存入某个指针，有两种情况：
1. 指针只可能指向一个地方：此时一定是为那个内存地址定义了新的变量。
1. 指针可能同时指向多个地方：此时可能为不同的内存地址定义变量，而且后续的load也会“有可能”访问某些变量，不知道具体访问的谁。

总之造成了定义-使用（Use-Def）关系的混乱。

**流敏感指针分析和SSA的关系**：因为现有SSA，LLVM IR都是不完善的SSA。所以只有表层变量是SSA。因此，这样分析下来，内存中变量因为没有分版本，本质上还是流非敏感的。only context-sensitive for top-level variables

**流敏感指针分析的三大挑战**：

- 保守的传播：基于传统的CFG的传播方式，所有的指针指向信息都得传播到每个块上，即使这个块不会用到。因为算法是遍历基本块，极大增加了时间复杂度和内存需求。
- 复杂的转换函数：每个基本块要做gen/kill等集合的交集并集操作，复杂度随着集合大小线性增加。

**流敏感指针分析的优化方法**：

- 优先级队列：按照CFG节点划分优先级worklist，优先处理靠上的CFG节点，使得前面先迭代，后面的块获取的就是更靠近最终结果的的IN信息，节约迭代时间。
- 函数调用时过滤有用的信息：被调函数能影响到的范围就是传入的对象，以及全局对象。其他无关的对象不用传入函数里面。

### 基于Memory SSA的稀疏指针分析

## 指针访问的SSA表示



## 参考文献

### 最简单的指针分析

- Anderson的指针分析论文
- 《Points-to Analysis in Almost Linear Time》 Bjarne Steensgaard 1996：最经典的基于unification的指针分析。

### 指针访问的SSA表示

- Effective representation of aliases and indirect memory operations in SSA form.


### 指针分析



## 两阶段的稀疏指针分析。

- Semi-Sparse Flow-Sensitive Pointer Analysis
- the ant and the grasshopper: fast and accurate pointer analysis for millions of lines of code PLDI07

