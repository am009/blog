---
title: Fuzz快速上手指南
date: 2023/12/03 11:11:12
categories:
- Hack
tags:
- Fuzz
---

Fuzz快速上手指南

<!-- more -->

不得不说，学习使用LibAFL的过程，就是学习Fuzz架构的过程。基于LibAFL的baby fuzzer教程，就可以了解到fuzzer的架构。LibAFL是基于Rust语言，高度可自定义的组件化fuzzer。他们的开发者正尝试用LibAFL复刻AFL++，libfuzzer等多个知名fuzzer，这足以说明LibAFL的强大。

Fuzzing的很重要的一部分就是调试崩溃和修复漏洞。这个和fuzz本身一样重要。这个暂时没涉及

本文涵盖以下内容：

- AFL的coverage map设计
    - PCGUARD模式
    - cmplog/input to state/redqueen
- Mutators
  - honggfuzz中的Mutator
  - AFL中的splice和havoc
  - MOpt

### 学习资源

- [fuzz101](https://github.com/antonio-morales/Fuzzing101)
- [AFL-TRAINING](https://github.com/mykter/afl-training)。

## Fuzzer 架构

### AFL的coverage map设计

即使是几年后的现在，最初[AFL的coverage map设计](https://github.com/google/AFL/blob/master/docs/technical_details.txt)依然被沿用。

本质上AFL的coverage map还是衡量的边覆盖率。很自然会想到，通过插桩在运行时维护一个从每条边到它执行时经过的次数的map。然而这样做了之后可能占内存很多，也很慢。和Hash函数的思想类似，通过放弃可解释性，放弃追溯具体是那条边的能力，以增加效率。

AFL在每个分支处插桩以下代码。每次fuzz执行前将对应coverage区域清零，执行结束后shared_mem（默认64 kB）被填充了每条边执行次数的信息。

```c
  cur_location = <COMPILE_TIME_RANDOM>;
  shared_mem[cur_location ^ prev_location]++; 
  prev_location = cur_location >> 1;
```

在执行时，每个分支处的随机数就代表了当前分支。每个随机数右移一位后就代表它作为prev_location时的值。异或操作得到的随机数代表具体的一条边。可以想象这些边作为下标均匀分布在数组的槽中，冲突概率较小，从而大致数组每个元素代表每个边经过的次数。

这个方法有明显的几个优点

- 插桩时只需要生成随机数，时间和内存效率高
- 运行时仅对每个分支增加几条汇编指令的开销，时间效率高。
- shared_mem内存大小可调，内存效率高。

对于fork server架构的fuzzer，这块内存可以直接作为共享内存，直接读取子进程的coverage map。对于in proc的fuzzer，则直接读取对应内存位置。此时该shared_mem区域可能作为一个全局数组/指针，在静态/动态链接时作为符号导出，fuzzer通过名字引用。

**coverage map过小可能产生大量冲突？**：确实有可能，而且这也是AFL++的[lto模式的优化](https://github.com/AFLplusplus/AFLplusplus/blob/stable/instrumentation/README.lto.md)之一。随机数的随机特性，肯定会没那么均匀，导致部分数组成员同时记录了多条边，然后有些数组成员没有代表任何边。

LTO指link time optimization.平时编译是按照编译单元，一个文件一个文件直接转换为汇编，然后在汇编层链接。在LTO模式下，编译器每次编译一个文件会先不生成汇编，而是保留便于分析的IR到object文件。在链接到一起的时候，即生成可执行文件的时候，此时有了程序的所有IR代码，能做的优化也更多了。

LTO允许部分优化推迟到链接后进行。之前（AFL-LLVM模式）是分文件生成汇编，此时每个编译单元内，插桩逻辑单独运行，为分支生成随机数。如果能把插桩推迟到最后一起做，则可以在生成随机数的时候相互协调，保证coverage map里每个数组成员都只代表一条边。

LTO模式不仅代码优化更好，而且插桩也有无冲突的优点，因此它也是使用AFL++时推荐能开则开的模式。

**coverage衡量指标的优化**

AFL++也提供了很多[其他模式](https://github.com/AFLplusplus/AFLplusplus/blob/stable/instrumentation/README.llvm.md#3-options)

- Context sensitive coverage：区分不同caller。比如函数A和B都调用了C，那么C里的同一条边，因为考虑的调用者，反馈的覆盖信息就会不同。 `map[current_location_ID ^ previous_location_ID >> 1 ^ hash_callstack_IDs] += 1`
- n-gram branch coverage：根据之前从哪个边走过来的，区分不同的当前边：`map[current_location ^ prev_location[0] >> 1 ^ prev_location[1] >> 1 ^ ... up to n-1] += 1`

**其他**

- 多线程并发修改coverage map时可能存在竞争问题：设置`AFL_LLVM_THREADSAFE_INST`

### 边执行次数的bucket设计

coverage map的成员是1字节，即会记录这条边从0到255的执行次数。AFL进一步细分，把执行次数分为如下8类。即，在程序执行完后，对每个字节应用一个函数，从0-255映射到0-7。

> 1, 2, 3, 4-7, 8-15, 16-31, 32-127, 128+

虽然信息量减少了，但是这样能关注更重要的信息：从执行1次，到执行2次或三次，有可能就有了新的突破，对一个执行了很多次的循环的次数增加则不需要那么敏感。


### AFL的Queue设计

除了每次程序执行的coverage map，AFL维护一个全局的coverage map，包含了之前执行时遇到的所有边。如果某次执行发现了新的边，则当前的输入就会认为是interesting的，放入队列中。

另外，每次执行都会设置超时时间。背后的思想是，即使有些输入能增加1%的coverage，但是会花了上百倍的执行时间，从务实的角度考虑也要排除这种情况。说不定后面就有更高效的方式覆盖到相同的代码了。毕竟速度和效率是fuzzing的精髓之一。


### 什么是PCGUARD？

PCGUARD是AFL的默认模式，名字[可能源自插桩pass的名字](https://stackoverflow.com/a/68919788/13798540)。它就是简单地在每个基本块前插桩。

### 什么是persistent mode？

[AFL++的文档](https://github.com/AFLplusplus/AFLplusplus/blob/stable/instrumentation/README.persistent_mode.md)介绍了，persistent mode是指在fork server架构下，每次fork会fuzz多次target。可以带来10-20x的速度提升。

这里指，AFL会在运行二进制后，让程序停在main函数，然后每次fuzz前尝试复制这个进程去fuzz，以避免初始化的开销，有点类似fork。如果程序有复杂的初始化逻辑，还可以选择合适的位置，推迟这里“fork”的点。



### `cmplog`/`input to state`/`redqueen`

当fuzzing遇到多字节比较的时候，很难通过随机变换生成对应的字节出来。[AFL++的fuzzing_in_depth.md](https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/fuzzing_in_depth.md#b-selecting-instrumentation-options)里有一些简单的介绍。

- `laf-intel`/`COMPCOV`：将多字节比较，拆分为很多单字节比较。
- `cmplog`/`input to state`/`redqueen`：当出现这种多字节比较的时候，对应的字节会被反馈给AFL++，作为关键词用于mutate填充。这种方式比前一种更加高效。


## Mutation

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

> 第一次进行变异的种子文件首先要进入deterministic stage，在这一阶段每个变异操作会对种子文件的每个byte或者bit进行变异，以此生成庞大数量的测试用例来测试目标程序。在结束了deterministic stage后，进入havoc stage，AFL从变异操作中随机选Ro个对种子文件进行变异，并使用变异后的测试用例来测试程序。第三个阶段是splicing stage，进入这一阶段的判断条件很苛刻，如果AFL变异了seed pool中的所有种子文件，得到的测试用例仍然没有发现新的interesting test cases，AFL才会执行这一阶段，splicing stage只执行一个变异操作：随机选取另一个种子文件，将它和当前文件的部分内容拼接在一起，然后重新进入havoc stage进行变异。（摘自[InForSec通讯](https://www.inforsec.org/wp/?p=3950)）

AFL的[文档](https://afl-1.readthedocs.io/en/latest/user_guide.html)里有下面的介绍：

> havoc - a sort-of-fixed-length cycle with stacked random tweaks. The operations attempted during this stage include bit flips, overwrites with random and “interesting” integers, block deletion, block duplication, plus assorted dictionary-related operations (if a dictionary is supplied in the first place).

havoc是很多操作的叠加。

> splice - a last-resort strategy that kicks in after the first full queue cycle with no new paths. It is equivalent to ‘havoc’, except that it first splices together two random inputs from the queue at some arbitrarily selected midpoint.

splice会选择两个输入，在某个位置把它们拼接起来。

其他介绍：

- [AFL-变异策略](https://www.zrzz.site/posts/49460ecb/)

### MOpt: Optimized Mutation Scheduling for Fuzzers

[InForSec通讯](https://www.inforsec.org/wp/?p=3950)

MOpt这篇paper，改进了AFL里面havoc阶段的Scheduler的过程。

LibAFL里也实现了这篇paper的方法。

### LibAFL里其他的mutation机制

LibAFL的mutation stage的主要逻辑在这个[`perform_mutational`](https://github.com/AFLplusplus/LibAFL/blob/d53503b73ea0425ffdcfbc467c167b02632077b6/libafl/src/stages/mutational.rs#L118)函数。

- 首先确定为了当前输入，打算进行多少次mutate，存入变量num。
- MutatedTransform机制，可以根据testcase和state转换输入，目前似乎只是从TestCase解出input。
- 循环num次
  - 复制一份输入
  - 对输入调用mutator变换：mutator可以返回MutationResult::Skipped，表示变换失败，直接continue
  - 运行输入，判断是否interesting（`fuzzer.evaluate_input`）。
    - evaluate_input_with_observers：执行程序，执行observer，执行`scheduler.on_evaluation`，执行`process_execution`。
      - process_execution这里会判断是否找到bug（`ExecuteInputResult::Solution`），或者interesting（`ExecuteInputResult::Corpus`），或者都不是。
        - 对于interesting的输入，复制
  - 执行Mutator的post_exec
