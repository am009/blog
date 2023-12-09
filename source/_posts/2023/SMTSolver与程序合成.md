---
title: SMTSolver与程序合成
date: 2023/12/08 11:11:12
categories:
- Read
tags:
- PL
---

SMTSolver与程序合成。如何合成命令式语言？

<!-- more -->

## SMT Solver

### 相关资料

搜索Many-sorted logic SMT 找到一些直接相关的课。

1. 《面向计算机科学的数理逻辑系统建模与推理》.pdf  这本书可以快速入门必要的逻辑学知识，而且本来就是讲SAT solver的。
1. https://users.aalto.fi/~tjunttil/2020-DP-AUT/notes-smt/part1.html
1. Z3支持的逻辑：http://theory.stanford.edu/~nikolaj/z3navigate.html  http://smtlib.cs.uiowa.edu/logics.shtml

熊英飞 《软件分析》课里也提到了相关的知识，从[第12节课](https://liveclass.org.cn/cloudCourse/#/courseDetail/8mI06L2eRqk8GcsW)开始:

- 第12课：SMT，基于SAT开始，把一系列理论结合起来。类似算数，数组，位向量就是一个理论。
- 第13课末尾：简单介绍了SMT-lib

### SAT和合取范式

- 1.4.1 前面介绍了基本的逻辑推理是什么，命题之间可以有哪些组合关系（合取，析取，蕴含，取反），有哪些公理去推导。这一节介绍了真值表，基于命题的语法树可以遍历根据子命题的真值得到整个命题的真值
- 1.4.3 证明真值表求值和自然演绎的推理规则，语义上是一致的
- 1.5.1 $$\phi \rightarrow \psi \equiv \neg \phi \vee \psi$$ 利用这个规则，可以将所有的命题转换为不带假设和推导的。进一步，都可以转换为CNF 合取范式。合取范式中都是先and，再or，再not，再其他命题。

    其次，对于合取范式，能够很有效地直接判断它是否恒为真（有效）。需要判断每一个and的规则为真，然后看这些由or组合的命题，是否存在q or (not q)的这种形式。（注：所有变量都是可以随意取值的，不然就可以直接代入化简了）。

    为什么要判断有效性？因为本质上和可满足是紧密相关的。如果想证明命题Q是可满足的，即存在某种取值使得Q为真，只需要证明Q不会一定为假，即not Q不是恒为真（有效）。

    如何转换为CNF等价公式？如果能列出完整的真值表就可以转换，但也因为要真值表，所以很难。我们对于真值表里每个为false的项，构造False or False or False...这种形式，长度等价于自由命题变量的个数。然后把每个False替换为q或者not q，根据q是否为真。最后把所有这种项目and起来即可。

    即使都是CNF，它们之间也有相互等价的命题。比如p和 p and (p or q)就是等价的，且都是CNF。可以编写一个算法，利用结构递归，基于推导消除，and和or交换顺序的分配律，not下降到子句的分配律，可以把任何命题转换为CNF。但是这个CNF并不是最简的，很可能很长。

**零阶逻辑和一阶逻辑的区别是什么？** 一阶逻辑引入了全程量词，for all 和 exist 这种复杂的东西。

**什么是合取范式？** 合取范式中都是先and，再or，再not，再其他命题。

### SMT Solver

**算术运算转换为SAT**：逻辑门可以转化为SAT约束。bitVector的运算可以基于加法和乘法电路，转换为SAT问题。但是有性能开销，16bit的乘法就有几万个变量。

**SAT的应用**：电路设计时，可以应用SAT证明电路的等价性。把两个电路异或起来，证明输出总是0。密码分析的时候，可以用于破译密码。

## Syntax-Guided Synthesis

### 相关的paper和项目

**[SyGuS: Syntax-Guided Synthesis](https://sygus-org.github.io/)** 它总结出了现有的大量的程序合成的共性，提出了通用的问题框架，涵盖了之前研究的很多子问题。

**相关Solver**：可以在[这个benchmark页面](https://sygus-org.github.io/comp/2019/)看到参与了比赛的solver：
- [CVC4](https://github.com/cvc5/cvc5)：可以通过`cvc4 --lang=sygus`调用

**[SemGuS: Semantics-Guided Synthesis](https://www.semgus.org/)** 它基于SyGuS，通过转化为Constrained Horn Clause，支持任意的语义表达，包括命令式语言的循环和赋值等语句。它也是一个程序合成框架，而不是具体的solver。它提供了对程序合成任务描述的统一格式（它们把自己类比为LLVM IR，作为各种工具的兼容层），鼓励各种相关工具支持它这种格式，同时提供一些benchmark用来测试合成工具性能。

在描述合成任务的树语法文件中，它需要描述想要生成程序的语法，语义。

**相关Solver**

- [Semgus-Messy](https://github.com/SemGuS-git/Semgus-Messy)
- [The ks2 synthesizer suite](https://github.com/kjcjohnson/ks2-mono)

**相关资料**

1. Sygus相关课程
    1. SyGus的简介：https://people.csail.mit.edu/asolar/SynthesisCourse/Lecture1.htm 
    2. https://people.eecs.berkeley.edu/~sseshia/219c/lectures/SyGuS.pdf 
    3. https://web.stanford.edu/class/cs357/lecture15.pdf
    4. https://simons.berkeley.edu/sites/default/files/docs/17371/andrewreynoldstfcssynthesisslides.pdf
    5. https://synthesis.to/presentations/introduction_to_program_synthesis.pdf 程序合成+二进制分析


**其他**

[Oracle-Guided Component-Based Program Synthesis](https://people.eecs.berkeley.edu/~sseshia/pubdir/synth-icse10.pdf) 第一个提出程序合成可以用于反混淆的工作。虽然其实并没有特别直接达到这个目标。

### 摘抄

https://arxiv.org/pdf/2008.09836.pdf
> Program synthesis has been studied from a variety of perspectives, which have led to great practical advances in specific domains [Feser et al. 2015; Gulwani 2011; Phothilimthana et al. 2019].


