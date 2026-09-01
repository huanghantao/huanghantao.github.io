---
title: Swoole Server中master进程投递数据到worker进程的性能优化
date: 2020-01-22 10:43:46
tags:
- Swoole
---

在`Swoole4.5`版本中（目前还未发布），我们的`Server`有一个性能需要优化的地方，就是`worker`进程在收到`master`进程发来的包的时候，需要进行两次的拷贝，才可以把数据从`PHP`扩展层传递到`PHP`上层（也就是我们事件回调函数的`data`参数）。

我们先来分析一下为什么会有性能的问题。首先，我们需要一份会有性能问题的代码。我们`git clone`下`swoole-src`代码，然后`git checkout`到`8235c82fea2130534a16fd20771dcab3408a763e`这个`commit`位置：

```shell
git checkout 8235c82fea2130534a16fd20771dcab3408a763e
```

我们来分析一下代码，首先看`master`进程是如何封装数据然后发送给`worker`进程的。在函数`process_send_packet`里面，我们看核心的地方：

```cpp
static int process_send_packet(swServer *serv, swPipeBuffer *buf, swSendData *resp, send_func_t _send, void* private_data)
{
    const char* data = resp->data;
    uint32_t send_n = resp->info.len;
    off_t offset = 0;

    uint32_t max_length = serv->ipc_max_size - sizeof(buf->info);

    if (send_n <= max_length)
    {
        buf->info.flags = 0;
        buf->info.len = send_n;
        memcpy(buf->data, data, send_n);

        int retval = _send(serv, buf, sizeof(buf->info) + send_n, private_data);
        return retval;
    }

    buf->info.flags = SW_EVENT_DATA_CHUNK;

    while (send_n > 0)
    {
        if (send_n > max_length)
        {
            buf->info.len = max_length;
        }
        else
        {
            buf->info.flags |= SW_EVENT_DATA_END;
            buf->info.len = send_n;
        }

        memcpy(buf->data, data + offset, buf->info.len);

        if (_send(serv, buf, sizeof(buf->info) + buf->info.len, private_data) < 0)
        {
            return SW_ERR;
        }

        send_n -= buf->info.len;
        offset += buf->info.len;
    }

    return SW_OK;
}
```

首先，我们来说一下`process_send_packet`这个函数的参数：

其中，

`swServer *serv`就是我们创建的那个`Server`。

`swPipeBuffer *buf`指向的内存里面的数据需要发送给`worker`进程。

`swSendData *resp`里面存放了`master`进程收到的客户端数据以及一个`swDataHead info`头部。

`_send`是一个回调函数，这里面的逻辑就是`master`进程把`swPipeBuffer *buf`里面的数据发送给`worker`进程。

`void* private_data`这里是一个`swWorker *worker`类型的指针转换过来的。指定了`master`进程需要发送的那个`worker`进程。

> 说明一点，这里我们是以`Server`设置了`eof`选项为例子讲解的（假设设置了`"\r\n"`）。因为`TCP`是面向字节流的，即使客户端发送了一个很大的包过来，服务器一次`read`出来的数据也不见得非常大。如果不设置`eof`的话，是不会导致我们这篇文章所说的性能问题。

介绍完了`process_send_packet`函数的参数之后，我们来看看代码是如何实现的：

```cpp
const char* data = resp->data;
```

首先，让`data`指向`resp->data`，也就是客户端发来的实际数据。例如，客户端发来了字符串`hello world\r\n`，那么`data`里面存放的就是`hello world\r\n`。

```cpp
uint32_t send_n = resp->info.len;
```

标志着`resp->data`数据的长度。例如，客户端往服务器发送了`1M`的数据，那么`resp->info.len`就是`1048576`。

```cpp
off_t offset = 0;
```

用来标志哪些数据`master`进程已经发送给了`worker`进程。

```cpp
uint32_t max_length = serv->ipc_max_size - sizeof(buf->info);
```

`max_length`表示`master`进程一次往`worker`进程发送的包最大长度。

> 注意：`master`进程和`worker`进程是通过`udg`方式进行通信的。所以，`master`进程发送多少，`worker`进程就直接收多少

```cpp
if (send_n <= max_length)
{
    buf->info.flags = 0;
    buf->info.len = send_n;
    memcpy(buf->data, data, send_n);

    int retval = _send(serv, buf, sizeof(buf->info) + send_n, private_data);
    return retval;
}
```

如果`master`进程要发给`worker`进程的数据小于`max_length`，那么就直接调用`_send`函数，直接把数据发给`worker`进程。

```cpp
buf->info.flags = SW_EVENT_DATA_CHUNK;
```

当`send_n`大于`max_length`的时候，设置`buf->info.flags`为`CHUNK`，也就意味着需要把客户端发来的数据先拆分成一小段一小段的数据，然后再发送给`worker`进程。

```cpp
while (send_n > 0)
{
    if (send_n > max_length)
    {
        buf->info.len = max_length;
    }
    else
    {
        buf->info.flags |= SW_EVENT_DATA_END;
        buf->info.len = send_n;
    }

    memcpy(buf->data, data + offset, buf->info.len);

    if (_send(serv, buf, sizeof(buf->info) + buf->info.len, private_data) < 0)
    {
        return SW_ERR;
    }

    send_n -= buf->info.len;
    offset += buf->info.len;
}
```

逻辑比较简单，就是一个分段发送的过程。这里需要注意的两点：

```shell
1、buf->info.len的长度需要更新为小段的chunk的长度，而不是大数据包的长度
2、最后一个chunk的info.flags需要变成SW_EVENT_DATA_END，意味着一个完整的包已经发完了
```

`OK`，分析完了`master`进程发包的过程，我们来分析一下`worker`进程收包的过程。

我们先看一下函数`swWorker_onPipeReceive`：

```cpp
static int swWorker_onPipeReceive(swReactor *reactor, swEvent *event)
{
    swServer *serv = (swServer *) reactor->ptr;
    swFactory *factory = &serv->factory;
    swPipeBuffer *buffer = serv->pipe_buffers[0];
    int ret;

    _read_from_pipe:

    if (read(event->fd, buffer, serv->ipc_max_size) > 0)
    {
        ret = swWorker_onTask(factory, (swEventData *) buffer);
        if (buffer->info.flags & SW_EVENT_DATA_CHUNK)
        {
            //no data
            if (ret < 0 && errno == EAGAIN)
            {
                return SW_OK;
            }
            else if (ret > 0)
            {
                goto _read_from_pipe;
            }
        }
        return ret;
    }

    return SW_ERR;
}
```

这个就是`worker`进程接收`master`进程发来的数据的代码。

我们看的，`worker`进程会直接把数据先读取到`buffer`内存里面，然后调用`swWorker_onTask`。我们再来看看`swWorker_onTask`函数：

```cpp
int swWorker_onTask(swFactory *factory, swEventData *task)
{
    swServer *serv = (swServer *) factory->ptr;
    swWorker *worker = SwooleWG.worker;

    //worker busy
    worker->status = SW_WORKER_BUSY;
    //packet chunk
    if (task->info.flags & SW_EVENT_DATA_CHUNK)
    {
        if (serv->merge_chunk(serv, task->info.reactor_id, task->data, task->info.len) < 0)
        {
            swoole_error_log(SW_LOG_WARNING, SW_ERROR_SESSION_DISCARD_DATA,
                    "cannot merge chunk to worker buffer, data[fd=%d, size=%d] lost", task->info.fd, task->info.len);
            return SW_OK;
        }
        //wait more data
        if (!(task->info.flags & SW_EVENT_DATA_END))
        {
            return SW_OK;
        }
    }

    switch (task->info.type)
    {
    case SW_SERVER_EVENT_SEND_DATA:
        //discard data
        if (swWorker_discard_data(serv, task) == SW_TRUE)
        {
            break;
        }
        swWorker_do_task(serv, worker, task, serv->onReceive);
        break;
    // 省略其他的case
    default:
        swWarn("[Worker] error event[type=%d]", (int )task->info.type);
        break;
    }

    //worker idle
    worker->status = SW_WORKER_IDLE;

    //maximum number of requests, process will exit.
    if (!SwooleWG.run_always && worker->request_count >= SwooleWG.max_request)
    {
        swWorker_stop(worker);
    }
    return SW_OK;
}
```

我们重点看看性能问题代码：

```cpp
if (task->info.flags & SW_EVENT_DATA_CHUNK)
{
    if (serv->merge_chunk(serv, task->info.reactor_id, task->data, task->info.len) < 0)
    {
        swoole_error_log(SW_LOG_WARNING, SW_ERROR_SESSION_DISCARD_DATA,
                "cannot merge chunk to worker buffer, data[fd=%d, size=%d] lost", task->info.fd, task->info.len);
        return SW_OK;
    }
    //wait more data
    if (!(task->info.flags & SW_EVENT_DATA_END))
    {
        return SW_OK;
    }
}
```

这里，`worker`进程会先判断`master`发来的数据是否是`CHUNK`数据，如果是，那么会进行`merge_chunk`的操作。我们看看`merge_chunk`对应的函数：

```cpp
static int swServer_worker_merge_chunk(swServer *serv, int key, const char *data, size_t len)
{
    swString *package = swServer_worker_get_input_buffer(serv, key);
    //merge data to package buffer
    return swString_append_ptr(package, data, len);
}
```

我们会先根据`key`的值（实际上是`reactor`线程的`id`），获取一块全局的内存，然后把接收到的`chunk`数据，追加到这个全局的内存上面，而`swString_append_ptr`执行的就是`memcpy`的操作。

所以，这就是一个性能问题了。`worker`进程接收到的所有数据都会被完整的拷贝一遍。如果客户端发来的数据很大，这个拷贝的开销也是很大声的。

因此，我们对这部分合并的代码进行了一个优化。我们让`worker`进程在接收`master`进程的数据之前，就准备好一块足够大的内存，然后直接把`master`进程发来的数据下来即可。

我们先更新一下`swoole-src`的源码：

```shell
git checkout 529ad44d578930b3607abedcfc278364df34bc73
```

我们依旧先看看`process_send_packet`函数的代码：

```cpp
static int process_send_packet(swServer *serv, swPipeBuffer *buf, swSendData *resp, send_func_t _send, void* private_data)
{
    const char* data = resp->data;
    uint32_t send_n = resp->info.len;
    off_t offset = 0;
    uint32_t copy_n;

    uint32_t max_length = serv->ipc_max_size - sizeof(buf->info);

    if (send_n <= max_length)
    {
        buf->info.flags = 0;
        buf->info.len = send_n;
        memcpy(buf->data, data, send_n);

        int retval = _send(serv, buf, sizeof(buf->info) + send_n, private_data);
        return retval;
    }

    buf->info.flags = SW_EVENT_DATA_CHUNK;
    buf->info.len = send_n;

    while (send_n > 0)
    {
        if (send_n > max_length)
        {
            copy_n = max_length;
        }
        else
        {
            buf->info.flags |= SW_EVENT_DATA_END;
            copy_n = send_n;
        }

        memcpy(buf->data, data + offset, copy_n);

        swTrace("finish, type=%d|len=%d", buf->info.type, copy_n);

        if (_send(serv, buf, sizeof(buf->info) + copy_n, private_data) < 0)
        {
            return SW_ERR;
        }

        send_n -= copy_n;
        offset += copy_n;
    }

    return SW_OK;
}
```

我们聚焦修改的地方，主要是对`CHUNK`的处理：

```cpp
buf->info.flags = SW_EVENT_DATA_CHUNK;
buf->info.len = send_n;
```

我们发现，`buf->info.len`的长度不是每个小段`chunk`的长度了，而是整个大包的长度了。为什么可以这样做呢？因为`master`进程与`worker`进程是通过`udg`进行通信的，所以，`worker`进程在调用`recv`的时候，返回值实际上就是`chunk`的长度了，所以`buf->info.len`里面存储`chunk`的长度没有必要。

其他地方的逻辑和之前的代码没有区别。

我们再来看看`worker`进程是如何接收`master`进程发来的数据的。在函数`swWorker_onPipeReceive`里面：

```cpp
static int swWorker_onPipeReceive(swReactor *reactor, swEvent *event)
{
    int ret;
    ssize_t recv_n = 0;
    swServer *serv = (swServer *) reactor->ptr;
    swFactory *factory = &serv->factory;
    swPipeBuffer *pipe_buffer = serv->pipe_buffers[0];
    void *buffer;
    struct iovec buffers[2];

    // peek
    recv_n = recv(event->fd, &pipe_buffer->info, sizeof(pipe_buffer->info), MSG_PEEK);
    if (recv_n < 0 && errno == EAGAIN)
    {
        return SW_OK;
    }
    else if (recv_n < 0)
    {
        return SW_ERR;
    }

    if (pipe_buffer->info.flags & SW_EVENT_DATA_CHUNK)
    {
        buffer = serv->get_buffer(serv, &pipe_buffer->info);
        _read_from_pipe:

        buffers[0].iov_base = &pipe_buffer->info;
        buffers[0].iov_len = sizeof(pipe_buffer->info);
        buffers[1].iov_base = buffer;
        buffers[1].iov_len = serv->ipc_max_size - sizeof(pipe_buffer->info);

        recv_n = readv(event->fd, buffers, 2);
        if (recv_n < 0 && errno == EAGAIN)
        {
            return SW_OK;
        }
        if (recv_n > 0)
        {
            serv->add_buffer_len(serv, &pipe_buffer->info, recv_n - sizeof(pipe_buffer->info));
        }

        if (pipe_buffer->info.flags & SW_EVENT_DATA_CHUNK)
        {
            //wait more chunk data
            if (!(pipe_buffer->info.flags & SW_EVENT_DATA_END))
            {
                goto _read_from_pipe;
            }
            else
            {
                pipe_buffer->info.flags |= SW_EVENT_DATA_OBJ_PTR;
                /**
                 * Because we don't want to split the swEventData parameters into swDataHead and data,
                 * we store the value of the worker_buffer pointer in swEventData.data.
                 * The value of this pointer will be fetched in the swServer_worker_get_packet function.
                 */
                serv->copy_buffer_addr(serv, pipe_buffer);
            }
        }
    }
    else
    {
        recv_n = read(event->fd, pipe_buffer, serv->ipc_max_size);
    }

    if (recv_n > 0)
    {
        ret = swWorker_onTask(factory, (swEventData *) pipe_buffer, recv_n - sizeof(pipe_buffer->info));
        return ret;
    }

    return SW_ERR;
}
```

其中，

```cpp
recv_n = recv(event->fd, &pipe_buffer->info, sizeof(pipe_buffer->info), MSG_PEEK);
if (recv_n < 0 && errno == EAGAIN)
{
    return SW_OK;
}
else if (recv_n < 0)
{
    return SW_ERR;
}
```

我们先对内核缓冲区里面的数据进行一次`peek`操作，来获取到`head`部分。这样我们就知道数据是否是以`CHUNK`方式发来的了。

```cpp
if (pipe_buffer->info.flags & SW_EVENT_DATA_CHUNK)
{
    buffer = serv->get_buffer(serv, &pipe_buffer->info);
    _read_from_pipe:

    buffers[0].iov_base = &pipe_buffer->info;
    buffers[0].iov_len = sizeof(pipe_buffer->info);
    buffers[1].iov_base = buffer;
    buffers[1].iov_len = serv->ipc_max_size - sizeof(pipe_buffer->info);

    recv_n = readv(event->fd, buffers, 2);
    if (recv_n < 0 && errno == EAGAIN)
    {
        return SW_OK;
    }
    if (recv_n > 0)
    {
        serv->add_buffer_len(serv, &pipe_buffer->info, recv_n - sizeof(pipe_buffer->info));
    }

    if (pipe_buffer->info.flags & SW_EVENT_DATA_CHUNK)
    {
        //wait more chunk data
        if (!(pipe_buffer->info.flags & SW_EVENT_DATA_END))
        {
            goto _read_from_pipe;
        }
        else
        {
            pipe_buffer->info.flags |= SW_EVENT_DATA_OBJ_PTR;
            /**
                * Because we don't want to split the swEventData parameters into swDataHead and data,
                * we store the value of the worker_buffer pointer in swEventData.data.
                * The value of this pointer will be fetched in the swServer_worker_get_packet function.
                */
            serv->copy_buffer_addr(serv, pipe_buffer);
        }
    }
}
```

如果是`CHUNK`方式发来的数据，那么我们执行如下的操作：

```cpp
buffer = serv->get_buffer(serv, &pipe_buffer->info);
```

`get_buffer`是一个回调函数，对应：

```cpp
static void* swServer_worker_get_buffer(swServer *serv, swDataHead *info)
{
    swString *worker_buffer = swServer_worker_get_input_buffer(serv, info->reactor_id);

    if (worker_buffer->size < info->len)
    {
        swString_extend(worker_buffer, info->len);
    }

    return worker_buffer->str + worker_buffer->length;
}
```

这里，我们会先判断这块全局的`buffer`是否足够的大，可以接收完整个大包，如果不够大，我们扩容到足够的大。

```cpp
_read_from_pipe:

buffers[0].iov_base = &pipe_buffer->info;
buffers[0].iov_len = sizeof(pipe_buffer->info);
buffers[1].iov_base = buffer;
buffers[1].iov_len = serv->ipc_max_size - sizeof(pipe_buffer->info);

recv_n = readv(event->fd, buffers, 2);
```

然后，我们调用`readv`，把`head`和实际的数据分别存在了两个地方。这么做是避免为了把`head`和实际的数据做拆分而导致的内存拷贝。

通过以上方式，`Swoole Server`减少了一次内存拷贝。
