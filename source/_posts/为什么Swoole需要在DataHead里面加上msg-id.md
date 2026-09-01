---
title: 为什么Swoole需要在DataHead里面加上msg_id
date: 2021-04-25 21:16:27
tags:
- Swoole
---

在`Swoole`最近的一次`PR`[#4163](https://github.com/swoole/swoole-src/pull/4163)中，修复了一个`bug`，这个`bug`发生的概率比较低，但是还是有发生的可能性的。

先简单描述一下`bug`是什么吧。

在`Swoole`的`Process`模式下，`master`会通过管道发送数据给`worker`进程处理，当发送的数据（我们叫做`EventData`）过大的时候，就会把`EventData`拆分成一个一个的`chunk`。假设，`master`进程一共要发送`10`个`chunk`给`worker`进程，`worker`进程在接收到第`5`个`chunk`之后挂了，那么还有`5`个`chunk`就会积压在管道里面。此时，如果一个新的`worker`进程被创建，那么就会去读取管道里面残留的`5`个`chunk`，但是，这剩下的`5`个`chunk`是不完整的，不足以还原成原来的`EventData`，所以，在后续`worker`进程组建`chunk`的时候，得到的数据是不完整的，这就会导致一些内存问题。

所以，在这里，我们为每一个`EventData`编号了，如果发现这个`EventData`是上一个进程的，那么我们会丢弃这个`EventData`剩下的所有`chunk`。那么怎么判断`EventData`是不是上一个进程的呢？也很简单，让每个`msg_id`对应着一块`buffer`，当`worker`进程通过`msg_id`查找不到`buffer`的时候，就说明这个`EventData`进程是之前某个挂了的进程接收过的（因为同一个`EventData`不会被多个进程接收，所以可以通过`msg_id`来判断）。

代码不复杂，具体的实现可以看`PR`。
