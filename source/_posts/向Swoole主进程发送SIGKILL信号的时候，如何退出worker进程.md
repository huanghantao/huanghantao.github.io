---
title: 向Swoole主进程发送SIGKILL信号的时候，如何退出worker进程
date: 2021-04-25 20:37:27
tags:
- Swoole
---

当我们想要给主进程发送`SIGKILL`信号的时候，会发现，`Worker`进程也会退出，因为`SIGKILL`信号无法设置信号处理器，所以需要其他的方法来实现这个功能。代码也很简单，只需要在子进程执行一行代码即可：

```cpp
prctl(PR_SET_PDEATHSIG, SIGTERM);
```

对于这行代码的解释如下：

```bash
PR_SET_PDEATHSIG

Set the parent process death signal of the calling process to arg2 (either a signal value in the range 1..maxsig, or 0 to clear). This is the signal that the calling process will get when its parent dies. This value is cleared for the child of a fork(2) and (since Linux 2.4.36 / 2.6.23) when executing a set-user-ID or set-group-ID binary.
```

意思就是说，当父进程挂了之后，子进程会收到`prctl`函数第二个参数设置的信号。
