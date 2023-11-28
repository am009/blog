---
title: 解读Fuzzer中的Mutator实现
date: 2023/11/27 11:11:12
categories:
- Hack
tags:
- Fuzz
---

解读Fuzzer中的Mutator实现

<!-- more -->

本篇文章基于LibAFL讲解。需要有一定的Rust基础。

[LibAFL里对Mutator的介绍](https://aflplus.plus/libafl-book/core_concepts/mutator.html)很简单。Mutator是一个[Trait](https://docs.rs/libafl/0.11.1/libafl/mutators/trait.Mutator.html)，包含两个函数，mutate和post_exec。同时，[Stage](https://aflplus.plus/libafl-book/core_concepts/stage.html)也很重要，Mutational Stage也会对语料库里的输入进行修改。

### [honggfuzz](https://honggfuzz.dev/)的mutator

honggfuzz的mutator就放在最外层的`honggfuzz/mangle.c`和`mangle.h`里。在mangle.c的最底部即可看到一个mangleFuncs数组。

一个值得注意的是，里面的随机数有很多是用的选取函数`mangle_getLen`，会经常倾向于更小的数字。

1. [mangle_Shrink](https://github.com/google/honggfuzz/blob/88709ce60f45ee13666a2628f03467c57429c7db/mangle.c#L687)：随机选取一个offset，然后选取一个长度（大部分情况是0-16随机选，1/16的可能性是0到<当前offset到末尾的距离>里选），然后删掉从这个offset到offset+len的数据。
1. [mangle_Expand](https://github.com/google/honggfuzz/blob/88709ce60f45ee13666a2628f03467c57429c7db/mangle.c#L675)：同上选取一个长度：（大部分情况是0-16随机选，1/16的可能性是当前offset到<程序最大输入长度>里选）。然后把从offset开始的数据向后搬运len个距离，如果要求了printable，则会把中间新增的字节填充为空格。
1. mangle_Bit：修改某个bit
1. mangle_IncByte、mangle_DecByte：将某个字节加1或者减1。如果要求是printable就麻烦一点，在printable范围内带模加减。
1. mangle_NegByte：将某个字节取反。
1. mangle_AddSub：随机选择1，2，4字节范围的值看作整数，然后生成一个范围内的，可正可负的值，加上去。
1. mangle_MemSet：随机选择一个offset，然后从0到<当前offset到末尾的距离>里随机选一个长度，随机选一个byte，然后有50%概率memset，50%概率插入（此时类似Expand）。
1. mangle_MemClr：同上，但是byte选0（printable的时候选空格）
1. mangle_MemSwap：随机选择一个offset，然后从0到<当前offset到末尾的距离>里随机选一个长度。操作两次得到两个offset和长度，取较小的长度，然后交换内容。
1. mangle_MemCopy：随机选择一个offset，然后从0到<当前offset到末尾的距离>里随机选一个长度，这样选出一块内容。随机找个offset，有50%概率覆盖，50%概率插入（此时类似Expand）。
1. mangle_Bytes：在buf中随机插入或覆盖1-2个随机字节。
1. mangle_ASCIINum：在buf中随机插入或覆盖一个十进制（可带负号）数字。
1. mangle_ASCIINumChange：在buf中找到一个现有的十进制数字，然后修改。有8种可能：加1，减1，乘2，除2，覆盖为随机值，加减0-256的随机数，取反。
1. mangle_ByteRepeat：选取一段范围，插入或者覆盖，重复随机offset里的某个字节。
1. mangle_Magic：有220个magic值，随机选里面的插入或者覆盖。
1. mangle_StaticDict：随机选dict里的值插入或者覆盖。
1. mangle_ConstFeedbackDict：需要开启cmpFeedback模式，然后从cmp feedback数组里面随机选一个值出来。
1. mangle_RandomBuf：随机选择一个offset，然后从0到<当前offset到末尾的距离>里随机选一个长度，这样选出一块内容。然后插入或者填充随机的字节。

### AFL中的splice和havoc

AFL的[文档](https://afl-1.readthedocs.io/en/latest/user_guide.html)里有下面的介绍：

> havoc - a sort-of-fixed-length cycle with stacked random tweaks. The operations attempted during this stage include bit flips, overwrites with random and “interesting” integers, block deletion, block duplication, plus assorted dictionary-related operations (if a dictionary is supplied in the first place).

havoc是很多操作的叠加。

> splice - a last-resort strategy that kicks in after the first full queue cycle with no new paths. It is equivalent to ‘havoc’, except that it first splices together two random inputs from the queue at some arbitrarily selected midpoint.

splice会选择两个输入，在某个位置把它们拼接起来。

其他介绍：
- [AFL-变异策略](https://www.zrzz.site/posts/49460ecb/)
