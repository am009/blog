---
title: Fuzzing-MOpt-Mutator
date: 2025/04/02 11:11:12
categories:
- Dev
tags:
- Fuzz
- Read
---

Fuzzing-MOpt-Mutator。再次阅读这篇paper《MOpt: Optimized Mutation Scheduling for Fuzzers》

<!-- more -->

### 主要思想

这里解决的问题是fuzzing过程中各种mutator的概率应该设置为多少比较好。这篇Paper引入了粒子群优化（Particle Swarm Optimization, PSO）。

**主要思想**：一个很简单的思想就是根据fuzzing过程中的结果去动态调整。先给一个基础的概率，然后在Fuzzing的过程中观察。众所周知，Fuzz过程中,会不断选取种子，随机使用mutator进行变异。如果当前的种子并没有覆盖额外的全局边，那么该种子将会直接被抛弃。如果产生了好的种子则会加入Corpus。如果当前Fuzz循环产生了好的种子，记录此时用到的mutator都是哪些，那么说明这些mutator肯定是更好的，应该增加他们的概率。

**为什么有粒子？**：但是具体增加多少呢？反正得朝那个方向调整。如果我们把这个从零到一的一维的概率空间，上面某个mutator被使用的可能性看作一个粒子，那我们调整概率本质上就是调整这个粒子的位置。那么我们可以采用速度和加速度的办法调整概率，根据上面统计出来的好的mutator的情况。把好的mutator对应的粒子给他增加一个向概率增加方向的速度或者加速度（具体怎么做需要学习粒子群优化算法）。

**局部优化方案与全局优化方案**：一方面，我们可以根据最近的固定数量的Fuzz尝试（最近的表现，局部表现），里面每个mutated的表现去调控，这对应了论文里面的Local Best Position。另外一方面，我们也可以统计整个fuzz过程的Mutated表现（全局表现）调控，对应了论文里面的Global Best Position。最后，论文尝试同时，维护这两个粒子群，选出表现比较好的用来调控mutator的概率。
