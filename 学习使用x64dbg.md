---
title: 学习使用x64dbg
date: 2019/6/25 20:46:25
categories:
- CTF
tags:
- x64dbg
- 逆向
---

# 学习使用x64dbg
<!-- more -->
## 快捷键

ctrl+G  GO TO
F4 执行到光标
; 注释
: 标签
右键菜单里有搜索注释和标签
F9 Run 有断点则执行到断点
*号 显示EIP指向的位置
-号 显示上一个光标的位置
菜单中也可以查看所有断点

另外在数据查看里面，选中一段数据（字符串），可以Ctrl+E修改

## 查找指定代码
1. 慢慢执行法
2. 字符串检索法
右键-search for 然后选string
3. api检索法
同上 右键-search for 然后选All intermodular calls

## patch后保存
File - Patch File

