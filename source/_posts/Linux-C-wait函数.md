---
title: Linux C wait函数
date: 2019-09-20 10:49:42
tags:
---

我们先来看一段代码：

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/wait.h>

int main()
{
    pid_t id;

    id = fork();
    if (id == 0)
    {
        printf("Hello from Child!, pid: %d\n", getpid());
    }
    else
    {
        printf("Hello from Parent!, pid: %d\n", getpid());
    }

    return 0;
}
```

执行结果如下：

```shell
~/codeDir/cCode/test # ./a.out
Hello from Parent!, pid: 2290
Hello from Child!, pid: 2291
~/codeDir/cCode/test #
```

我们看看进程状态：

```shell
~/codeDir # ps -a
  PID TTY          TIME CMD
 2292 pts/2    00:00:00 ps
~/codeDir #
```

我们发现，进程直接退出了。

然后，我们再来看一段代码：

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/wait.h>

int main()
{
    pid_t id;

    id = fork();
    if (id == 0)
    {
        printf("Hello from Child!, pid: %d\n", getpid());
        sleep(-1);
    }
    else
    {
        printf("Hello from Parent!, pid: %d\n", getpid());
    }

    return 0;
}
```

我们让子进程无限阻塞住，然后父进程先退出。执行结果如下：

```shell
~/codeDir/cCode/test # ./a.out
Hello from Parent!, pid: 2314
Hello from Child!, pid: 2315
~/codeDir/cCode/test #
```

我们发现，终端不会卡住，而是直接退出了。说明，终端是否会卡住，是由父进程是否退出决定的。然后，我们看看进程状态：

```shell
~/codeDir # ps -a
  PID TTY          TIME CMD
 2315 pts/4    00:00:00 a.out
 2316 pts/2    00:00:00 ps
~/codeDir #
```

我们发现，此时子进程（`PID`是`2315`）还存在。因为进程`2315`的父进程`2314`不在了，所以进程`2315`会变成孤儿进程，此时，它的父进程会变成`init`进程。

此时，我们可以通过`wait`函数来等待子进程的结束：

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/wait.h>

int main()
{
    pid_t id;

    id = fork();
    if (id == 0)
    {
        printf("Hello from Child!, pid: %d\n", getpid());
        sleep(-1);
    }
    else
    {
        wait(NULL);
        printf("Hello from Parent!, pid: %d\n", getpid());
    }

    return 0;
}
```

其中，`wait`可以等待子进程结束。代码执行结果如下：

```shell
~/codeDir/cCode/test # ./a.out
Hello from Child!, pid: 2327

```

我们发现，此时的终端没有直接退出。我们查看一下进程状态：

```shell
~/codeDir # ps -a
  PID TTY          TIME CMD
 2326 pts/4    00:00:00 a.out
 2327 pts/4    00:00:00 a.out
 2329 pts/2    00:00:00 ps
~/codeDir #
```

我们发现，父进程和子进程都没挂掉。正是因为我们子进程一直没有退出，所以父进程阻塞在了`wait`这个地方。

我们再来看一段代码：

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/wait.h>

int main()
{
    pid_t id;

    id = fork();
    if (id == 0)
    {
        printf("Hello from Child!, pid: %d\n", getpid());
    }
    else
    {
        printf("Hello from Parent!, pid: %d\n", getpid());
        sleep(-1);
    }

    return 0;
}
```

目的是，子进程退出后，父进程一直阻塞住不退出。

执行结果如下：

```shell
~/codeDir/cCode/test # ./a.out
Hello from Parent!, pid: 2337
Hello from Child!, pid: 2338

```

因为父进程没有退出，所以终端会被卡住。此时，我们看看进程状态。

```shell
~/codeDir # ps -a
  PID TTY          TIME CMD
 2337 pts/4    00:00:00 a.out
 2338 pts/4    00:00:00 a.out <defunct>
 2339 pts/2    00:00:00 ps
~/codeDir #
```

我们发现，这里依然会看到子进程`2338`的信息，为什么呢？因为这个子进程目前是一个僵尸进程。我们发现这个进程旁边有一个标识`defunct`，代表这个进程执行完了它的任务，已经挂了，但是，这个进程的一些基本信息依然保存着。这个僵尸进程的信息依旧被内核保存在进程表里面。

我们现在用`kill`命令给僵尸进程发送信号看看：

```shell
~/codeDir # kill 2338
~/codeDir # ps -a
  PID TTY          TIME CMD
 2337 pts/4    00:00:00 a.out
 2338 pts/4    00:00:00 a.out <defunct>
 2342 pts/2    00:00:00 ps
~/codeDir #
```

我们发现，这个僵尸进程还是会被查找到？为什么呢？因为这个僵尸进程本来就是死的，当然`kill`会失效。那么，我们如何去避免这个问题呢？

可以用`wait`函数。`wait`函数除了可以等待子进程的结束，其实还可以获取到子进程的信息。这也是为什么一个子进程挂了之后，它的基本信息不会立马被内核清除的原因，因为，操作系统认为父进程会在某个时候要用到子进程的一些信息，例如子进程的退出状态。只有当父进程获取完子进程的信息之后，操作系统才会清理子进程残留的信息。而操作系统如何知道父进程读取过子进程的信息呢？那就是`wait`函数。实际上，`wait`函数可以传入一个获取子进程信息的结构体，只不过我们的代码没有去获取而已，说明此时我们是不关心子进程的状态的。

分析完之后，我们来看下一段代码：

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/wait.h>

int main()
{
    pid_t id;

    id = fork();
    if (id == 0)
    {
        printf("Hello from Child!, pid: %d\n", getpid());
        sleep(1);
    }
    else
    {
        wait(NULL);
        printf("Hello from Parent!, pid: %d\n", getpid());
        sleep(-1);
    }

    return 0;
}
```

这里，我们的父进程会去调用`wait`函数，等待子进程结束。等子进程执行完毕之后，父进程才回去执行。一旦父进程调用了`wait`函数，操作系统就知道父进程获取过子进程的信息，因此，操作系统就可以放心的去清理僵尸进程的残留信息了。

我们执行代码：

```shell
~/codeDir/cCode/test # ./a.out
Hello from Child!, pid: 2350
Hello from Parent!, pid: 2349

```

我们看看进程的状态：

```shell
~/codeDir # ps -a
  PID TTY          TIME CMD
 2349 pts/4    00:00:00 a.out
 2351 pts/2    00:00:00 ps
~/codeDir #
```

我们发现，子进程`2350`不再出现在进程状态列表里面了，说明子进程已经被清理完毕了。
