---
title: Linux下Mutex的所有者线程先死亡造成其他线程获取锁阻塞的问题
date: 2020-09-16 18:48:38
tags:
- Linux
- 锁
---

`Swoole`最近有一个`BUG`，大概就是`Mutex`的所有者线程先死亡，造成其他线程获取锁阻塞的问题。具体的[`iseue`在这里](https://github.com/swoole/swoole-src/issues/3644)。

我们可以用如下代码来对这个问题进行复现：

```cpp
#include <pthread.h>
#include <iostream>
#include <unistd.h>

pthread_mutex_t mutex;

void *handler(void *)
{
    std::cout << "child thread" << std::endl;
    int ret = pthread_mutex_lock(&mutex);
    std::cout << "child ret: " << ret << std::endl;
    pthread_exit(NULL);
}

int main()
{
    pthread_t tid;
    pthread_mutexattr_t attr;
    int ret;

    pthread_mutexattr_init(&attr);
    pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_ERRORCHECK);

    pthread_mutex_init(&mutex, &attr);
    pthread_mutexattr_destroy(&attr);

    pthread_create(&tid, NULL, handler, NULL);

    sleep(2);
    std::cout << "father awake" << std::endl;

    ret = pthread_mutex_lock(&mutex);
    std::cout << "father ret: " << ret << std::endl;
    return 0;
}
```

这段代码的意思是，主线程创建了子线程之后，立马调用`sleep`阻塞住。然后子线程去获取锁，然后子线程立马退出，导致锁没有被解开。然后当主线程`sleep`结束后，尝试获取锁的时候，就会阻塞住。（导致这个现象的原因是，子线程退出后，操作系统并不会把锁解开）

我们可以执行下上面这段代码，执行结果如下：

```bash
g++ lock.cc -lpthread
./a.out

child thread
child ret: 0
father awake

```

解决方式如下：

```cpp
#include <iostream>
#include <unistd.h>
#include <pthread.h>
#include <string.h>
#include <errno.h>

pthread_mutex_t lock;

void *dropped_thread(void*)
{
    std::cout << "Setting lock..." << std::endl;
    pthread_mutex_lock(&lock);
    std::cout << "Lock set, now exiting without unlocking..." << std::endl;
    pthread_exit(NULL);
}

int main(int argc, char *argv[])
{
    int ret;
    pthread_t lock_getter;
    pthread_mutexattr_t attr;

    pthread_mutexattr_init(&attr);
    pthread_mutexattr_setrobust(&attr, PTHREAD_MUTEX_ROBUST);
    pthread_mutex_init(&lock, &attr);
    pthread_mutexattr_destroy(&attr);
    pthread_create(&lock_getter, NULL, dropped_thread, NULL);
    sleep(2);

    std::cout << "Inside main" << std::endl;
    std::cout << "Attempting to acquire mutex?" << std::endl;

    ret = pthread_mutex_lock(&lock);
    if (ret == EOWNERDEAD) {
        std::cout << "errno: " << ret << ", error: " << strerror(ret) << std::endl;

        std::cout << "consistent mutex" << std::endl;
        pthread_mutex_consistent(&lock);

        std::cout << "unlock mutex" << std::endl;
        pthread_mutex_unlock(&lock);
    } else {
        std::cout << "errno: " << ret << ", error: " << strerror(ret) << std::endl;
    }
    std::cout << "Attempting to acquire mutex?" << std::endl;
    ret = pthread_mutex_lock(&lock);
    if (ret != 0) {
        std::cout << "errno: " << ret << ", error: " << strerror(ret) << std::endl;
    } else {
        std::cout << "Successfully acquired lock!" << std::endl;
        pthread_mutex_destroy(&lock);
    }

    return 0;
}
```

执行结果如下：

```bash
g++ lock.cc -lpthread
./a.out

Setting lock...
Lock set, now exiting without unlocking...
Inside main
Attempting to acquire mutex?
errno: 130, error: Owner died
consistent mutex
unlock mutex
Attempting to acquire mutex?
Successfully acquired lock!
```

这里的核心是，我们对这个锁设置了`PTHREAD_MUTEX_ROBUST`。这样的话，锁的拥有者退出后，其他线程去获取锁的时候，就不会阻塞住了，而是返回错误码`EOWNERDEAD`。并且，这里还有一个细节就是，在获得了错误码之后，我们需要设置锁的状态为`consistent`。这样，其他线程就能解开锁了。
