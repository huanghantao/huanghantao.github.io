---
title: 浅谈Reactor设计模式
date: 2018-12-11 16:02:03
tags:
- 设计模式
---

顾名思义，Reactor是一种事件发生时候做出某种**反应**的机制。我们先来说一说概念，Reactor设计模式至少应该具备什么东西：

- 事件源（Handle）
- 事件处理器 （Event Handler）
- 反应器 （Reactor）
- 同步事件分离器（Synchronous Event Demultiplexer）

我们一个一个来解释一下。

**事件源**

可以产生事件的东西。例如文件，可以产生可读/可写事件。

**事件处理器**

当事件发生的时候，用来处理事件的函数。

**反应器**

注册/移除事件处理器；执行事件分发器；调度合适的事件处理器。

**同步事件分离器**

分离（区分）不同handle上发生的事件。我们可以通过操作系统为我们提供的某些接口（例如epoll）来实现同步事件分离器。

从上面的介绍可以看出，整个Reactor围绕的都是事件，而不是事件源。这样做的好处是可以让我们从众多的事件源中解放出来。这里贴两段代码对比一下：

**没有使用Reactor的代码**

```c
for (;;) {
    int nfds;

    nfds = epoll_wait(epollfd, events, MAXEVENTS, -1);
    for (int i = 0; i < nfds; i++) {
        if (events[i].data.fd == listenfd) {
            int connfd;
            tswEvent event;
            event.fd = events[i].data.fd;
            connfd = tswServer_master_onAccept(epollfd, &event);
            if (serv->onConnect != NULL) {
                serv->onConnect(connfd);
            }
            continue;
        }
        if (events[i].events & EPOLLIN) {
            int n;

            n = read(events[i].data.fd, buffer, MAX_BUF_SIZE);
            if (n == 0) {
                epoll_del(epollfd, events[i].data.fd);
                close(events[i].data.fd);
                continue;
            }
            buffer[n] = 0;
            if (serv->onReceive != NULL) {
                serv->onReceive(serv, events[i].data.fd, buffer);
            }
            continue;
        }
    }
}
```

使用了Reactor的代码：

```c
for (;;) {
    int nfds;

    nfds = reactor->wait(reactor);

    for (int i = 0; i < nfds; i++) {
        int connfd;
        tswReactorEpoll *reactor_epoll_object = reactor->object;

        tswEvent *tswev = (tswEvent *)reactor_epoll_object->events[i].data.ptr;
        if (tswev->event_handler(reactor, tswev) < 0) {
            tswWarn("event_handler error");
            continue;
        }
    }
}
```

可以发现，使用了Reactor的代码看起来变干净了许多。为什么会这样呢？我个人认为，第一种写法我们关注的点是复杂多变的事件源，例如，我们先判断了一下这个fd是不是listenfd，如果是的话，我们执行accept，如果不是的话，我们进行其他处理。因为第一段代码还算是业务简单，要么是accept一个客户端连接，要么是处理客户端发来的请求，但是如果业务变复杂了，应该是有点痛苦的。而第二种写法我们关注的点是**事件**，我们只需要调用处理事件的处理器（函数）即可。我不管你是什么类型的事件源，统一调用事件处理器。

为什么我这里要特别说一说Reactor呢？因为这会有助于我们理解Swoole。后续，我会介绍如何实现Reactor设计模式。同学们也可以提前学习，[仓库地址](https://github.com/huanghantao/tinyswoole)。也欢迎大家一起来贡献代码，一同学习如何开发服务器扩展。（不用担心没有基础，勇敢的迈出第一步）











