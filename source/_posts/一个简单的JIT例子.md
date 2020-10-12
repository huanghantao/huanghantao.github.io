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
    int arg2 = 1;

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

其中，生成`opcode`的过程是一次性的。真正耗时间的是执行`opcode`的部分。参考`PHP`内核的首先我们会发现，每个`opcode`都会对应一个`handler`，例如这里的`+`和`-`。而这些`handler`实际上是`C`函数，也就意味着，没执行一条`opcode`，我们就会调用一次`C`函数。换句话来说，对于我们的`PHP`脚本的那个`foo`函数来说，执行这两条语句至少需要`2`次`C`层面的函数调用（实际上可能不止`2`次，我没有去打印真正的`opcode`）。对于我们写的这份`C`代码，查看`a.out`的汇编代码，我们会发现，除去调用`main`和`printf`，可以看到还有`2`次`call`指令的调用。

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
    int arg2 = 1;

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

查看汇编代码我们会发现，这个`PHP`脚本的`foo`函数，在二进制文件里面，会被编译为对应的`foo`函数。这种感觉就像是我们在用`C`代码。

所以，实际上，`JIT`是在对我们的`opcode`产生的指令进行精简，减少`CPU`指令的条数，从而达到速度的提升。

那么，从这个例子，我们也可以看出，我们在`JIT`的时候，是需要消耗一部分时间去把`opcode`转化为二进制代码。如果后期执行这些精简后的二进制代码节约的时间，远远大于把`opcode`翻译成二进制代码的时间，那么收益是很明显的。但是，如果后续没怎么跑我们翻译好的二进制代码，那么，像`PHP`这种`FPM`模型，一个请求就`JIT`一次的话，性能反而会下降。所以我们也会看到，`JIT`是可以指定`JIT`哪一部分的。

像我们这个例子，我们是以函数为单位来进行`JIT`的，那如果我们函数的逻辑复杂一点的话，我们理论上甚至可以对某些路径进行`JIT`。

后续，我们会使用`LLVM`来完成`yaphp`的`JIT`工作。
