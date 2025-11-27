---
title: 多态类型推理SimpleSub和MLsub
date: 2025/11/20 11:11:12
categories:
- Read
tags:
- StaticAnalysis
- PL
---

多态类型推理SimpleSub和MLsub

<!-- more -->

## 简介

SimpleSub 和 MLsub是编程语言理论（PL）中的类型推理方向的，一系列支持多态子类型推理的算法框架。子类型（可以想象为继承和接口之类的）其实在现实编程中用得特别多，而多态类型（可以想象一下C++的模板）其实用的也多。这个算法可以让编译型语言也不用写很多类型标注，同时它在程序分析领域也可能有着很广泛的应用。

## 资源总结

按时间线：

- 2016 Dolan的PhD paper，提出了被称为[MLsub](https://www.cs.tufts.edu/~nr/cs257/archive/stephen-dolan/thesis.pdf)的系统，是最原始的资源。但是偏理论。到中间才介绍自动机和图等算法，最后的算法不涉及前几章的理论。（我反复看了，略懂一些，有问题可以发邮件问我）[代码](https://github.com/stedolan/mlsub)是OCaml写的。
- 2020 SimplSub [论文](https://infoscience.epfl.ch/entities/publication/106da598-3385-4029-892b-27ea85194046)和[代码](https://github.com/LPTK/simple-sub)和[博客](https://lptk.github.io/programming/2020/03/26/demystifying-mlsub.html)
- CubiML系列[博客](https://blog.polybdenum.com/2020/07/04/subtype-inference-by-example-part-1-introducing-cubiml.html)，以及系列代码[CubiML](https://github.com/Storyyeller/cubiml-demo) -> [IntercalScript](https://github.com/Storyyeller/IntercalScript) -> [polysubML](https://github.com/Storyyeller/polysubml-demo)。

入门优先看几篇博客，然后考虑看SimpleSub论文，这个论文作者专门希望没什么基础的人也能看懂，对着代码讲。然后可以考虑看看相关代码。MLsub原代码和论文比较难懂，优先级最低。

## Introduction

简单来说：
- **子类型**：子类型的本质是，如果某个类型在任何情况下都能替代另外一个类型被使用，我们称这种情况下为子类型。这种替换关系很直接，也很核心。
- **多态**：一段程序可以接受不同类型的变量，使得它在不同上下文中有着不同的类型。可以想象一下C++的模版。但是python通过动态检查对象的类型不属于这种情况。

应用的角度：
- **新的编程语言**：想象一下，比如C++语言，所有的引用和指针不需要声明类型，只需要写auto即可。现在的auto只支持明确有类型的地方，而且只根据定义的初始化的时候确定类型。而这个算法可以根据变量的使用去综合确定类型，比如直接写不带初始化的auto变量。总之，它可以让你使用更多的auto。如果类型出问题了，编译器会告诉你的。不过分配堆、栈上的结构体对象的时候还是得完整声明具体的类型。
- **更深入的静态分析**：
    - **基于类型的安全检查**：比如Rust语言的生命周期就是一种可以在类型中表示的特性，而编译器需要推理出没有标注的地方的生命周期，使得程序不会出错。这个算法可以应用于这种情况，让语言编写者写出更多的基于类型的安全检查。
    - **加速动态语言**：比如pypy这种转译python的方式，通过深入的类型分析，说不定可以让更多的python程序直接被编译出来，极大增加速度。


从PL的角度来说，
- SimpleSub与MLsub**不需要很深入的基础知识**：现在的类型推理算法其实并没有进步很多，入门PL与类型系统的经典书籍《Types and Programming Languages》 TaPL中就介绍了ML语言的基于unification的类型推理。
- **系统更加简单高效**：后续很长一段时间的类型推理系统，它基于单独维护一个约束集合，和约束求解器那边有一些关联，效率低且复杂。这次的算法高效且简洁。

## MLsub

传统的 Hindley-Milner 类型推断是基于Unification算法的过程，该过程通过不断地强迫两个未知类型相等，直到达到矛盾或所有程序约束满足为止。在MLsub中，Dolan 提出了一个叫做biunification的过程。biunification的一个关键部分是极性类型系统。极性意味着类型被分为两种类型，传统上称为正类型（+）和负类型（-）。[这个](https://blog.polybdenum.com/2020/07/11/subtype-inference-by-example-part-2-parsing-and-biunification.html)里面说的很好，为了使理解更加容易，我称之为值类型（+）和用法类型（-）。为了避免导致半统一不可判定的无限循环，biunification将所有子类型约束限制为 v <= u 的形式，其中 v 是值类型，u 是用法类型。这些约束可以自然地解释为要求程序值与其使用方式相兼容。

**类型变量**：基本原则是，为每个表达式创建一个值类型（+），为每个表达式操作数创建一个用法类型（-），并在每个值类型（+）与其使用上下文（-）之间建立子类型约束，以确保一致性。我们将这些约束 v+ <= u- 称为值流向其使用，它们是通过 TypeCheckerCore 中的 flow 方法创建的。

还有一个更复杂的方面——变量。变量由一对类型表示——值类型（+）和用法类型（-）。从概念上讲，值类型表示从变量读取的类型（），而用法类型表示分配给该变量的类型。自然，我们需要约束 v- <= v+，即对变量的每一次写入与对该变量的每一次读取都是兼容的。然而，这种类型的约束我们使用数据流边直接表示。它确保流关系的传递性。对于每个变量 (v1, u1)，以及每个流向 u1 的值类型 v2 和 v1 流向的每个用法类型 u2，我们添加约束 v2 流向 u2。本质上，变量（一对连接了数据流边的类型节点）在类型图中像小隧道或虫洞一样运作。无论从一端进入的是什么，都会从数据流另一端出来。直接把下界/上界类型约束沿着数据流边传递。

