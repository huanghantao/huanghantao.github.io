---
title: Swoole协程栈保护
date: 2020-06-28 14:34:45
tags:
- PHP
- Swoole
---

> 本文基于的Swoole commit为：1e283dfa109fcb0887a46f3ba53bf67af021c931

我们来看一下`Swoole`分配协程栈的代码：

```cpp
Context::Context(size_t stack_size, coroutine_func_t fn, void* private_data) :
        fn_(fn), stack_size_(stack_size), private_data_(private_data)
{
    // 省略其他代码
#ifdef SW_CONTEXT_PROTECT_STACK_PAGE
    stack_ = (char*) ::mmap(0, stack_size_, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
#else
     stack_ = (char*) sw_malloc(stack_size_);
#endif
    // 省略其他代码
#ifdef SW_CONTEXT_PROTECT_STACK_PAGE
    mprotect(stack_, SwooleG.pagesize, PROT_NONE);
#endif
}
```

这里，我们发现，如果在编译`Swoole`的时候，定义了`SW_CONTEXT_PROTECT_STACK_PAGE`，即打算开启栈保护，那么就调用函数`mmap`来分配内存，否则，直接调用`malloc`来分配内存。

并且，如果打算开启栈保护，将会调用`mprotect`函数来对栈的第一页进行保护。`PROT_NONE`表明该内存空间不可访问。

> 这里，我补充一下，栈保护的意义。如果不进行保护，那么就可能会因为栈溢出攻击导致函数的返回地址被修改，从而执行一段恶意的代码。
> 
> 至于为什么要特意去保护协程栈的原因如下：当我们自己去模拟栈的时候，可能会出现访问栈越界的问题。只读操作还好，如果进行了写操作，就非常的危险了，因此，栈保护工作还是十分必要的。（如果是编译器实现的栈，编译器就会帮我们完成栈保护的工作，所以我们在编程的时候，不需要去做这种保护栈的工作）

那么，这里为啥使用`mmap`呢？因为`mmap`分配的内存是按`page`对齐的。而`mprotect`是按照页来进行设置的。因此，如果栈地址没有对齐，应该先对齐之后再去调用`mprotect`。

当然，这里也可以调用`malloc`来获取一块内存，然后从`malloc`返回的指针开始，找到对齐的那个位置，然后再调用`mprotect`，代码大概如下：

```cpp
stack = malloc(stack_size);
stack = (void *) (((uintptr_t) stack) & ~(pagesize - 1));

#ifdef SW_CONTEXT_PROTECT_STACK_PAGE
mprotect(stack + pagesize, pagesize, PROT_NONE);
#endif
```

所以，`Swoole`这里通过`mmap`来分配内存，实际上是一种简化的做法。
