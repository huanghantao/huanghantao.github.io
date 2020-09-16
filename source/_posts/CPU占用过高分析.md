---
title: CPU占用过高分析
date: 2020-09-07 18:42:17
tags:
- 性能分析
---

今天遇到了一个`hyperf`死循环的`bug`，排查了很久，没有思路。后来峰哥指导，立马定位出了问题。

定位步骤，首先，通过`perf top`命令来查看系统的`cpu`占用情况：

```bash
perf top -p 19732

Samples: 16K of event 'cpu-clock', 4000 Hz, Event count (approx.): 3364648111 lost: 0/0 drop: 0/0
Overhead  Shared Object        Symbol
   9.02%  [kernel]             [k] _raw_spin_unlock_irqrestore
   7.96%  php                  [.] execute_ex
   6.85%  [kernel]             [k] finish_task_switch
   2.82%  php                  [.] ZEND_FETCH_OBJ_R_SPEC_UNUSED_CONST_HANDLER
   2.05%  libpthread-2.17.so   [.] __libc_recv
   1.93%  php                  [.] ZEND_INIT_METHOD_CALL_SPEC_UNUSED_CONST_HANDLER
   1.55%  libc-2.17.so         [.] __memmove_ssse3_back
   1.43%  [vdso]               [.] __vdso_gettimeofday
   1.35%  libc-2.17.so         [.] __memcpy_ssse3_back
   1.31%  php                  [.] zend_leave_helper_SPEC
```

可以看到，`execute_ex`这个函数的`Overhead`非常的高。所以，可以大改猜测是`PHP`代码的问题。然后，我们找到`PHP`的进程，用`strace`看一下进程在做啥事情：

```bash
strace -p 19732

sendto(20, "*3\r\n$5\r\nBRPOP\r\n$13\r\nqueue:waiting\r\n$1\r\n2\r\n", 42, 0, NULL, 0) = 42
recvfrom(20, "-", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "-MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk. Commandsthat may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails (stop-writes-on-bgsave-error option). Please check the Redis logs for details about the RDB error.\r\n", 8192, 0, NULL, NULL) = 346
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$13\r\nqueue:delayed\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 101, 0, NULL, 0) = 101
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$14\r\nqueue:reserved\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 102, 0, NULL, 0) = 102
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*3\r\n$5\r\nBRPOP\r\n$13\r\nqueue:waiting\r\n$1\r\n2\r\n", 42, 0, NULL, 0) = 42
recvfrom(20, "-", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "-MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk. Commandsthat may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails (stop-writes-on-bgsave-error option). Please check the Redis logs for details about the RDB error.\r\n", 8192, 0, NULL, NULL) = 346
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$13\r\nqueue:delayed\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 101, 0, NULL, 0) = 101
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$14\r\nqueue:reserved\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 102, 0, NULL, 0) = 102
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*3\r\n$5\r\nBRPOP\r\n$13\r\nqueue:waiting\r\n$1\r\n2\r\n", 42, 0, NULL, 0) = 42
recvfrom(20, "-", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "-MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk. Commandsthat may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails (stop-writes-on-bgsave-error option). Please check the Redis logs for details about the RDB error.\r\n", 8192, 0, NULL, NULL) = 346
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$13\r\nqueue:delayed\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 101, 0, NULL, 0) = 101
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$14\r\nqueue:reserved\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 102, 0, NULL, 0) = 102
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*3\r\n$5\r\nBRPOP\r\n$13\r\nqueue:waiting\r\n$1\r\n2\r\n", 42, 0, NULL, 0) = 42
recvfrom(20, "-", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "-MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk. Commandsthat may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails (stop-writes-on-bgsave-error option). Please check the Redis logs for details about the RDB error.\r\n", 8192, 0, NULL, NULL) = 346
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$13\r\nqueue:delayed\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 101, 0, NULL, 0) = 101
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$14\r\nqueue:reserved\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 102, 0, NULL, 0) = 102
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*3\r\n$5\r\nBRPOP\r\n$13\r\nqueue:waiting\r\n$1\r\n2\r\n", 42, 0, NULL, 0) = 42
recvfrom(20, "-", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "-MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk. Commandsthat may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails (stop-writes-on-bgsave-error option). Please check the Redis logs for details about the RDB error.\r\n", 8192, 0, NULL, NULL) = 346
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$13\r\nqueue:delayed\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 101, 0, NULL, 0) = 101
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$14\r\nqueue:reserved\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 102, 0, NULL, 0) = 102
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*3\r\n$5\r\nBRPOP\r\n$13\r\nqueue:waiting\r\n$1\r\n2\r\n", 42, 0, NULL, 0) = 42
recvfrom(20, "-", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "-MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk. Commandsthat may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails (stop-writes-on-bgsave-error option). Please check the Redis logs for details about the RDB error.\r\n", 8192, 0, NULL, NULL) = 346
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$13\r\nqueue:delayed\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 101, 0, NULL, 0) = 101
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$14\r\nqueue:reserved\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 102, 0, NULL, 0) = 102
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*3\r\n$5\r\nBRPOP\r\n$13\r\nqueue:waiting\r\n$1\r\n2\r\n", 42, 0, NULL, 0) = 42
recvfrom(20, "-", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "-MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk. Commandsthat may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails (stop-writes-on-bgsave-error option). Please check the Redis logs for details about the RDB error.\r\n", 8192, 0, NULL, NULL) = 346
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$13\r\nqueue:delayed\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 101, 0, NULL, 0) = 101
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$14\r\nqueue:reserved\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 102, 0, NULL, 0) = 102
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*3\r\n$5\r\nBRPOP\r\n$13\r\nqueue:waiting\r\n$1\r\n2\r\n", 42, 0, NULL, 0) = 42
recvfrom(20, "-", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "-MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk. Commandsthat may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails (stop-writes-on-bgsave-error option). Please check the Redis logs for details about the RDB error.\r\n", 8192, 0, NULL, NULL) = 346
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$13\r\nqueue:delayed\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 101, 0, NULL, 0) = 101
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$14\r\nqueue:reserved\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 102, 0, NULL, 0) = 102
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*3\r\n$5\r\nBRPOP\r\n$13\r\nqueue:waiting\r\n$1\r\n2\r\n", 42, 0, NULL, 0) = 42
recvfrom(20, "-", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "-MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk. Commandsthat may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails (stop-writes-on-bgsave-error option). Please check the Redis logs for details about the RDB error.\r\n", 8192, 0, NULL, NULL) = 346
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$13\r\nqueue:delayed\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 101, 0, NULL, 0) = 101
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*7\r\n$16\r\nZREVRANGEBYSCORE\r\n$14\r\nqueue:reserved\r\n$10\r\n1599474415\r\n$4\r\n-inf\r\n$5\r\nLIMIT\r\n$1\r\n0\r\n$3\r\n100\r\n", 102, 0, NULL, 0) = 102
recvfrom(20, "*", 1, MSG_PEEK, NULL, NULL) = 1
recvfrom(20, "*0\r\n", 8192, 0, NULL, NULL) = 4
recvfrom(20, 0x7f08526509e0, 1, MSG_PEEK, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(20, "*3\r\n$5\r\nBRPOP\r\n$13\r\nqueue:waiting\r\n$1\r\n2\r\n", 42, 0, NULL, 0) = 42
```

可以看到，`hyperf`应该是没有去判断`redis`是否崩溃，即使`redis`崩溃了，也在一直循环的读取`redis`。
