---
title: Fuzzing-MOpt-Mutator
date: 2025/04/02 11:11:12
categories:
- Dev
tags:
- Fuzz
---

Fuzzing-MOpt-Mutator。再次阅读这篇paper《MOpt: Optimized Mutation Scheduling for Fuzzers》

<!-- more -->

### 主要思想

这里解决的问题是fuzzing过程中各种mutator的概率应该设置为多少比较好。这篇Paper引入了粒子群优化（Particle Swarm Optimization, PSO）。

**主要思想**：一个很简单的思想就是根据fuzzing过程中的结果去动态调整。先给一个基础的概率，然后在Fuzzing的过程中观察。众所周知，Fuzz过程中,会不断选取种子，随机使用mutator进行变异。如果当前的种子并没有覆盖额外的全局边，那么该种子将会直接被抛弃。如果产生了好的种子则会加入Corpus。如果当前Fuzz循环产生了好的种子，记录此时用到的mutator都是哪些，那么说明这些mutator肯定是更好的，应该增加他们的概率。

**为什么有粒子？**：但是具体增加多少呢？反正得朝那个方向调整。如果我们把这个从零到一的一维的概率空间，上面某个mutator被使用的可能性看作一个粒子，那我们调整概率本质上就是调整这个粒子的位置。那么我们可以采用速度和加速度的办法调整概率，根据上面统计出来的好的mutator的情况。把好的mutator对应的粒子给他增加一个向概率增加方向的速度或者加速度（具体怎么做需要学习粒子群优化算法）。

**局部优化方案与全局优化方案**：一方面，我们可以根据最近的固定数量的Fuzz尝试（最近的表现，局部表现），里面每个mutated的表现去调控，这对应了论文里面的Local Best Position。另外一方面，我们也可以统计整个fuzz过程的Mutated表现（全局表现）调控，对应了论文里面的Global Best Position。最后，论文尝试同时，维护这两个粒子群，选出表现比较好的用来调控mutator的概率。

## 粒子群算法

推荐阅读这里 [《数学建模：非常通俗易懂的粒子群算法（PSO）入门》](https://zhuanlan.zhihu.com/p/398856271) 以及[论文的slides](https://www.usenix.org/sites/default/files/conference/protected-files/sec19_slides_lyu.pdf)的关键的两页

粒子群算法背后的直觉和原理请看上面的文章。我们这里详细解析MOpt是怎么应用过来的。

粒子群算法的核心公式如下（借用知乎文章的比喻，每个例子是一只鸟）：

这只鸟第d步的速度=上一步自身的速度惯性+自我认知部分+社会认知部分

$$
v_i^d = w v_i^{d-1} + c_1 r_1 (pbest_i^d - x_i^d) + c_2 r_2 (gbest^d - x_i^d)
$$

鸟的新位置 = 鸟在上一轮的位置 + 鸟当前轮的速度   (因为每一步运动的时间t一般取1)

$$
x_i^{d+1} = x_i^d + v_i^d
$$

和论文PPT里的说明一致：

![alt text](PSO.png)

## 为Fuzzing改进的粒子群算法

![alt text](Fuzz-PSO.png)

每个鸟对应了一个Mutator。所在的位置对应了概率。因此，一开始随机分布位置的时候，让他们位置加起来等于1。整个粒子群就对应了所有mutator的概率分布。

w是惯性权重。如果惯性太高，会到处冲去搜索解空间，容易找到全局最优解，但是收敛更慢。而r对应的是个体学习因子和社会学习因子的学习权重。反正后面会有多个粒子群，这里可以随机取？

关键是怎么评判哪里是更优的解。根据最近的固定数量的Fuzz尝试，mutator的表现被定义为Local Best Position，整个fuzz过程的Mutated表现，对应了论文里面的Global Best Position。

局部最优位置 LBest：在一次iteration中，某个mutator的效率被定义为，有趣的测试用例的数量，除以调用这个mutator的次数。而LBest是历史上效率最高的时候的所在位置（概率）。

全局最优位置 GBest：在所有swan（群）中，该粒子（变异操作）得到的有趣测试用例数目。
