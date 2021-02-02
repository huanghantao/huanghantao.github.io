---
title: ptrace使用
date: 2021-02-02 20:19:39
tags:
---

`ptrace`系统调用提供了一种方法让追踪者`tracer`对被追踪者`tracee`进行观察与控制，我们通过一些例子来进行讲解。

首先，我们看这么一段代码：

```cpp
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <sys/reg.h>
#include <stdio.h>
#include <signal.h>

int main()
{
    pid_t pid;

    pid = fork();

    if (pid == 0) { // child process
        printf("child process pid %ld\n", getpid());

        char *argv[] = {"ls", NULL};
        execv("/bin/ls", argv);
    }
    else { // parent process
        printf("parent process pid %ld\n", getpid());

        wait(NULL);
        printf("parent process exit\n");
    }

    return 0;
}
```

大概说说这段代码做的事情。首先创建一个子进程，父进程调用`wait`函数等待子进程退出。子进程调用`execv`创建一个新的进程，并且在很短的时间内执行`ls`命令。最后，父进程等待`1s`后退出。

所以，输出结果如下：

```bash
[root@97043d024896 test]# gcc ptrace.c
[root@97043d024896 test]# ./a.out
parent process pid 42037
child process pid 42038
a.out  ptrace.c
parent process exit
[root@97043d024896 test]#
```

我们来看看`strace`信息：

```bash
strace -f ./a.out

[pid 49922] exit_group(0)               = ?
[pid 49922] +++ exited with 0 +++
<... wait4 resumed>NULL, 0, NULL)       = 49922
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=49922, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
write(1, "parent process exit\n", 20parent process exit
)   = 20
exit_group(0)                           = ?
+++ exited with 0 +++
```

其中，`49922`这个是子进程。可以看到，子进程退出前，父进程处于`wait`状态，当父进程收到`SIGCHLD`信号之后，父进程`resume`了，然后父进程退出了。

现在，让我们使用`ptrace`，看看进程的状态会发送什么变化：

```cpp
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <sys/reg.h>
#include <stdio.h>
#include <signal.h>

int main()
{
    pid_t pid;

    pid = fork();

    if (pid == 0) { // child process
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        printf("child process pid %ld\n", getpid());

        char *argv[] = {"ls", NULL};
        execv("/bin/ls", argv);
    }
    else { // parent process
        printf("parent process pid %ld\n", getpid());

        wait(NULL);
        wait(NULL);

        printf("parent process exit\n");
    }

    return 0;
}
```

我们在子进程里面调用了`ptrace`函数，并且设置了`PTRACE_TRACEME`，表示希望父进程跟踪这个子进程。运行结果如下：

```bash
[root@97043d024896 test]# ./a.out
parent process pid 43160
child process pid 43161

```

我们发现，子进程调用了`execv`之后，不会立马执行`ls`命令了。我们来看看进程的状态：

查看父进程的状态：

```bash
[root@97043d024896 test]# ps -aux | grep -w "43160" | grep -v "grep"
root     43160  0.0  0.0   4228   680 pts/7    S+   13:16   0:00 ./a.out
[root@97043d024896 test]#
```

此时父进程的状态是`S+`，表示处于睡眠状态。

查看子进程的状态：

```bash
[root@97043d024896 test]# ps -aux | grep -w "43161" | grep -v "grep"
root     43161  0.0  0.0    416     4 pts/7    t+   13:16   0:00 ls
[root@97043d024896 test]#
```

此时子进程的状态是`t+`，表示处于暂停状态。要么是因为一个`job control`信号，要么是因为它正在被追踪。我们这个场景就是属于被追踪。

`strace`信息如下：

```bash
child process pid 51875
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 7), ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fb5a89b1000
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_TRAPPED, si_pid=51875, si_uid=0, si_status=SIGTRAP, si_utime=0, si_stime=0} ---
write(1, "parent process pid 51874\n", 25parent process pid 51874
) = 25
wait4(-1, NULL, 0, NULL)                = 51875
wait4(-1,
```

此时，没有子进程退出的状态。并且，父进程也处于`wait`信号的状态。

那么，如何让子进程恢复运行呢？代码如下：

```cpp
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <sys/reg.h>
#include <stdio.h>
#include <signal.h>

int main()
{
    pid_t pid;

    pid = fork();

    if (pid == 0) { // child process
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        printf("child process pid %ld\n", getpid());

        char *argv[] = {"ls", NULL};
        execv("/bin/ls", argv);
    }
    else { // parent process
        printf("parent process pid %ld\n", getpid());

        wait(NULL);

        ptrace(PTRACE_CONT, pid, NULL, NULL);

        wait(NULL);

        printf("parent process exit\n");
    }

    return 0;
}
```

结果如下：

```cpp
[root@97043d024896 test]# ./a.out
parent process pid 49313
child process pid 49314
a.out  ptrace.c
parent process exit
```

此时，子进程成功的执行了`ls`命令。

`strace`信息如下：

```bash
child process pid 52573
getpid()                                = 52572
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_TRAPPED, si_pid=52573, si_uid=0, si_status=SIGTRAP, si_utime=0, si_stime=0} ---
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 7), ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6eb99d8000
write(1, "parent process pid 52572\n", 25parent process pid 52572
) = 25
wait4(-1, NULL, 0, NULL)                = 52573
ptrace(PTRACE_CONT, 52573, NULL, SIG_0) = 0
wait4(-1, a.out  ptrace.c
NULL, 0, NULL)                = 52573
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=52573, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
write(1, "parent process exit\n", 20parent process exit
)   = 20
exit_group(0)                           = ?
+++ exited with 0 +++
```

我们发现，父进程收到了两次`SIGCHLD`信号，说明子进程执行完`ls`命令之后退出了。
