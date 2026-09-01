---
title: Swoole的多个线程如何处理信号
date: 2020-08-18 18:02:14
tags:
- Swoole
---

在`Swoole`内核里面，有多种线程。比如说心跳线程（心跳线程我们会在未来的版本进行移除），`reactor`线程，中断检查线程等等。

那么，在信号的管理方面，`Swoole`又是怎么做的呢？我们又如下规则：

```txt
1. 如果是异常产生的信号（比如程序错误，像SIGPIPE、SIGEGV这些），则只有产生异常的线程收到并处理。
2. 如果是用pthread_kill产生的内部信号，则只有pthread_kill参数中指定的目标线程收到并处理。
3. 如果是外部使用kill命令产生的信号，通常是SIGINT、SIGHUP等job control信号，则会遍历所有线程，直到找到一个不阻塞该信号的线程，然后调用它来处理。注意只有一个线程能收到。
```

那么`Swoole`是如何实现阻塞信号的呢？它提供了一个叫做`swSignal_none`的函数：

```cpp
void swSignal_none(void) {
    sigset_t mask;
    sigfillset(&mask);
    int ret = pthread_sigmask(SIG_BLOCK, &mask, nullptr);
    if (ret < 0) {
        swSysWarn("pthread_sigmask() failed");
    }
}
```

其中，

```cpp
sigfillset(&mask);
```

表示设置所有的信号。

```cpp
int ret = pthread_sigmask(SIG_BLOCK, &mask, nullptr);
```

表示对所有的信号进行阻塞。

我们发现，`Swoole`对心跳线程、中断检查线程等线程调用了`swSignal_none`。因为`Swoole`不希望这些线程去处理信号以及被这些信号打扰。具体哪些地方调用了`swSignal_none`，感兴趣的小伙伴可以看一看源码。
