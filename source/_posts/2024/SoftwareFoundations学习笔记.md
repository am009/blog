---
title: SoftwareFoundations学习笔记
date: 2024/11/03 11:11:12
categories:
- Read
tags:
- PL
---

SoftwareFoundations学习笔记。

<!-- more -->

不得不说，还是应该跟着各种课程，好好的去重新学习一下编程语言理论相关的知识。首先，我在问过很多大佬之后，他们提到的学习资源也就是这些东西。第二，奢求其他人耐心的给你讲解基础知识基本上是不可能的，所以这些东西都需要靠自己自学。最近完全没有耐下心来去好好学习，从现在开始冲刺还来得及。而且，这种基础知识类的东西，无论什么时候学，都是好处远大于投入。学了之后，基本马上就能用到。不学，只能浪费更多的时间自己推理。



https://coq.inria.fr/tutorial/

https://blog.vero.site/ref/coq

## 常用命令与环境配置：

手动命令行命令：

```
# 最好先运行make编译每个.v为.vo文件
coqc -Q . LF Basics.v
coqc -Q . LF BasicsTest.v
coqtop -Q . LF # 进入交互式模式。退出使用 `Quit.`
ledit coqtop -Q . LF # 直接使用coqtop 会无法使用左右箭头，上下翻历史等功能。搜了发现需要先`opam install ledit`然后用ledit执行coqtop
```

然后执行 `From LF Require Export Induction.` 导入当前在读的章节。

最基本的做题环境设置：VSCode下面开两个分屏的终端，上面看代码（即同时也是看书）。每次做题目就把上面的定理复制到下面coqtop。然后在下面手动证明。做好了把答案整理上去。

更好用的配置，首先安装vscoq插件并配置好，然后在要做题的地方使用快捷键Alt + 右箭头。插件会解释代码到文件当前位置。同时在右侧展示当前Goal。不断写语句并查看右侧。


## 1 逻辑基础 Basics
声明（Declaration），将一个名字和规范（specification）对应起来，有三种规范：logical propositions, mathematical collections, and abstract types。
- Hypothesis 假设，一个返回Prop（bool）类型值的logical propositions，
- Definition 定义，就是给个别名。Definition double (m : nat) := plus m m.
- Inductive 递归的定义，即定义里面可以用当前正在定义的类型。类似Fixpoint，但是Fixpoint更复杂。 
- Fixpoint 递归函数定义，可以函数体内用这个函数。 Coq 要求每个 Fixpoint 定义中的某些参数必须是“递减的”。

```
Fixpoint evenb (n:nat) : bool :=
  match n with
  | O ⇒ true
  | S O ⇒ false
  | S (S n') ⇒ evenb n'
  end.
```
- Check 类型检查。如果不加类型则打印类型。
- Example / Theorem /Lemma / Fact / Remark 都是表示一个需要证明的定理？

### 数据结构

有参数的构造函数：
```
Inductive color : Type :=
  | black
  | white
  | primary (p : rgb).
```

模式匹配：

```
Definition isred (c : color) : bool :=
  match c with
  | black ⇒ false
  | white ⇒ false
  | primary red ⇒ true
  | primary _ ⇒ false
  end.
```

元组：

```
Inductive nybble : Type :=
  | bits (b(0) b(1) b(2) b(3) : bit).
```

定义的数组的表示：

```
Notation "x :: y" := (cons x y)
                     (at level 60, right associativity).
Notation "[ ]" := nil.
Notation "[ x ; .. ; y ]" := (cons x .. (cons y []) ..).
Notation "x ++ y" := (app x y)
                     (at level 60, right associativity).
```

### 证明模式：

https://www.cs.cornell.edu/courses/cs3110/2018sp/a5/coq-tactics-cheatsheet.html#induction

- intro H：将条件成立的定理的条件移入到局部的假设，假设命名为H。intros：反复应用intro，自动取假设的名字。
- Symmetry 交换等式左右两边。
- Apply H: H是一个条件成立的定理，且结果和待证明目标相同。想要应用H，则需要满足H的条件。接下来只需要证明H的条件，即可证明待证明的目标。
- Exact HA: 待证明目标就是假设HA。不需要证明/直接完成证明。
- Assumption: 当前需要证明的目标，直接可以由已有的假设得到。尝试在上下文中查找完全匹配目标的前提 H。 如果找到了，那么其行为与 apply H 相同。
- assert：引入一个子目标，先证明它然后把它加入假设。例如：`assert (H : n * m * n = n * n * m). { rewrite mul_comm. apply mult_assoc. }`
- Qed.: 检查当前的证明并保存。
- rewrite -> H ：基于相等的某个前提，把等式一边的东西改写成另外一边的东西。右箭头表示从左往右改写
- Destruct n as [| n'] eqn:E  : 根据结构体结构解构，引入多个子目标。这里方括号内用|分割，是给每个构造函数的参数取的变量名字。然后后面用标号依次证明每个子目标。减号和加号 星号，或者它们的重复都可以。也可以花括号。
  - 这里的eqn，表示用E表示引入的等价关系。比如这里将n解构为`0`或者`S n1`，这里E就分别代表`n = 0`或者 `n = S n1`。
- unfold 展开定义。有时候定义会阻止simpl.简化。
- induction：归纳法，证明基础的，证明传递保持的。和destruct类似，引入多个子目标。
- injection：利用构造子的单射性，比如把`S n = S m`的两边都移除构造子，`injection H as Hnm`，得到假设`Hnm` 即`n = m`。
- discriminate 理由构造子的不相交性，对一个假设说这个是不可能成立的。因此直接停止证明。因为错误的假设可以推出任何东西。
- transitivity 连续的应用等式，利用等式的传递性。
- f_equal 可以将结论外面相同的函数脱去。
  - 如果每个参数依次等价，则相同形式的构造器/函数等价。对一个`f a1 ... an = g b1 ... bn`的场景应用该定理，得到多个子目标：`[f = g], [a1 = b1], ..., [an = bn]`。
- inversion 和上面类型，但是应用在假设上，把假设里相同的构造器脱去。
- specialize 将一个通用的假设，用一个情况特化。比如假设`H : forall m : nat, m * n = 0` 然后执行`specialize H with (m := 1).`，这个假设变成`H : 1 * n = 0`。可以specialize一个外面的定理进来，得到新的假设，控制apply的工作方式。
- generalize dependent 可以将变量重新放回forall的量词。在想要给量词顺序换位置的时候有用。

直接使用这些Tactics都是反向证明，使用`apply xx in H`这种带in的Tactics则可以进行前向证明。即，不断变换前提条件，最后直接在某个假设中得到结论，Apply一下即可完成证明。

- clear H：从上下文中删除前提 H。
- subst x：对于变量 x，在上下文中查找假设 x = e 或 e = x， 将整个上下文和当前目标中的所有 x 替换为 e 并清除该假设。
- subst：替换掉'所有'形如 x = e 或 e = x 的假设（其中 x 为变量）。
- rename... into...：更改证明上下文中前提的名字。例如， 如果上下文中包含名为 x 的变量，那么 rename x into y 就会将所有出现的 x 重命名为 y。
- contradiction：尝试在当前上下文中查找逻辑等价于 False 的前提 H。 如果找到了，就解决该目标。
- constructor：尝试在当前环境中的 Inductive 定义中查找可用于解决当前目标的构造子 c。如果找到了，那么其行为与 apply c 相同。

### 连接不同的证明：

- `T(1) ; T(2) (read T(1) then T(2))` 先应用T1，出现了多个需要证明的子目标时，对所有子目标应用T2
- `T; [T(1) | T(2) | ... | T(n)]` 先应用T，出现了多个需要证明的子目标时，对每个子目标按顺序应用T1 T2 ... Tn

