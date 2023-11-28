---
title: Fuzz快速上手指南
date: 2023/11/27 11:11:12
categories:
- Hack
tags:
- Fuzz
---

Fuzz快速上手指南

<!-- more -->

本篇文章基于LibAFL讲解。需要有一定的Rust基础。

不得不说，学习使用LibAFL的过程，就是学习Fuzz架构的过程。基于LibAFL的baby fuzzer教程，就可以了解到fuzzer的架构。

Fuzzing的很重要的一部分就是调试崩溃和修复漏洞。这个和fuzz本身一样重要。

学习资源：

- [fuzz101](https://github.com/antonio-morales/Fuzzing101)
- [AFL-TRAINING](https://github.com/mykter/afl-training)。

### AFL的bitmap反馈


