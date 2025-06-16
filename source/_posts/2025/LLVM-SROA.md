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

存在着通用的分析，分析指针的escaped特性和aborted特性。

指针会标记为escaped，如果：

1. 
