---
title: 《就是要你懂Swoole》--虚拟机中断
date: 2019-05-27 14:09:29
tags:
- PHP
- PHP扩展
- Swoole
---

这篇文章是讲解`PHP7`虚拟机中断的用法，理解这篇文章之后就可以更好的理解`Swoole`在CPU密集型的情况下协程的调度原理。

Swoole之前尝试了多种方法来处理CPU密集型环境下协程的切换问题。分别是Hook for循环，使用`PHP`的`declare(ticks=N)`语法。因为种种原因，`Swoole`现在使用了虚拟机中断的方法来切换一直占用`CPU`的协程。具体原因可以查看PHP最近的`Issue`以及`PR`，里面写的很清楚了。

好的，我们现在来感受一下虚拟机中断的用法。

首先，我们实现一个扩展函数，它的作用是开启对虚拟机中断的使用：

```c
PHP_FUNCTION(start_interrupt) {
	init();
	create_scheduler_thread();
};
```

init的实现如下：

```c
void init()
{
	orig_interrupt_function = zend_interrupt_function;
	zend_interrupt_function = new_interrupt_function;
}
```

这里的核心是

```c
zend_interrupt_function = new_interrupt_function;
```

也就是说，我们需要实现`zend_interrupt_function`。`zend_interrupt_function`的作用是在虚拟机中断发生的时候，会去执行的函数。并且，`zend_interrupt_function`是`PHP`内核定义的一个函数指针。

（我们这篇文章不多讲`orig_interrupt_function = zend_interrupt_function;`，我们假设没有其他扩展实现了`zend_interrupt_function`）

`new_interrupt_function`的定义如下：

```c
static void new_interrupt_function(zend_execute_data *execute_data)
{
		php_printf("yield coroutine\n");
		if (orig_interrupt_function)
    {
        orig_interrupt_function(execute_data);
    }
}
```

可以看出，我们的这个函数`'模拟'`了`yield`协程的过程。OK，我们接着看`create_scheduler_thread`：

```c
static void create_scheduler_thread()
{
    pthread_t pidt;

    if (pthread_create(&pidt, NULL, (void * (*)(void *)) schedule, NULL) < 0)
    {
        php_printf("pthread_create[PHPCoroutine Scheduler] failed");
    }
}
```

这个函数的作用是创建一个线程，这个线程的执行体是`schedule`函数：

```c
void schedule()
{
    while (1)
    {
        EG(vm_interrupt) = 1;
        usleep(5000);
    }
}
```

而这个`schedule`线程我们可以认为它是一个负责调用的一个线程。它设置`EG(vm_interrupt)`的值为1。设置完之后，当虚拟机检查到这个值为1的时候，就会去执行`new_interrupt_function`函数，从而实现了`yield`协程。

每次触发完虚拟机中断后，虚拟机会把`EG(vm_interrupt)`设置为0。因此，这里需要循环的去设置`EG(vm_interrupt)`的值为1。为什么这里需要使用`usleep`呢？因为中断不是每分每秒都在进行的，所以可以挂起这个线程，让其他线程跑。

好的，我们来写一段`PHP`脚本来测试一下：

```php
<?php

start_interrupt();

for (;;) {
    echo "1\n";
    sleep(1);
}
```

```shell
~/codeDir/phpCode/test # php interrupt.php 
1
yeild coroutine
1
yeild coroutine
1
yeild coroutine
1
yeild coroutine
1
yeild coroutine
1
......
^C
~/codeDir/phpCode/test # 
```





