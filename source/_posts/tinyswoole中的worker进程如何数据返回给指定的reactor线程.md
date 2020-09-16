---
title: tinyswoole中的worker进程如何把数据返回给指定的reactor线程?
date: 2019-01-07 12:52:03
tags:
- tinyswoole
- swoole
---

我们的tinyswoole的`[1.1.0-beta]`版本看似完成了worker进程与reactor线程的通信，但是如果细心分析的话会发现，这里完成的实际上是worker进程与master进程的通信。为什么这么说呢？因为，worker进程返回给master进程的数据不是给指定的reactor线程的，很可能其他的reactor线程也会读取到worker进程返回给master进程的数据。

举个例子，假设我们的reactor线程有3个，worker进程有4个。

master线程把connfd交给reactor线程的算法是：

```c
sub_reactor = &(serv->reactor_threads[connfd % serv->reactor_num].reactor);
```

也就是connfd对reactor线程的个数取模。

reactor线程把数据交给worker进程的算法是：

```c
worker_id = tswev->fd % TSwooleG.serv->process_pool->workers_num;
```

也就是connfd对worker进程的个数取模。

OK，假设有一个connfd是1，那么，reactor线程1处理这个连接，并且把connfd1这个连接发来的数据传给worker进程1。假设还有一个connfd是5，那么reactor线程2处理这个连接，并且把connfd5这个连接发来的数据传给进程1。

可以看出，reactor线程1和reactor线程2都有可能与worker进程1进行通信，也就意味着，reactor线程1可能与connfd5对应的客户端通信，reactor线程2可能与connfd1对应的客户端进行通信。虽然这样子我们的程序还可以跑，但是似乎违背了我们最初的设计—每个reactor线程维持自己的那个connfd连接即可。所以，这个问题我们需要解决掉。