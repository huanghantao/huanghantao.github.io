---
title: 协程--ucontext实战分析
date: 2019-01-24 21:51:40
tags:
- C
- Linux
- 协程
---

小伙伴们大家好，今天，我要讲解的知识是ucontext以及与之对应的几个函数。为什么需要去学习这个知识点呢？因为这对于我们理解swoole的协程大有帮助（虽然Swoole不是基于ucontex的库，但是核心思想很类似）。至少来说，得清楚协程切换时大概做了什么事情。然后呢，会让我们的tinyswoole服务器扩展支持协程。

我将会通过gdb调试来分析它。(这里，我假设大家都使用过协程）

我的实验环境是centos。并且gdb使用了peda插件（为了实时观察寄存器、下一条指令位置、函数栈的状态）。

我的实验代码如下：

```c
#include <stdlib.h>
#include <ucontext.h>
#include <stdio.h>
#include <string.h>

#define CO_DEFAULT_STACK_SIZE 2 * 1024 * 1024

void task()
{
    printf("hello world\n");
}

int main()
{
    int i;
    char *co_stack;
    ucontext_t ctx;
    ucontext_t ctx_main;

    co_stack = (char *)malloc(CO_DEFAULT_STACK_SIZE);
    if (co_stack == NULL) {
        return -1;
    }
    memset(co_stack, 0, CO_DEFAULT_STACK_SIZE);

    getcontext(&ctx);

    ctx.uc_stack.ss_sp = co_stack;
    ctx.uc_stack.ss_size = CO_DEFAULT_STACK_SIZE;
    ctx.uc_link = &ctx_main;
    makecontext(&ctx, &task, 0);

    for (i = 0; i < 10; i++) {
        swapcontext(&ctx_main, &ctx);
        makecontext(&ctx, &task, 0);
    }

    free(co_stack);
    return 0;
}
```

大概讲解一下里面的变量以及这个程序做了什么事情吧。

`co_stack`是我们自定义的栈。`CO_DEFAULT_STACK_SIZE`是自定义栈的大小。`ctx`和`ctx_main`是协程运行时的上下文。

而这个程序的作用就是切换10次task协程。

执行它的效果如下：

```shell
hello world
hello world
hello world
hello world
hello world
hello world
hello world
hello world
hello world
hello world
```

OK，我们现在开始调试。

```shell
sh-4.2# gcc test.c -g
sh-4.2# gdb a.out 
GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-114.el7
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /root/codeDir/cCode/test/a.out...done.
gdb-peda$ 
```

我们现在main函数处打断点：

```shell
gdb-peda$ b main
```

然后运行程序：

```shell
Starting program: /root/codeDir/cCode/test/a.out 

[----------------------------------registers-----------------------------------]
RAX: 0x4006ed (<main>:	push   rbp)
RBX: 0x0 
RCX: 0x4007e0 (<__libc_csu_init>:	push   r15)
RDX: 0x7fffffffed68 --> 0x7fffffffef22 ("HOSTNAME=0bce98bd0fe9")
RSI: 0x7fffffffed58 --> 0x7fffffffef03 ("/root/codeDir/cCode/test/a.out")
RDI: 0x1 
RBP: 0x7fffffffec70 --> 0x0 
RSP: 0x7fffffffe500 --> 0x0 
RIP: 0x4006f8 (<main+11>:	mov    edi,0x200000)
R8 : 0x7ffff7dd5e80 --> 0x0 
R9 : 0x0 
R10: 0x7fffffffe7a0 --> 0x0 
R11: 0x7ffff7a302e0 (<__libc_start_main>:	push   r14)
R12: 0x4005f0 (<_start>:	xor    ebp,ebp)
R13: 0x7fffffffed50 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x206 (carry PARITY adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x4006ed <main>:	push   rbp
   0x4006ee <main+1>:	mov    rbp,rsp
   0x4006f1 <main+4>:	sub    rsp,0x770
=> 0x4006f8 <main+11>:	mov    edi,0x200000
   0x4006fd <main+16>:	call   0x4005c0 <malloc@plt>
   0x400702 <main+21>:	mov    QWORD PTR [rbp-0x10],rax
   0x400706 <main+25>:	cmp    QWORD PTR [rbp-0x10],0x0
   0x40070b <main+30>:	jne    0x400717 <main+42>
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffe500 --> 0x0 
0008| 0x7fffffffe508 --> 0x0 
0016| 0x7fffffffe510 --> 0x0 
0024| 0x7fffffffe518 --> 0x0 
0032| 0x7fffffffe520 --> 0x0 
0040| 0x7fffffffe528 --> 0x0 
0048| 0x7fffffffe530 --> 0x0 
0056| 0x7fffffffe538 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, main () at test.c:20
warning: Source file is more recent than executable.
20	    co_stack = (char *)malloc(CO_DEFAULT_STACK_SIZE);
Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
gdb-peda$ 
```

此时断点被触发，程序运行到了20行之前。此时，我们打印一下`ctx`和`ctx_main`的地址：

```shell
gdb-peda$ p &ctx
$28 = (ucontext_t *) 0x7fffffffe8b0
gdb-peda$ p &ctx_main
$29 = (ucontext_t *) 0x7fffffffe500
gdb-peda$ 
```

我们继续往下走，分配我们自定义栈的空间（执行第20行的malloc代码）：

```shell
gdb-peda$ n
[----------------------------------registers-----------------------------------]
RAX: 0x7ffff780d010 --> 0x0 
RBX: 0x0 
RCX: 0x7ffff780d010 --> 0x0 
RDX: 0x7ffff780d010 --> 0x0 
RSI: 0x201000 
RDI: 0x0 
RBP: 0x7fffffffec70 --> 0x0 
RSP: 0x7fffffffe500 --> 0x0 
RIP: 0x400706 (<main+25>:	cmp    QWORD PTR [rbp-0x10],0x0)
R8 : 0xffffffffffffffff 
R9 : 0x200000 ('')
R10: 0x22 ('"')
R11: 0x1000 
R12: 0x4005f0 (<_start>:	xor    ebp,ebp)
R13: 0x7fffffffed50 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x206 (carry PARITY adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x4006f8 <main+11>:	mov    edi,0x200000
   0x4006fd <main+16>:	call   0x4005c0 <malloc@plt>
   0x400702 <main+21>:	mov    QWORD PTR [rbp-0x10],rax
=> 0x400706 <main+25>:	cmp    QWORD PTR [rbp-0x10],0x0
   0x40070b <main+30>:	jne    0x400717 <main+42>
   0x40070d <main+32>:	mov    eax,0xffffffff
   0x400712 <main+37>:	jmp    0x4007d9 <main+236>
   0x400717 <main+42>:	mov    rax,QWORD PTR [rbp-0x10]
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffe500 --> 0x0 
0008| 0x7fffffffe508 --> 0x0 
0016| 0x7fffffffe510 --> 0x0 
0024| 0x7fffffffe518 --> 0x0 
0032| 0x7fffffffe520 --> 0x0 
0040| 0x7fffffffe528 --> 0x0 
0048| 0x7fffffffe530 --> 0x0 
0056| 0x7fffffffe538 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
21	    if (co_stack == NULL) {
gdb-peda$ 
```

我们看一下自定义栈的地址：

```shell
gdb-peda$ p co_stack 
$1 = 0x7ffff780d010 ""
gdb-peda$ 
```

地址是`0x7ffff780d010`。

我们让程序继续执行到26行之前：

```shell
gdb-peda$ u 26
[----------------------------------registers-----------------------------------]
RAX: 0x7ffff780d010 --> 0x0 
RBX: 0x0 
RCX: 0x7ffff7a0d000 --> 0x0 
RDX: 0x7ffff7a0d000 --> 0x0 
RSI: 0x0 
RDI: 0x7ffff780d010 --> 0x0 
RBP: 0x7fffffffec70 --> 0x0 
RSP: 0x7fffffffe500 --> 0x0 
RIP: 0x40072d (<main+64>:	lea    rax,[rbp-0x3c0])
R8 : 0xffffffffffffffff 
R9 : 0x200000 ('')
R10: 0x7fffffffdf60 --> 0x0 
R11: 0x7ffff7a9cf40 (<__memset_sse2>:	movd   xmm8,esi)
R12: 0x4005f0 (<_start>:	xor    ebp,ebp)
R13: 0x7fffffffed50 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x400720 <main+51>:	mov    esi,0x0
   0x400725 <main+56>:	mov    rdi,rax
   0x400728 <main+59>:	call   0x400590 <memset@plt>
=> 0x40072d <main+64>:	lea    rax,[rbp-0x3c0]
   0x400734 <main+71>:	mov    rdi,rax
   0x400737 <main+74>:	call   0x4005d0 <getcontext@plt>
   0x40073c <main+79>:	mov    rax,QWORD PTR [rbp-0x10]
   0x400740 <main+83>:	mov    QWORD PTR [rbp-0x3b0],rax
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffe500 --> 0x0 
0008| 0x7fffffffe508 --> 0x0 
0016| 0x7fffffffe510 --> 0x0 
0024| 0x7fffffffe518 --> 0x0 
0032| 0x7fffffffe520 --> 0x0 
0040| 0x7fffffffe528 --> 0x0 
0048| 0x7fffffffe530 --> 0x0 
0056| 0x7fffffffe538 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
main () at test.c:26
26	    getcontext(&ctx);
gdb-peda$ 
```

此时，我们来看一看当前线程常见的寄存器内容：

```shell
gdb-peda$ info registers 
rax            0x7ffff780d010	0x7ffff780d010
rbx            0x0	0x0
rcx            0x7ffff7a0d000	0x7ffff7a0d000
rdx            0x7ffff7a0d000	0x7ffff7a0d000
rsi            0x0	0x0
rdi            0x7ffff780d010	0x7ffff780d010
rbp            0x7fffffffec70	0x7fffffffec70
rsp            0x7fffffffe500	0x7fffffffe500
r8             0xffffffffffffffff	0xffffffffffffffff
r9             0x200000	0x200000
r10            0x7fffffffdf60	0x7fffffffdf60
r11            0x7ffff7a9cf40	0x7ffff7a9cf40
r12            0x4005f0	0x4005f0
r13            0x7fffffffed50	0x7fffffffed50
r14            0x0	0x0
r15            0x0	0x0
rip            0x40072d	0x40072d <main+64>
eflags         0x246	[ PF ZF IF ]
cs             0x33	0x33
ss             0x2b	0x2b
ds             0x0	0x0
es             0x0	0x0
fs             0x0	0x0
gs             0x0	0x0
```

然后再看看ctx这个结构体的内容：

```shell
gdb-peda$ p ctx
$2 = {
  uc_flags = 0x7ffff7a1e3b0, 
  uc_link = 0x7ffff7ff7658, 
  uc_stack = {
    ss_sp = 0x0, 
    ss_flags = 0x0, 
    ss_size = 0x0
  }, 
  uc_mcontext = {
    gregs = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x601038, 0x4005f0, 0x7fffffffed50, 0x0, 0x0, 0x0, 0x7ffff7de9d1e, 0x1, 0x0, 0x0, 0x7ffff7a1e3b0, 0x7fffffffed20, 0x7ffff7df192a, 0x1c, 0x4007e0}, 
    fpregs = 0x7fffffffed58, 
    __reserved1 = {0x1, 0x4006ed, 0x400850, 0x7ffff7deadd0, 0x0, 0x0, 0x0, 0x0}
  }, 
  uc_sigmask = {
    __val = {0xffff00001f80, 0x0 <repeats 11 times>, 0x7ffff7de45b4, 0x0, 0x7ffff7ffa1a8, 0x0}
  }, 
  __fpregs_mem = {
    cwd = 0x1, 
    swd = 0x0, 
    ftw = 0x0, 
    fop = 0x0, 
    rip = 0x0, 
    rdp = 0x0, 
    mxcsr = 0x2f2f2f2f, 
    mxcr_mask = 0x2f2f2f2f, 
    _st = {{
        significand = {0x2f2f, 0x2f2f, 0x2f2f, 0x2f2f}, 
        exponent = 0x0, 
        padding = {0x0, 0x0, 0x0}
      }, {
        significand = {0x0, 0x0, 0x0, 0x0}, 
        exponent = 0xff00, 
        padding = {0x0, 0x0, 0x0}
      }, {
        significand = {0x0, 0x0, 0x0, 0x0}, 
        exponent = 0x0, 
        padding = {0x0, 0x0, 0x0}
      }, {
        significand = {0x0, 0x0, 0x0, 0x0}, 
        exponent = 0x0, 
        padding = {0x0, 0x0, 0x0}
      }, {
        significand = {0x0, 0x0, 0x0, 0x0}, 
        exponent = 0x0, 
        padding = {0x0, 0x0, 0x0}
      }, {
        significand = {0x0, 0x0, 0x0, 0x0}, 
        exponent = 0x0, 
        padding = {0x0, 0x0, 0x0}
      }, {
        significand = {0x0, 0x0, 0x0, 0x0}, 
        exponent = 0x0, 
        padding = {0x0, 0x0, 0x0}
      }, {
        significand = {0x0, 0x0, 0x0, 0x0}, 
        exponent = 0x0, 
        padding = {0x0, 0x0, 0x0}
      }}, 
    _xmm = {{
        element = {0x0, 0x0, 0x0, 0x0}
      }, {
        element = {0x0, 0x0, 0x0, 0x0}
      }, {
        element = {0x0, 0x0, 0x0, 0x0}
      }, {
        element = {0x0, 0x0, 0x0, 0x0}
      }, {
        element = {0x0, 0x0, 0x0, 0x0}
      }, {
        element = {0x0, 0x0, 0x0, 0x0}
      }, {
        element = {0x0, 0x0, 0x0, 0x0}
      }, {
        element = {0x0, 0x0, 0x0, 0x0}
      }, {
        element = {0x0, 0x0, 0x0, 0x0}
      }, {
        element = {0xf7ffea68, 0x7fff, 0xffffebc0, 0x7fff}
      }, {
        element = {0xffffebb0, 0x7fff, 0x6562b026, 0x0}
      }, {
        element = {0xf7b94b27, 0x7fff, 0xffffffff, 0x0}
      }, {
        element = {0xf7a45b18, 0x7fff, 0x2, 0x0}
      }, {
        element = {0x0, 0x0, 0x0, 0x0}
      }, {
        element = {0x0, 0x0, 0x0, 0x0}
      }, {
        element = {0x0, 0x0, 0x0, 0x0}
      }}, 
    padding = {0x0 <repeats 14 times>, 0x1, 0x0, 0x40082d, 0x0, 0x0, 0x0, 0x0, 0x0, 0x4007e0, 0x0}
  }
}
gdb-peda$ 
```

我们此时重点关注一下`ctx`的`uc_mcontext`这个成员变量：

```shell
gdb-peda$ p ctx.uc_mcontext
$3 = {
  gregs = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x601038, 0x4005f0, 0x7fffffffed50, 0x0, 0x0, 0x0, 0x7ffff7de9d1e, 0x1, 0x0, 0x0, 0x7ffff7a1e3b0, 0x7fffffffed20, 0x7ffff7df192a, 0x1c, 0x4007e0}, 
  fpregs = 0x7fffffffed58, 
  __reserved1 = {0x1, 0x4006ed, 0x400850, 0x7ffff7deadd0, 0x0, 0x0, 0x0, 0x0}
}
gdb-peda$
```

greps里面保存了寄存器的值。

OK，我们继续往下执行第26行代码，即`getcontext(&ctx);`：

```shell
gdb-peda$ n
[----------------------------------registers-----------------------------------]
RAX: 0x0 
RBX: 0x0 
RCX: 0x7ffff7a53a31 (<getcontext+129>:	cmp    rax,0xfffffffffffff001)
RDX: 0x7fffffffe9d8 --> 0x0 
RSI: 0x0 
RDI: 0x0 
RBP: 0x7fffffffec70 --> 0x0 
RSP: 0x7fffffffe500 --> 0x0 
RIP: 0x40073c (<main+79>:	mov    rax,QWORD PTR [rbp-0x10])
R8 : 0xffffffffffffffff 
R9 : 0x200000 ('')
R10: 0x8 
R11: 0x246 
R12: 0x4005f0 (<_start>:	xor    ebp,ebp)
R13: 0x7fffffffed50 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x40072d <main+64>:	lea    rax,[rbp-0x3c0]
   0x400734 <main+71>:	mov    rdi,rax
   0x400737 <main+74>:	call   0x4005d0 <getcontext@plt>
=> 0x40073c <main+79>:	mov    rax,QWORD PTR [rbp-0x10]
   0x400740 <main+83>:	mov    QWORD PTR [rbp-0x3b0],rax
   0x400747 <main+90>:	mov    QWORD PTR [rbp-0x3a0],0x200000
   0x400752 <main+101>:	lea    rax,[rbp-0x770]
   0x400759 <main+108>:	mov    QWORD PTR [rbp-0x3b8],rax
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffe500 --> 0x0 
0008| 0x7fffffffe508 --> 0x0 
0016| 0x7fffffffe510 --> 0x0 
0024| 0x7fffffffe518 --> 0x0 
0032| 0x7fffffffe520 --> 0x0 
0040| 0x7fffffffe528 --> 0x0 
0048| 0x7fffffffe530 --> 0x0 
0056| 0x7fffffffe538 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
28	    ctx.uc_stack.ss_sp = co_stack;
gdb-peda$ 

```

此时，我们来看一看ctx.uc_mcontext.greps的内容：

```shell
gdb-peda$ p ctx.uc_mcontext
$4 = {
  gregs = {0xffffffffffffffff, 0x200000, 0x0, 0x0, 0x4005f0, 0x7fffffffed50, 0x0, 0x0, 0x7fffffffe8b0, 0x0, 0x7fffffffec70, 0x0, 0x7ffff7a0d000, 0x0, 0x7ffff7a0d000, 0x7fffffffe500, 0x40073c, 0x0, 
    0x7ffff7a1e3b0, 0x7fffffffed20, 0x7ffff7df192a, 0x1c, 0x4007e0}, 
  fpregs = 0x7fffffffea58, 
  __reserved1 = {0x1, 0x4006ed, 0x400850, 0x7ffff7deadd0, 0x0, 0x0, 0x0, 0x0}
}
gdb-peda$
```

我们结合`gdb-peda`registers区域显示的寄存器内容以及code区域显示的代码地址，可以大致判断出ctx的gregs里面保存的寄存器为：

```
0xffffffffffffffff (r8), 0x200000 (r9), 
0x0 (r10), 0x0 (r11), 
0x4005f0 (r12), 0x7fffffffed50 (r13), 0x0 (r14), 0x0 (r15), 
0x7fffffffe8b0 (rdi), 0x0 (rsi), 
0x7fffffffec70 (rbp), 0x0 (bx), 0x7ffff7a0d000 (rdx), 0x0 (rax), 0x7ffff7a0d000 (rcx), 
0x7fffffffe500 (rsp), 0x40073c (rip)
```

并且，下一条指令会是

```shell
=> 0x40073c <main+79>:	mov    rax,QWORD PTR [rbp-0x10]
```

也就是

```c
ctx.uc_stack.ss_sp = co_stack;
```

这条语句的起始汇编代码。所以，`getcontext(&ctx)`的行为会把一些寄存器的值保存到ctx这个结构体的uc_mcontext.greps里面（如果ctx里面本来就保存了寄存器的值，也会全部被更新一遍）。

OK，28、29、30这三行是赋值语句，表达的意思很清晰。

我们这里打印一下`&ctx_main`的值，同时也是`ctx.uc_link`的值：

```shell
gdb-peda$ p &ctx_main
$20 = (ucontext_t *) 0x7fffffffe500
gdb-peda$ 
```

所以我们让代码运行到31行，也就是`makecontext(&ctx, &task, 0)`之前：

```shell
gdb-peda$ u 31
[----------------------------------registers-----------------------------------]
RAX: 0x7fffffffe500 --> 0x0 
RBX: 0x0 
RCX: 0x7ffff7a53a31 (<getcontext+129>:	cmp    rax,0xfffffffffffff001)
RDX: 0x7fffffffe9d8 --> 0x0 
RSI: 0x0 
RDI: 0x0 
RBP: 0x7fffffffec70 --> 0x0 
RSP: 0x7fffffffe500 --> 0x0 
RIP: 0x400760 (<main+115>:	lea    rax,[rbp-0x3c0])
R8 : 0xffffffffffffffff 
R9 : 0x200000 ('')
R10: 0x8 
R11: 0x246 
R12: 0x4005f0 (<_start>:	xor    ebp,ebp)
R13: 0x7fffffffed50 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x400747 <main+90>:	mov    QWORD PTR [rbp-0x3a0],0x200000
   0x400752 <main+101>:	lea    rax,[rbp-0x770]
   0x400759 <main+108>:	mov    QWORD PTR [rbp-0x3b8],rax
=> 0x400760 <main+115>:	lea    rax,[rbp-0x3c0]
   0x400767 <main+122>:	mov    edx,0x0
   0x40076c <main+127>:	mov    esi,0x4006dd
   0x400771 <main+132>:	mov    rdi,rax
   0x400774 <main+135>:	mov    eax,0x0
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffe500 --> 0x0 
0008| 0x7fffffffe508 --> 0x0 
0016| 0x7fffffffe510 --> 0x0 
0024| 0x7fffffffe518 --> 0x0 
0032| 0x7fffffffe520 --> 0x0 
0040| 0x7fffffffe528 --> 0x0 
0048| 0x7fffffffe530 --> 0x0 
0056| 0x7fffffffe538 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
main () at test.c:31
31	    makecontext(&ctx, &task, 0);
gdb-peda$ 
```

（此时，ctx里面存储的寄存器值还没有变化）

我们继续执行`makecontext(&ctx, &task, 0)`：

```shell
gdb-peda$ n
[----------------------------------registers-----------------------------------]
RAX: 0x7fffffffe4c0 --> 0x7fffffffed50 --> 0x1 
RBX: 0x0 
RCX: 0x7fffffffe500 --> 0x0 
RDX: 0x0 
RSI: 0x4006dd (<task>:	push   rbp)
RDI: 0x7fffffffe8b0 --> 0x7ffff7a1e3b0 --> 0xd001200002953 
RBP: 0x7fffffffec70 --> 0x0 
RSP: 0x7fffffffe500 --> 0x0 
RIP: 0x40077e (<main+145>:	mov    DWORD PTR [rbp-0x4],0x0)
R8 : 0xffffffffffffffff 
R9 : 0x200000 ('')
R10: 0x7fffffffdf60 --> 0x0 
R11: 0x7ffff7a0cff8 --> 0x7ffff7a56010 (<__start_context>:	mov    rsp,rbx)
R12: 0x4005f0 (<_start>:	xor    ebp,ebp)
R13: 0x7fffffffed50 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x400771 <main+132>:	mov    rdi,rax
   0x400774 <main+135>:	mov    eax,0x0
   0x400779 <main+140>:	call   0x4005b0 <makecontext@plt>
=> 0x40077e <main+145>:	mov    DWORD PTR [rbp-0x4],0x0
   0x400785 <main+152>:	jmp    0x4007c2 <main+213>
   0x400787 <main+154>:	lea    rdx,[rbp-0x3c0]
   0x40078e <main+161>:	lea    rax,[rbp-0x770]
   0x400795 <main+168>:	mov    rsi,rdx
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffe500 --> 0x0 
0008| 0x7fffffffe508 --> 0x0 
0016| 0x7fffffffe510 --> 0x0 
0024| 0x7fffffffe518 --> 0x0 
0032| 0x7fffffffe520 --> 0x0 
0040| 0x7fffffffe528 --> 0x0 
0048| 0x7fffffffe530 --> 0x0 
0056| 0x7fffffffe538 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
33	    for (i = 0; i < 10; i++) {
gdb-peda$ 
```

此时，我们看一看ctx里面保存的寄存器的值：

```shell
gdb-peda$ p ctx.uc_mcontext
$6 = {
  gregs = {0xffffffffffffffff, 0x200000, 0x0, 0x0, 0x4005f0, 0x7fffffffed50, 0x0, 0x0, 0x7fffffffe8b0, 0x0, 0x7fffffffec70, 0x7ffff7a0d000, 0x7ffff7a0d000, 0x0, 0x7ffff7a0d000, 0x7ffff7a0cff8, 0x4006dd, 
    0x0, 0x7ffff7a1e3b0, 0x7fffffffed20, 0x7ffff7df192a, 0x1c, 0x4007e0}, 
  fpregs = 0x7fffffffea58, 
  __reserved1 = {0x1, 0x4006ed, 0x400850, 0x7ffff7deadd0, 0x0, 0x0, 0x0, 0x0}
}
```

对比之前执行`getcontext(&ctx);`之后的ctx，我们发现此时ctx存储的`sp、rip、bx`寄存器值发生了变化。

```shell
0x7fffffffe500 -> 0x7ffff7a0cff8 (rsp)
0x40073c -> 0x4006dd (rip)
0x0 -> 0x7ffff7a0d000 (rbx)
```

我们发现，ctx里面保存的这个sp的值和co_stack所指向的自定义的栈的地址（0x7ffff780d010）有一点接近。那么这个是如何计算出来的呢？公式如下：

```c
sp = (greg_t *) ((uintptr_t) ucp->uc_stack.ss_sp
		   + ucp->uc_stack.ss_size);
sp -= (argc > 6 ? argc - 6 : 0) + 1;
sp = (greg_t *) ((((uintptr_t) sp) & -16L) - 8);
```

我们根据这个公式来计算一下（我们转化为10进制来计算）：

```shell
ctx中保存的sp值: 0x7ffff7a0cff8 -> 140737347899384
co_stack指向的自定义栈地址: 0x7ffff780d010 -> 140737345802256
140737345802256 + 2097152 = 140737347899408
140737347899408 - 1 = 140737347899407
140737347899407 & -16 = 140737347899392
140737347899392 - 8 = 140737347899384
我们发现，得到的结果就是ctx中保存的sp值。
```

`140737347899407 & -16`的作用是使我们自定义的栈对齐。`140737347899392 - 8`的作用是预留出8字节的trampoline空间(防止相互递归的发生)。

OK，我们继续来看看ctx中rip的值`0x4006dd`，很容易可以猜出来，他就是task这个函数的地址：

```shell
gdb-peda$ p &task
$67 = (void (*)()) 0x4006dd <task>
gdb-peda$ 
```

所以，makecontext的行为之一是修改ctx中rsp、rip、rbx的值。

然后，我们来看一看co_stack自定义的栈里面有没有保存些什么内容吧。

```shell
gdb-peda$ x /48b 0x7ffff7a0cff8
0x7ffff7a0cff8:	0x10	0x60	0xa5	0xf7	0xff	0x7f	0x00	0x00
0x7ffff7a0d000:	0x00	0xe5	0xff	0xff	0xff	0x7f	0x00	0x00
0x7ffff7a0d008:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7ffff7a0d010:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7ffff7a0d018:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7ffff7a0d020:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
```

我们发现，这里面存放了一些内容：

```shell
0x7ffff75a6010
0x7fffffffe500
```

其中，`0x7ffff75a6010`的值是`__start_context`函数（这个函数是context这个库里面带的）的地址：

```shell
gdb-peda$ p &__start_context
$68 = (<text variable, no debug info> *) 0x7ffff7a56010 <__start_context>
```

而`0x7fffffffe500`的值是`ctx.uc_link`的值：

```shell
gdb-peda$ p ctx.uc_link 
$71 = (struct ucontext *) 0x7fffffffe500
gdb-peda$ 
```

也就是说，执行了makecontext函数之后，会填充我们自定义的栈，填充的内容分别是`__start_context`函数的地址和`ctx.uc_link`的值。

注意，我们的这个实验因为没有给makecontext后面传递参数，所以，`ctx.uc_link`的值是紧挨着`__start_context`来存放的，并且只有`ctx`里面的rsp和rip发生了变化，如果传递了参数并且小于等于6个，那么还是会有其他寄存器发生变化的。如果大于了6个，那么会把多出来的那些参数压栈（这个和c编译器的行为类似）。

OK，我们继续执行：

```shell
gdb-peda$ n
[----------------------------------registers-----------------------------------]
RAX: 0x7fffffffe4c0 --> 0x7fffffffed50 --> 0x1 
RBX: 0x0 
RCX: 0x7fffffffe500 --> 0x0 
RDX: 0x0 
RSI: 0x4006dd (<task>:	push   rbp)
RDI: 0x7fffffffe8b0 --> 0x7ffff7a1e3b0 --> 0xd001200002953 
RBP: 0x7fffffffec70 --> 0x0 
RSP: 0x7fffffffe500 --> 0x0 
RIP: 0x400787 (<main+154>:	lea    rdx,[rbp-0x3c0])
R8 : 0xffffffffffffffff 
R9 : 0x200000 ('')
R10: 0x7fffffffdf60 --> 0x0 
R11: 0x7ffff7a0cff8 --> 0x7ffff7a56010 (<__start_context>:	mov    rsp,rbx)
R12: 0x4005f0 (<_start>:	xor    ebp,ebp)
R13: 0x7fffffffed50 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x293 (CARRY parity ADJUST zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x400779 <main+140>:	call   0x4005b0 <makecontext@plt>
   0x40077e <main+145>:	mov    DWORD PTR [rbp-0x4],0x0
   0x400785 <main+152>:	jmp    0x4007c2 <main+213>
=> 0x400787 <main+154>:	lea    rdx,[rbp-0x3c0]
   0x40078e <main+161>:	lea    rax,[rbp-0x770]
   0x400795 <main+168>:	mov    rsi,rdx
   0x400798 <main+171>:	mov    rdi,rax
   0x40079b <main+174>:	call   0x400580 <swapcontext@plt>
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffe500 --> 0x0 
0008| 0x7fffffffe508 --> 0x0 
0016| 0x7fffffffe510 --> 0x0 
0024| 0x7fffffffe518 --> 0x0 
0032| 0x7fffffffe520 --> 0x0 
0040| 0x7fffffffe528 --> 0x0 
0048| 0x7fffffffe530 --> 0x0 
0056| 0x7fffffffe538 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
34	        swapcontext(&ctx_main, &ctx);
gdb-peda$ 
```

然后，再看一下寄存器的内容：

```shell
gdb-peda$ info registers 
rax            0x7fffffffe4c0	0x7fffffffe4c0
rbx            0x0	0x0
rcx            0x7fffffffe500	0x7fffffffe500
rdx            0x0	0x0
rsi            0x4006dd	0x4006dd
rdi            0x7fffffffe8b0	0x7fffffffe8b0
rbp            0x7fffffffec70	0x7fffffffec70
rsp            0x7fffffffe500	0x7fffffffe500
r8             0xffffffffffffffff	0xffffffffffffffff
r9             0x200000	0x200000
r10            0x7fffffffdf60	0x7fffffffdf60
r11            0x7ffff7a0cff8	0x7ffff7a0cff8
r12            0x4005f0	0x4005f0
r13            0x7fffffffed50	0x7fffffffed50
r14            0x0	0x0
r15            0x0	0x0
rip            0x400787	0x400787 <main+154>
eflags         0x293	[ CF AF SF IF ]
cs             0x33	0x33
ss             0x2b	0x2b
ds             0x0	0x0
es             0x0	0x0
fs             0x0	0x0
gs             0x0	0x0
gdb-peda$
```

再看一下`ctx_main`的内容：

```shell
gdb-peda$ p ctx_main.uc_mcontext 
$72 = {
  gregs = {0x0 <repeats 23 times>}, 
  fpregs = 0x0, 
  __reserved1 = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}
}
```

我们发现，此时的`ctx_main`里面没有保存任何寄存器的值对吧。（原因很简单，之前我们都是对ctx执行操作，没有对`ctx_main`操作过）

OK，我们接着在task函数处打一个断点：

```shell
gdb-peda$ b task
Note: breakpoint 2 also set at pc 0x4006e1.
Breakpoint 3 at 0x4006e1: file test.c, line 10.
gdb-peda$
```

OK，我们继续执行：

```shell
gdb-peda$ n
[----------------------------------registers-----------------------------------]
RAX: 0x0 
RBX: 0x7ffff7a0d000 --> 0x7fffffffe500 --> 0x0 
RCX: 0x7ffff7a0d000 --> 0x7fffffffe500 --> 0x0 
RDX: 0x7ffff7a0d000 --> 0x7fffffffe500 --> 0x0 
RSI: 0x0 
RDI: 0x7fffffffe8b0 --> 0x7ffff7a1e3b0 --> 0xd001200002953 
RBP: 0x7ffff7a0cff0 --> 0x7fffffffec70 --> 0x0 
RSP: 0x7ffff7a0cff0 --> 0x7fffffffec70 --> 0x0 
RIP: 0x4006e1 (<task+4>:	mov    edi,0x400870)
R8 : 0xffffffffffffffff 
R9 : 0x200000 ('')
R10: 0x8 
R11: 0x202 
R12: 0x4005f0 (<_start>:	xor    ebp,ebp)
R13: 0x7fffffffed50 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x4006d8 <frame_dummy+40>:	jmp    0x400650 <register_tm_clones>
   0x4006dd <task>:	push   rbp
   0x4006de <task+1>:	mov    rbp,rsp
=> 0x4006e1 <task+4>:	mov    edi,0x400870
   0x4006e6 <task+9>:	call   0x400570 <puts@plt>
   0x4006eb <task+14>:	pop    rbp
   0x4006ec <task+15>:	ret    
   0x4006ed <main>:	push   rbp
[------------------------------------stack-------------------------------------]
0000| 0x7ffff7a0cff0 --> 0x7fffffffec70 --> 0x0 
0008| 0x7ffff7a0cff8 --> 0x7ffff7a56010 (<__start_context>:	mov    rsp,rbx)
0016| 0x7ffff7a0d000 --> 0x7fffffffe500 --> 0x0 
0024| 0x7ffff7a0d008 --> 0x0 
0032| 0x7ffff7a0d010 --> 0x0 
0040| 0x7ffff7a0d018 --> 0x0 
0048| 0x7ffff7a0d020 --> 0x0 
0056| 0x7ffff7a0d028 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 2, task () at test.c:10
10	    printf("hello world\n");
gdb-peda$ 
```

此时，断点触发。

也就是说，我们执行完

```c
swapcontext(&ctx_main, &ctx);
```

这条语句之后，程序调到了task函数处执行。为什么呢？我们来看看寄存器的内容：

```shell
gdb-peda$ info registers 
rax            0x0	0x0
rbx            0x7ffff7a0d000	0x7ffff7a0d000
rcx            0x7ffff7a0d000	0x7ffff7a0d000
rdx            0x7ffff7a0d000	0x7ffff7a0d000
rsi            0x0	0x0
rdi            0x7fffffffe8b0	0x7fffffffe8b0
rbp            0x7ffff7a0cff0	0x7ffff7a0cff0
rsp            0x7ffff7a0cff0	0x7ffff7a0cff0
r8             0xffffffffffffffff	0xffffffffffffffff
r9             0x200000	0x200000
r10            0x8	0x8
r11            0x202	0x202
r12            0x4005f0	0x4005f0
r13            0x7fffffffed50	0x7fffffffed50
r14            0x0	0x0
r15            0x0	0x0
rip            0x4006e1	0x4006e1 <task+4>
eflags         0x246	[ PF ZF IF ]
cs             0x33	0x33
ss             0x2b	0x2b
ds             0x0	0x0
es             0x0	0x0
fs             0x0	0x0
gs             0x0	0x0
```

我们可以观察一下这些寄存器的值，会发现，其实就是`ctx`里面保存的寄存器值。也就是说，ctx里面的寄存器值，`mov`给了cpu里面对应的寄存器。可以看看，cpu里面的`rsp`指向的是堆中的内存，而不是栈里面的内存，其实就是我们自定义的那个`co_stack`。正是因为这个栈是通过堆来进行模拟的，所以，我们发现这个自定义的栈可以开的比较大，而且切换协程，里面的数据不会被丢失。

OK，我们继续往下执行：

```shell
gdb-peda$ n
hello world

[----------------------------------registers-----------------------------------]
RAX: 0xc ('\x0c')
RBX: 0x7ffff7a0d000 --> 0x7fffffffe500 --> 0x0 
RCX: 0x7ffff7afcfd0 (<__write_nocancel+7>:	cmp    rax,0xfffffffffffff001)
RDX: 0x7ffff7dd6a00 --> 0x0 
RSI: 0x7ffff7ff6000 ("hello world\n")
RDI: 0x1 
RBP: 0x7ffff7a0cff0 --> 0x7fffffffec70 --> 0x0 
RSP: 0x7ffff7a0cff0 --> 0x7fffffffec70 --> 0x0 
RIP: 0x4006eb (<task+14>:	pop    rbp)
R8 : 0xffffffffffffffff 
R9 : 0x0 
R10: 0x22 ('"')
R11: 0x246 
R12: 0x4005f0 (<_start>:	xor    ebp,ebp)
R13: 0x7fffffffed50 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x4006de <task+1>:	mov    rbp,rsp
   0x4006e1 <task+4>:	mov    edi,0x400870
   0x4006e6 <task+9>:	call   0x400570 <puts@plt>
=> 0x4006eb <task+14>:	pop    rbp
   0x4006ec <task+15>:	ret    
   0x4006ed <main>:	push   rbp
   0x4006ee <main+1>:	mov    rbp,rsp
   0x4006f1 <main+4>:	sub    rsp,0x770
[------------------------------------stack-------------------------------------]
0000| 0x7ffff7a0cff0 --> 0x7fffffffec70 --> 0x0 
0008| 0x7ffff7a0cff8 --> 0x7ffff7a56010 (<__start_context>:	mov    rsp,rbx)
0016| 0x7ffff7a0d000 --> 0x7fffffffe500 --> 0x0 
0024| 0x7ffff7a0d008 --> 0x0 
0032| 0x7ffff7a0d010 --> 0x0 
0040| 0x7ffff7a0d018 --> 0x0 
0048| 0x7ffff7a0d020 --> 0x0 
0056| 0x7ffff7a0d028 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
11	}
gdb-peda$ 
```

我们发现我们的终端上面打印出了字符串`hello world`。

为什么我们的程序执行完task之后可以回到34行之后，也就是`swapcontext(&ctx_main, &ctx);`之后继续执行呢？因为`ctx.uc_link`的存在，这个成员变量里面保存了一个后继上下文，在这里，就是我们的`ctx_main`。

而`swapcontext(&ctx_main, &ctx)`的行为除了可以把`ctx`里面保存的寄存器值`mov`到cpu的寄存器里面，第二个行为就是把当前cpu的寄存器值保存到`ctx_main`里面。所以，一旦`task`执行完了，那么就会去执行`ctx.uc_link`这个上下文，而这个上下文在每次执行`swapcontext`的时候，都被提前保存在了`ctx_main`里面了。所以，每次都是可以回到34行然后继续执行下去。

那为什么每次执行完`swapcontext`之后，需要`makecontext`一下呢？因为，`ctx`里面保存的上下文被破坏了，所以需要重新`makecontext`一下。

如果理解了我这篇文章讲解的知识点，是不是发现可以对代码进行一个小小的改动呢？改动如下：

```c
#include <stdlib.h>
#include <ucontext.h>
#include <stdio.h>
#include <string.h>

#define CO_DEFAULT_STACK_SIZE 2 * 1024 * 1024

void task()
{
    printf("hello world\n");
}

int main()
{
    int i;
    char *co_stack;
    ucontext_t ctx;
    ucontext_t ctx_main;

    co_stack = (char *)malloc(CO_DEFAULT_STACK_SIZE);
    if (co_stack == NULL) {
        return -1;
    }
    memset(co_stack, 0, CO_DEFAULT_STACK_SIZE);

    getcontext(&ctx);

    ctx.uc_stack.ss_sp = co_stack;
    ctx.uc_stack.ss_size = CO_DEFAULT_STACK_SIZE;
    ctx.uc_link = &ctx_main;

    for (i = 0; i < 10; i++) {
        makecontext(&ctx, &task, 0);
        swapcontext(&ctx_main, &ctx);
    }

    free(co_stack);
    return 0;
}
```

我们编译运行一下：

```shell
sh-4.2# gcc test.c -g
sh-4.2# ./a.out 
hello world
hello world
hello world
hello world
hello world
hello world
hello world
hello world
hello world
hello world
sh-4.2#
```

也就是说，我只需要保证每次`swapcontext`的时候，`ctx`里面的上下文是正常的，就可以啦。

（本文完结）























