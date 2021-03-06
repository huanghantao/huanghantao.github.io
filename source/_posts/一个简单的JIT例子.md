---
title: 一个简单的JIT例子
date: 2020-10-12 22:12:17
tags:
- JIT
- 编译原理
---

> 我的处理器是`x86-64`，操作系统是`Mac OSX`：

这篇文章，会通过一个非常简单的例子，来讲解一下`JIT`的意思。

首先，假设我们有如下`PHP`代码：

```php
<?php

function foo(int $a, int $b)
{
    $ret = $a + $b;
    $ret += $a - $b;

    return $ret;
}

$ret = 0;

$ret = $ret + foo(1, 2);
echo $ret;
```

代码很简单，就是调用函数`foo`，然后打印结果。执行后，结果是`2`。

现在，我们来写一个简单解释器，省去解析`PHP`代码的部分，直接来生成`opcode`：

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>

typedef int (*func_ptr)(int, int);

typedef struct _zend_op_array zend_op_array;
typedef struct _zend_op zend_op;

struct _zend_op {
    int op1;
    int op2;
    char opcode;
    func_ptr handler;
};

struct _zend_op_array
{
    zend_op *opcodes;
};

int add_func(int a, int b)
{
    return a + b;
}

int div_func(int a, int b)
{
    return a - b;
}

int main(int argc, char const *argv[])
{
    // generate opcode
    zend_op_array op_array;

    op_array.opcodes = malloc(2 * sizeof(zend_op));

    int arg1 = 1;
    int arg2 = 2;

    op_array.opcodes[0].op1 = arg1;
    op_array.opcodes[0].op2 = arg2;
    op_array.opcodes[0].opcode = '+';
    op_array.opcodes[0].handler = add_func;

    op_array.opcodes[1].op1 = arg1;
    op_array.opcodes[1].op2 = arg2;
    op_array.opcodes[1].opcode = '-';
    op_array.opcodes[1].handler = div_func;

    // execute opcode
    int ret = 0;

    ret += op_array.opcodes[0].handler(op_array.opcodes[0].op1, op_array.opcodes[0].op2);
    ret += op_array.opcodes[1].handler(op_array.opcodes[1].op1, op_array.opcodes[1].op2);
    printf("%d\n", ret);
}
```

编译后，运行结果如下：

```bash
gcc jit.c
./a.out
2
```

其中，生成`opcode`的过程是一次性的。真正耗时间的是执行`opcode`的部分。参考`PHP`内核的实现我们会发现，每个`opcode`都会对应一个`handler`，例如这里的`+`和`-`。而这些`handler`实际上是`C`函数，也就意味着，每执行一条`opcode`，我们就会调用一次`C`函数。换句话来说，对于我们的`PHP`脚本的那个`foo`函数来说，执行这两条语句至少需要`2`次`C`层面的函数调用（实际上可能不止`2`次，我没有去打印真正的`opcode`）。对于我们写的这份`C`代码，查看`a.out`的汇编代码，我们会发现，除去调用`main`和`printf`，可以看到还有`2`次`call`指令的调用。

那么，如果我们对这个`foo`函数进行`jit`的话，会怎么样呢？我们来看看代码：

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>

typedef int (*func_ptr)(int, int);

typedef struct _zend_op_array zend_op_array;
typedef struct _zend_op zend_op;

struct _zend_op {
    int op1;
    int op2;
    char opcode;
    func_ptr handler;
};

struct _zend_op_array
{
    zend_op *opcodes;
};

int add_func(int a, int b)
{
    return a + b;
}

int div_func(int a, int b)
{
    return a - b;
}

func_ptr foo;

void jit_code(zend_op_array *op_array)
{
    unsigned char code[] = {
        0x55,                   	    // push   %rbp
        0x48, 0x89, 0xe5,               // mov    %rsp,%rbp
        0x89, 0x7d, 0xfc,               // mov    %edi,-0x4(%rbp)
        0x89, 0x75, 0xf8,               // mov    %esi,-0x8(%rbp)
        0xc7, 0x45, 0xf4, 0x00, 0x00, 0x00, 0x00, 	// movl   $0x0,-0xc(%rbp)
        0x8b, 0x45, 0xfc,             	// mov    -0x4(%rbp),%eax
        0x03, 0x45, 0xf8,             	// add    -0x8(%rbp),%eax
        0x89, 0x45, 0xf4,             	// mov    %eax,-0xc(%rbp)
        0x8b, 0x45, 0xfc,             	// mov    -0x4(%rbp),%eax
        0x2b, 0x45, 0xf8,             	// sub    -0x8(%rbp),%eax
        0x03, 0x45, 0xf4,             	// add    -0xc(%rbp),%eax
        0x89, 0x45, 0xf4,             	// mov    %eax,-0xc(%rbp)
        0x8b, 0x45, 0xf4,             	// mov    -0xc(%rbp),%eax
        0x5d,                   	    // pop    %rbp
        0xc3,                   	    // retq
    };

    void *mem = mmap(NULL, sizeof(code), PROT_WRITE | PROT_EXEC,
                MAP_ANON | MAP_PRIVATE, -1, 0);
    memcpy(mem, code, sizeof(code));
    foo = mem;
}

int main(int argc, char const *argv[])
{
    // generate opcode
    zend_op_array op_array;

    op_array.opcodes = malloc(2 * sizeof(zend_op));

    int arg1 = 1;
    int arg2 = 2;

    op_array.opcodes[0].op1 = arg1;
    op_array.opcodes[0].op2 = arg2;
    op_array.opcodes[0].opcode = '+';
    op_array.opcodes[0].handler = add_func;

    op_array.opcodes[1].op1 = arg1;
    op_array.opcodes[1].op2 = arg2;
    op_array.opcodes[1].opcode = '-';
    op_array.opcodes[1].handler = div_func;

    // execute opcode
    int ret = 0;

    jit_code(&op_array);

    ret = foo(arg1, arg2);
    printf("%d\n", ret);
}
```

编译后执行结果如下：

```bash
gcc jit.c
./a.out

2
```

查看汇编代码我们会发现，汇编代码里面并没有`foo`函数，但是，因为解释器在运行脚本的过程中 ，我们已经把PHP脚本的foo函数编译成了对应的二进制代码，并且放在内存里面了。这种感觉就像是我们在用`C`代码。

所以，实际上，JIT是在对我们的opcode产生的指令进行精简，减少CPU指令的条数，从而达到速度的提升（我们一定要纠正一个误区就是“没有JIT前不是跑二进制，JIT后才是跑二进制”。实际上，无论JIT不JIT，都是跑二进制，只不过跑的CPU指令会有一些变化，也就是指令被优化了）。然而，JIT要完成的工作并不仅仅是我们这个例子那么简单，所以对应的优化也并不仅仅是省去一些函数调用。

那么，从这个例子，我们也可以看出，我们在JIT的时候，是需要消耗一部分时间去把opcode转化为二进制代码。如果后期执行这些精简后的二进制代码节约的时间，远远大于把opcode翻译成二进制代码的时间，那么收益是很明显的。但是，如果后续没怎么跑我们翻译好的二进制代码，那么，像PHP这种FPM模型，一个请求就JIT一次的话，性能反而会下降（然而，这种问题PHP官方肯定也想得到，就好比opcache缓存opcode一样）。除了这种函数调用之类的优化还有其他水很深的问题，不是想JIT就JIT，例如灵剑大佬指出的方方面面，这些问题我们后面会慢慢的去探索，所以，我们会看到，JIT是可以指定JIT哪一部分的。

像我们这个例子，我们是以函数为单位来进行`JIT`的，那如果我们函数的逻辑复杂一点的话，我们理论上甚至可以对某些路径进行`JIT`。

后续，我们会使用`LLVM`来完成`yaphp`的`JIT`工作。
