---
title: LLVM的SROA解析
date: 2025/04/02 11:11:12
categories:
- Dev
tags:
- LLVM
- Compiler
---

LLVM的[SROA（Scalar Replacement of Aggregates）](https://llvm.org/docs/Passes.html#sroa-scalar-replacement-of-aggregates)的实现解析

<!-- more -->

FCA: First Class Aggregate（第一类聚合体，如结构体或数组）

## 总体思路

SROA是一个Function Pass，输入是单个函数。

- [`SROAPass::runImpl`](https://github.com/llvm/llvm-project/blob/e188aae406f3fecaed65a1f7e6562205f0de937e/llvm/lib/Transforms/Scalar/SROA.cpp#L4722) 里面的主体while函数包含了如下的步骤：

1. **提前promote简单alloca**：首先遍历entry基本块内的alloca指令。如果某个Alloca指令可以直接被Promote（[`isAllocaPromotable`](https://github.com/llvm/llvm-project/blob/e188aae406f3fecaed65a1f7e6562205f0de937e/llvm/lib/Transforms/Utils/PromoteMemoryToRegister.cpp#L65)），简单来说仅被load和store，则提前给promote掉，用Mem2Reg相关的SSA构建函数。否则就加入工作队列中依次分析处理。
2. **runOnAlloca**：处理剩下的每个alloca指令
    - 维护了PromotableAllocas队列，存储可以被提升的Alloca指令。在循环结尾提升并清理。
    - 维护了DeadInsts，存储等待被删除的指令，由函数deleteDeadInstructions删除
    - 维护了DeletedAllocas，从DeletedAllocas里

### 处理每个alloca指令（runOnAlloca）

1. 如果没有任何User，或者是Array形式的alloca指令（即alloca的第二个操作数不是1）则直接不处理。
1. 使用`AggLoadStoreRewriter`处理带有extractvalue指令的复杂的结构体的load和store。将结构体的load和store拆分为每个成员的load和store，然后再用extractvalue/insertvalue分开或者拼在一起。
1. **构建AllocaSlices**，如果认定该alloca为escaped，则放弃
1. 删掉没有User的一些地址的引用，比如bitcast（DeadUser）和DeadOp（即Alloca被Gep指令的使用，但是这个operand不可能被取到）
1. 调用**splitAlloca**，基于计算得到的Alloca Slice分割。
    1. **presplitLoadsAndStores**将过大的批量内存复制给分割开。
    1. 如果某个Slice和别的Slice要么不相交，要么互为包含关系就还好，如果出现了交替重叠则标记为无法split。
    1. 排序后依次调用**rewritePartition**重写每个分区。如果重写成功就放到Fragments。
        1. 确定拆分后的Alloca的具体类型。找不到就用相关的整数或者i8数组
        1. 创建新的Alloca。
        1. **AllocaSliceRewriter** 访问每个Slice重写访问，同时判断是否Promotable。
        1. 可以提升则提升，不可以提升则回退相关结果然后返回
    1. 修复每个Fragment的DebugInfo。

对于一个很大的alloca，会尽量将内部标记为unsplittable的

### **AllocaSlices**：Alloca能否被拆分的判断

`AllocaSlices`的构造函数里调用了用于构建的`AllocaSlices::SliceBuilder`，它继承了`PtrUseVisitor`，递归遍历指针的Use树，分析指针的escaped Flag和aborted Flag。如果遇到了指针难以处理的情况，指针会标记为escaped。每次Visit某个Use时，会将成员变量`Use* U`设置，然后再把User cast为指令然后访问。

- alloca可以被store和load读写。但是alloca自身的地址不能被store到别的地方。根据Load和Store的边界会分割Slice。
- 可以被Gep指令计算，但是如果计算得到了未知偏移的地址，则放弃处理。
- 遇到内存转移指令，memcpy/memmove，会根据复制点分割出Slice。

继承后拓展的逻辑里，，额外维护了alloca的分割点，并创建Slice，partition等数据结构：

- `Slice`表示一个Alloca的位置中的一个部分`[BeginOffset, EndOffset)`同时有一个`Use*`成员和一个isSplittable Flag。这里的`Use*`存入的user一般是Load/Store指令。
- `Partition`是一种特殊的Slice数组的iterator，将相同范围内的Slice聚合起来一起返回。

**已知偏移的维护**：相关的逻辑在`PtrUseVisitor`里面首先在工作队列里面，存储Use的同时也会存储该指令基于的偏移。访问的时候，根据是否是什么gep指令去增加偏移。然后需要进一步访问的时候会调用`enqueueUsers`函数，然后在将Users放入工作队列的同时会存入当前的Offset。
