---
title: 盘点开源Fuzzer的策略-LibAFL-AFL++-Honggfuzz
date: 2025/04/02 11:11:12
categories:
- Dev
tags:
- Fuzz
---

盘点开源Fuzzer的策略-LibAFL-AFL++-Honggfuzz

<!-- more -->

## 种子调度策略

**Honggfuzz**

在December 21st, 2025，Honggfuzz更新了新的调度函数`power_calculateEnergy`，这次在种子调度方面直接涵盖了大量的因素

- Phase-aware energy: 在探索节点，倾向于小于256的种子，给1.5倍分数
- Novelty: 根据种子覆盖到的新的边的数量，给指数级增益，最大乘以`2**8`。随着时间递减，每隔10分钟递减一级
- Density：根据coverage情况，优先fuzz 总coverage更高的种子。
- Speed：优先速度更快的种子
- Fertility：优先生成更多子种子的种子。
- Freshness：优先新来的种子，60s内给4倍，5分钟内2倍，60分钟开外给0.5倍
- Size：更小的种子
- Stack depth：倾向于栈深度更深的种子。
- Execution path diversity：有unique执行路径的种子，如果pathHash不为空。
- CMP progress：绕过更多比较运算的种子。
- Rare edge bonus：覆盖了其他种子没有覆盖的边
- Depth：倾向于派生深度低的种子。
- Stagnation：当最近60秒都没有覆盖新的边的时候，倾向于coverage更高的种子。
- Entropy：计算输入的熵，惩罚总体看起来更随机的种子。
- timedout：如果超时了，除以2**5。

**AFL++**

AFL采用了[别名方法](https://zh.wikipedia.org/wiki/%E5%88%A5%E5%90%8D%E6%96%B9%E6%B3%95)，实现了快速的O(1)采样。但是每次来了新种子需要重新更新别名表。

