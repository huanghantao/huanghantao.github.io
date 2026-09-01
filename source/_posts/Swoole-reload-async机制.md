---
title: Swoole reload_async机制
date: 2021-03-01 17:42:28
tags:
- Swoole
---

在`Swoole`的异步`Server`里面，有一个叫做`reload_async`的配置：

```php
$serv->set([
    'max_wait_time' => 60,
    'reload_async' => true,
]);
```

这个配置是用来异步安全重启服务的。

比如，我们要重启`worker`进程，但是`worker`进程正在处理着一些事件，那么，我们就不能够让旧的`worker`进程挂掉，我们需要让旧的`worker`进程处理那些事件，然后再让旧的`worker`进程退出。但是，我们不能一直去等待旧的`worker`进程去处理事件，所以，我们可以在创建新的`worker`进程的之后，保留旧的`worker`进程一段时间，让旧的`worker`进程去处理那些事件，直到超过了`max_wait_time`设置的时间之后，让旧的`worker`进程退出。
