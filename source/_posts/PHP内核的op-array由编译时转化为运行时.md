---
title: PHP内核的op_array由编译时转化为运行时
date: 2020-10-31 23:00:46
tags:
- PHP
- PHP内核
---

`PHP`内核在`pass_two`这个函数里面，会对`op_array`进行一个编译时到运行时的转化，其中一个就是重新分配`literals`。让`literals`和`opcodes`由原来分散存储的内存合并为连续的一块内存。这么做的其中一个好处是，在执行`opline`的时候，直接通过偏移量就可以拿到对应的字面量了，不需要传递`op_array`，相当于少传递了一个参数（之前需要通过`op_array->literals`的方式来获取）。
