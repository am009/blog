---
title: AFLplusplus的enhancedDeterministicMode
date: 2025/04/03 11:11:12
categories:
- Dev
tags:
- Fuzz
---

AFL++的[Enhanced Deterministic mode](https://github.com/AFLplusplus/AFLplusplus/pull/1972)的实现解析

<!-- more -->

deterministic mode是最开始的时候AFL就有的一个和havoc并列的mutate阶段。而后续AFL++则进一步改进了这一阶段，增加了效率。本文介绍相关的内容。

## 之前的deterministic mode

[《AFL++: Combining Incremental Steps of Fuzzing Research》](https://www.usenix.org/system/files/woot20-paper-fioraldi.pdf) 里面提到了AFL++的mutation分为两个主要部分，分别是这里解析的确定性变异（deterministic）和havoc。确定性变异阶段，AFL++会对种子进行单次的确定性的变异。使用的mutation通常包括bit blips, addition, 用常见的整数值，比如INT_MAX, -1等进行替换（substitution）。

[这个网站](https://blog.adacore.com/advanced-fuzz-testing-with-aflplusplus-3-00)提到，deterministic策略会尝试bit flip每个test-case的bit。

这一确定性的过程比较花时间，所以在fuzz初期的覆盖率上，开了会比不过关闭deterministic mode。但是如果关闭deterministic mode会直接缺失一部分的mutator，所以最后coverage收敛后，开启deterministic模式会高一点。

### Effect Map

如果某个字节，完全翻转（fully flipped）都不会对执行的路径造成影响，则跳过对这些字节的deterministic阶段。

比如，如果一个值被用作for循环的循环次数，那么翻转了肯定是影响执行路径的，此时在deterministic阶段使用一些int_max或者-1这种值进行mutate就很有用，但是如果是普通的无关执行路径的值，那么浪费时间在上面也没有意义。

原版的AFL中就有了Effect map机制，在[这里](https://github.com/ThePatrickStar/AFL-Original/blob/d09714865d3a211a9c1a682eaa9bb8b00327e842/afl-fuzz.c#L5421)。


## Enhanced deterministic mode

原始的Pull Request: https://github.com/AFLplusplus/AFLplusplus/pull/1972

在2024年1月份左右，有人提交了PR改进了deterministic模式，跳过了很多不必要的mutation。增加了总体的效率。在fuzzbench上的表现有所提升。在openthread_ot-ip6-send-fuzzer上提升很大。在sqlite3_ossfuzz，freetype2_ftfuzzer，woff2_convert_woff2ttf_fuzzer上略有提升。

这个Patch里面最核心的文件`afl-fuzz-skipdet.c`里面的最核心的函数是`skip_deterministic_stage`。在fuzz-one里面插入了对skip_deterministic_stage的调用，该函数设置了skip_eff_map(初始都为0)，然后正常执行deterministic mode的时候，如果当前mutate的字节在skip_eff_map里面的值是0，则会直接跳过该字节。

### skip_deterministic_stage

#### should_det_fuzz

首先，使用`should_det_fuzz`函数做初步的判断：

- 如果种子不是favored，则直接跳过deterministic stage（`!q->favored`）
- 基于coverage map和virgin_det_bits进行考虑。virgin_det_bits是全局的一个数据结构，和coverage map大小一样。如果当前的种子的coverage map 的某个bit是1，而全局的这个virgin_det_bits对应的地方不是1，则给new_det_bits加1。如果new_det_bits大于undet_bits_threshold则

undet_bits_threshold是一个动态的阈值，每隔一段时间（20分钟），如果大于2，会乘以0.75。

#### 推理阶段（Inference Stage）

首先，在队列的每个种子的数据结构里，增加了`skipdet_entry`成员：
- `u8 continue_inf`
- `u8 done_eff`
- `u32 undet_bits`
- `u32 quick_eff_bytes`
- `u8 *skip_eff_map` 最重要的map，如果设置某个字节为0，则表示跳过。
- `u8 *done_inf_map`：

首先这个节点会把skip_eff_map都设置为1，表示默认不跳过。

