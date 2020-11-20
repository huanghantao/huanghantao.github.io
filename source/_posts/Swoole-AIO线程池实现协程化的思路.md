---
title: Swoole AIO线程池实现协程化的思路
date: 2020-11-20 15:13:26
tags:
- PHP
- Swoole
- 协程
---

`Swoole`在实现一些不好`Hook`的函数的时候，采用了`AIO`线程池来完成协程化的工作。

它的基本工作思路如下：

```bash
    ┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐                                               
    │                                                       main thread                                                        │                                               
    │                                                                                                                          │                                               
    │                                                                                                                          │                                               
    │      ┌─────────┐                                      ┌─────────┐                                       ┌─────────┐      │                                               
    │      │         │                                      │         │                                       │         │      │◀─────────────────────────┐                    
    │      │  co 1   │                                      │  co 2   │                                       │  co n   │      │                          │                    
    │      │         │                                      │         │                                       │         │      │                          │                    
    │      └─────────┘                                      └─────────┘                                       └─────────┘      │                          │                    
    │           │                                                │                                                 │           │       receive event and resume the coroutine  
    └───────────┼────────────────────────────────────────────────┼─────────────────────────────────────────────────┼───────────┘                          │                    
                │                                                │                                                 │                                      │                    
    dispatch task to pool::queue,                    dispatch task to pool::queue,                     dispatch task to pool::queue,                      │                    
             then yield                                       then yield                                        then yield                                │                    
                │                                                │                                                 │                                      │                    
                │                                                │                                                 │                                      │                    
                └────────────────────────────────────────────────┼─────────────────────────────────────────────────┘                                      │                    
                                                                 │                                                                                        │                    
                                                                 │                                                                                        │                    
                                                                 │                                                                             ┌────────────────────┐          
                                                                 │                                                                             │                    │          
                                                                 │                                                                             │    unix socket     │          
                                                                 │                                                                             │                    │          
                                                                 ▼                                                                             │                    │          
                                            ┌────────────────────────────────────────┐                                                         └────────────────────┘          
                                            │                                        │                                                                    ▲                    
                                            │           ThreadPool::queue            │                                                                    │                    
                                            │                                        │                                                                    │                    
                                            └────────────────────────────────────────┘                                                                    │                    
                                                                 │                                                                                        │                    
                                                                 │                                                                                        │                    
                                                                 │                                                                                        │                    
                                                                 │                                                                                        │                    
                                                                 │                                                                                        │                    
                ┌───────────────────────────────┬────────────────┴──────────────┬────────────────────────────────┐                                        │                    
                │                               │                               │                                │                                        │                    
                │                               │                               │                                │                                        │                    
    pop task from pool::queue       pop task from pool::queue       pop task from pool::queue        pop task from pool::queue                            │                    
                │                               │                               │                                │                                        │                    
                │                               │                               │                                │                                        │                    
                ▼                               ▼                               ▼                                ▼                                        │                    
    ┌───────────────────────┐       ┌───────────────────────┐       ┌───────────────────────┐        ┌───────────────────────┐                            │                    
    │                       │       │                       │       │                       │        │                       │                            │                    
    │     AIO thread 1      │       │     AIO thread 2      │       │     AIO thread 3      │        │     AIO thread n      │                            │                    
    │                       │       │                       │       │                       │        │                       │                            │                    
    │                       │       │                       │       │                       │        │                       │                            │                    
    └───────────────────────┘       └───────────────────────┘       └───────────────────────┘        └───────────────────────┘                            │                    
                │                               │                               │                                │                                        │                    
           send event                      send event                      send event                       send event                                    │                    
                └───────────────────────────────┴───────────────────────────────┴────────────────────────────────┴────────────────────────────────────────┘                    
```

大概讲一讲这个流程：

1.当一个协程执行一个不好协程化的任务的时候，就会创建一个任务，投递到线程池的`queue`里面，对应代码：

```cpp
AsyncEvent *dispatch(const AsyncEvent *request) {
     if (SwooleTG.aio_schedule) {
          schedule();
     }
     auto _event_copy = new AsyncEvent(*request);
     _event_copy->task_id = current_task_id++;
     _event_copy->timestamp = swoole_microtime();
     _event_copy->pipe_socket = SwooleTG.aio_write_socket;
     event_mutex.lock();
     _queue.push(_event_copy);
     _cv.notify_one();
     event_mutex.unlock();
     swDebug("push and notify one: %f", swoole_microtime());
     return _event_copy;
}
```

2.投递完任务之后，挂起当前协程：

```cpp
bool async(const std::function<void(void)> &fn, double timeout) {
     TimerNode *timer = nullptr;
     AsyncEvent event{};
     AsyncLambdaTask task{Coroutine::get_current_safe(), fn};

     event.object = &task;
     event.handler = async_lambda_handler;
     event.callback = async_lambda_callback;

     AsyncEvent *_ev = async::dispatch(&event);
     if (_ev == nullptr) {
          return false;
     }
     if (timeout > 0) {
          timer = swoole_timer_add((long) (timeout * 1000), false, async_task_timeout, _ev);
     }
     task.co->yield();
     // 省略其他代码
}
```

其他，`async_lambda_handler`会被`AIO`线程使用，`async_lambda_callback`被主线程的调度器使用。

3.当`AIO`线程抢到一个任务的时候，会调用`async_lambda_handler`，而`async_lambda_handler`就会去执行协程投递任务时设置的那个不好协程化的函数。

4.`AIO`线程执行完任务之后，通过`unix socket`通知主线程。此时，主线程就会执行`async_lambda_callback`，这个函数会`resume`这个任务对应的协程。然后，该协程继续往下运行。

这种方式很好用，但是，我们使用这种方式协程化的时候，需要注意一个问题，不要在任务里面去调用`PHP`的函数，因为这样就会让`AIO`线程操作`ZendVM`。因为主线程和`AIO`线程同时在修改同一个`ZendVM`上的数据，会导致一些内存错误。
