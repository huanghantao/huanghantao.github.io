---
title: c++的unique_lock
date: 2019-09-02 10:49:16
tags:
- c++
- unique_lock
---

今天，在看`Swoole`的`commit`的时候，看到了一个`commit`[f413593](https://github.com/swoole/swoole-src/commit/f413593b0007c83b4869ab25457cae5e88e18960)。

代码如下：

```cpp
inline size_t count()
{
    unique_lock<mutex> lock(_mutex);
    return _queue.size();
}
```

也就是使用了`unique_lock`，之前没见过。并且，这里我只看到了加锁的过程，没看到解锁的过程，所以比较好奇。于是搜了一下，得到如下答案：

```
When you want to lock a mutex, you create a local variable of type std::unique_lock passing the mutex as parameter. When the unique_lock is constructed it will lock the mutex, and it gets destructed it will unlock the mutex. More importantly: If a exceptions is thrown, the std::unique_lock destructer will be called and so the mutex will be unlocked.

当需要锁定mutex时，可以创建一个 std::unique_lock 类型的局部变量，将 mutex 作为参数传递。当这个 unique_lock 被构造出来时，它会锁定 mutex，当 unique_lock 被析构的时候，它会解锁 mutex。 更重要的是: 如果抛出异常，将调用     std::unique_lock destructer，从而解锁 mutex。
```

这里写一个完整的例子：

```cpp
#include <mutex>
#include <thread>
#include <iostream>
#include <vector>

using namespace std;
 
int some_shared_var = 0;

void func()
{
    int a = 1;

    some_shared_var += a;
}

int main()
{
    size_t count = 10000;
    vector<thread> threads;

    for (size_t i = 0; i < count; i++)
    {
        threads.push_back(thread(func));
    }

    for (size_t i = 0; i < count; i++)
    {
        threads[i].join();
    }

    cout << some_shared_var << endl;
}
```

这里，我们创建了`10000`个线程来执行`func`函数里面的代码。但是，在写入线程间共享的变量`some_shared_var`的时候我们没有去保护它，而是任由线程间去竞争执行。

我们来编译运行下代码：

```shell
~/codeDir/cppCode/test # g++ mutex.cpp ; ./a.out ; ./a.out ; ./a.out 
9998
9999
9999
~/codeDir/cppCode/test # 
```

我们发现，这个共享的变量累加出错了，我们期待的值应该是`10000`。

然后，我们使用下`unique_lock`：

```cpp
#include <mutex>
#include <thread>
#include <iostream>
#include <vector>

using namespace std;
 
int some_shared_var = 0;
mutex my_mutex;

void func()
{
    int a = 1;

    unique_lock<mutex> lock(my_mutex);
    some_shared_var += a;
}

int main()
{
    size_t count = 10000;
    vector<thread> threads;

    for (size_t i = 0; i < count; i++)
    {
        threads.push_back(thread(func));
    }

    for (size_t i = 0; i < count; i++)
    {
        threads[i].join();
    }

    cout << some_shared_var << endl;
}
```

在函数`func`中，

```cpp
unique_lock<mutex> lock(my_mutex);
```

后面的代码都是安全的。所以：

```cpp
some_shared_var += a;
```

这一行代码就是线程安全的。

我们编译运行下：

```shell
~/codeDir/cppCode/test # g++ mutex.cpp ; ./a.out ; ./a.out ; ./a.out 
10000
10000
10000
~/codeDir/cppCode/test # 
```

最后，`c++`有点好用鸭，有些喜欢了，慢慢学习中。

