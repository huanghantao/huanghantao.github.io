---
title: Swoole 协程Channel实现原理
date: 2019-09-20 14:15:52
tags:
- Swoole
- PHP
- 协程
---

为了给我们的扩展实现`Channel`做准备，这里，很有必要简单分析一下`Swoole`协程的`Channel`实现原理。核心点如下：

```shell
1、什么情况下可以pop
2、当channel不可以pop的时候，应该如何处理消费者协程
3、什么情况下可以push
4、当channel不可以push的时候，应该如何处理生产者协程
5、当channel可以pop的时候，应该如何通知消费者协程
6、当channel可以push的时候，应该如何通知生产者协程
```

如果解决了这些问题，就可以去实现`Channel`了。

## 什么情况下可以pop

这个问题很简单，当`channel`里面有数据的时候。我们来看看`Swoole`对应的源码，在方法`Channel::pop(double timeout)`里面：

```cpp
void *data = data_queue.front();
data_queue.pop();
```

## 消费者协程和生产者协程

我们定义一个协程是消费者协程还是生产者协程，取决于这个协程**正在**执行哪种操作。如果这个协程**此时**正在执行`pop`操作，那么这个协程**此时**就是消费者协程；如果这个协程**此时**正在执行`push`操作，那么这个协程**此时**就是生产者协程。也就是说，协程是生产者还是消费者不是死的，是会随着协程的操作动态变化的。

## 当channel不可以pop的时候，应该如何处理消费者协程

当`channel`不可以`pop`的时候，我们应该挂起这个消费者协程。我们来看看`Swoole`对应的代码：

```c++
if (is_empty() || !consumer_queue.empty())
{
    // 省略其他代码
    yield(CONSUMER);
    // 省略其他代码
}
```

我们可以看到，当`Channel`为空的时候，消费者协程是不可以进行`pop`操作的，此时被`yield`出去了。

## 什么情况下可以push

这个问题很简单，当`channel`容器没有满的时候。我们来看看`Swoole`对应的源码，在方法`Channel::push(void *data, double timeout)`里面：

```cpp
data_queue.push(data);
swTraceLog(SW_TRACE_CHANNEL, "push data to channel, count=%ld", length());
```

## 当channel不可以push的时候，应该如何处理生产者协程

当`channel`不可以`push`的时候，我们应该挂起这个生产者协程。我们来看看`Swoole`对应的代码：

```c++
if (is_full() || !producer_queue.empty())
{
    // 省略其他的代码
    yield(PRODUCER);
    // 省略其他的代码
}
```

我们可以看到，当`Channel`满了的时候，生产者协程是不可以进行`push`操作的，此时被`yield`出去了。

## 当channel可以pop的时候，应该如何通知消费者协程

首先，我们这里需要明白，是谁在通知消费者协程。是`Swoole`的那套事件循环吗？不是的。

通知消费这协程的是生产者协程。我们来看看`pop`的逆操作`push`的代码：

```cpp
/**
 * push data
 */
data_queue.push(data);
swTraceLog(SW_TRACE_CHANNEL, "push data to channel, count=%ld", length());
/**
 * notify consumer
 */
if (!consumer_queue.empty())
{
    Coroutine *co = pop_coroutine(CONSUMER);
    co->resume();
}
```

可以看到，当生产者协程`push`完数据的时候，此时`channel`必定是有数据的。然后，如果有消费者协程在等待`channel`的话，那么就唤醒第一个等待的那个消费者协程。（所以规则是：谁先等待`channel`，谁就先被唤醒）

## 当channel可以push的时候，应该如何通知生产者协程

这个的原理和上一个问题的答案类似，不重复说了。一句话：**消费者通知的生产者协程**。
