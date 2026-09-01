---
title: Swoole内核中和连接关闭有关的各种标志位
date: 2020-08-11 10:37:05
tags:
- Swoole
---

我们来总结一下`Swoole`内核中和连接关闭有关的各种标志位。

在`swoole::Connection`结构里面：

```cpp
struct Connection {
    //--------------------------------------------------------------
    /**
     * server is actively close the connection
     */
    uint8_t close_actively;
    uint8_t closed;
    uint8_t close_queued;
    uint8_t closing;
    uint8_t close_reset;
    uint8_t peer_closed;
    /**
     * protected connection, cannot be closed by heartbeat thread.
     */
    uint8_t protect;
    //--------------------------------------------------------------
    uint8_t close_notify;
    uint8_t close_force;
    //--------------------------------------------------------------

    /**
     * received time with last data
     */
    time_t last_time;

#ifdef SW_BUFFER_RECV_TIME
    /**
     * received time(microseconds) with last data
     */
    double last_time_usec;
#endif
};
```

其中，

`close_actively`代表服务器主动关闭了连接。

`closing`代表服务器将要调用`onClose`回调函数（但还未调用）。

`closed`代表服务器已经调用完了`onClose`回调函数。

`close_queued`代表关闭连接的事件已经在排队了，一旦服务器要发送给客户端的数据发送完了，就会关闭对应的连接。这个东西是挂在对应的`socket`的`out_buffer`上的`chunk`上面。

`close_reset`代表要暴力关闭连接，不会等待`send_buffer`的数据发送完之后关闭连接，所以这种关闭模式会产生`RST`分节。

`peer_closed`代表客户端主动关闭了连接。

`protect`用来设置客户端连接为保护状态，不被心跳线程切断。

`close_notify`心跳线程设置这个标志位，用来通知`reactor`线程关闭连接。

`close_force`当`reactor`线程从管道里面收到`SW_SERVER_EVENT_CLOSE_FORCE`类型的数据的时候，`reactor`线程会去设置这个标志位。

`last_time`代表这个连接最后一次收到数据的时间，单位是毫秒。

`last_time_usec`代表这个连接最后一次收到数据的时间，单位是微秒。

理解这些标志位，对于处理一些连接泄漏的问题，会非常的有帮助。一旦某个进程连接泄漏了，我们可以`attach`进这个进程里面，然后选一个泄漏的`Connection`，看看这些标志位哪些是不正常的，就可以大概找到泄漏的原因。
