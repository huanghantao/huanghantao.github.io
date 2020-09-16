---
title: TLS（线程局部存储）
date: 2019-04-01 09:34:29
tags:
- 线程
---

我们知道，在同一个进程里面的多个线程，它们是会共享全局变量的。但是，这其中或许会涉及到一些线程同步的问题。如果我只打算让同一个线程里面调用的各个函数都可以访问这个数据，不希望其他线程访问到别的线程的数据，该怎么办？可以通过线程局部存储来解决：

```c
#include <stdio.h>
#include <unistd.h>
#include <stdint.h>
#include <stdlib.h>
#include <pthread.h>

pthread_key_t key;

void func()
{
    printf("in func function------\n");
    printf ("pthread_getspecific(key)返回的指针为: %p\n", (int *)pthread_getspecific(key));
    printf ("pthread_getspecific(key)对应的值为: %d\n", *(int *)pthread_getspecific(key));
}

void *start(void *arg)
{
    int id = (int)(uintptr_t)arg;

    if (id == 2) {
        sleep(1);
    }

    pthread_setspecific (key, &id);
    printf("in start function------\n");
    printf("局部变量id的地址为: %p\n", &id);
    printf ("pthread_getspecific(key)返回的指针为: %p\n", (int *)pthread_getspecific(key));
    printf ("pthread_getspecific(key)对应的值为: %d\n", *(int *)pthread_getspecific(key));
    func();
}

int main(int argc, char const *argv[])
{
    pthread_t tid1, tid2;

    pthread_key_create(&key, NULL);

    pthread_create(&tid1, NULL, (void *)start, (void *)(uintptr_t)1);
    pthread_create(&tid2, NULL, (void *)start, (void *)(uintptr_t)2);
    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);

    pthread_key_delete(key);
    return (0);
}
```

结果：

```shell
sh-4.2# gcc test.c -pthread
sh-4.2# ./a.out 
in start function------
局部变量id的地址为: 0x7f38de731f0c
pthread_getspecific(key)返回的指针为: 0x7f38de731f0c
pthread_getspecific(key)对应的值为: 1
in func function------
pthread_getspecific(key)返回的指针为: 0x7f38de731f0c
pthread_getspecific(key)对应的值为: 1
in start function------
局部变量id的地址为: 0x7f38ddf30f0c
pthread_getspecific(key)返回的指针为: 0x7f38ddf30f0c
pthread_getspecific(key)对应的值为: 2
in func function------
pthread_getspecific(key)返回的指针为: 0x7f38ddf30f0c
pthread_getspecific(key)对应的值为: 2
sh-4.2# 
```

我们发现，虽然，这个全局变量key是所有线程都可以访问到的，但是，key对应的那个数据，却只有线程自己可以访问得到，从而实现了全局的数据在线程间的隔离。