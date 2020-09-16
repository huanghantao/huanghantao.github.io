---
title: supervisor原理分析
date: 2020-08-14 15:21:57
tags:
- Linux
---

> 背景
>
> 之前听过这个工具，但是没用过它，只知道它是一个进程管理的工具。然后最近我在公司要用到这个东西来部署`Swoole Server`服务，并且在使用它的时候，遇到了端口占用的问题，经过同事指点，说是`supervisor`不能够用来管理守护进程，而我的`Swoole Server`配置了守护进程。于是我对这个工具的工作原理产生了兴趣。

我们先来感受一下`supervisor`的工作原理。首先，我们来写一份配置：

```ini
[root@e2a14c00e7f6 ~]# cat /etc/supervisord.d/cat.ini
[program:cat]
process_name=%(program_name)s
directory=/tmp
command=cat
autostart=true
autorestart=true
user=root
stdout_logfile=/tmp/cat.log
stderr_logfile=/tmp/cat.err.log
```

这里，我打算起一个`cat`命令进程。

然后，我们来启动服务：

```bash
[root@e2a14c00e7f6 ~]# supervisorctl restart cat
cat: stopped
cat: started
[root@e2a14c00e7f6 ~]#
```

这个时候，我们来看一看我们的进程：

```bash
[root@e2a14c00e7f6 ~]# ps -ef
[root@e2a14c00e7f6 ~]# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 05:26 pts/0    00:00:00 /usr/sbin/sshd -D
root         6     1  0 05:26 ?        00:00:01 sshd: root@pts/1
root         8     6  0 05:26 pts/1    00:00:00 -bash
root       153     1  0 07:17 ?        00:00:00 /usr/bin/python /usr/bin/supervisord -c /etc/supervis
root       172     1  0 07:19 ?        00:00:00 sshd: root@pts/2
root       174   172  0 07:19 pts/2    00:00:00 -bash
root       201   153  0 07:31 ?        00:00:00 cat
root       202   174  0 07:32 pts/2    00:00:00 ps -ef
[root@e2a14c00e7f6 ~]#
```

我们发现，这里有一个`supervisord`进程，根据名字后面有一个`d`，我们可以很容易猜到，这应该是一个守护进程。然后，我们发现`cat`进程他的父进程是`153`，这正好是`supervisord`进程的`pid`。所以，我们可以大概猜测，`supervisord`是通过监听`SIGCHLD`来实现进程重启的。我们来验证下。

首先，我们查看一下`supervisord`进程的系统调用：

```bash
[root@e2a14c00e7f6 ~]# strace -p 153
strace: Process 153 attached
restart_syscall(<... resuming interrupted read ...>) = 0
gettimeofday({tv_sec=1597390386, tv_usec=920369}, NULL) = 0
wait4(-1, 0x7ffc80915174, WNOHANG, NULL) = 0
gettimeofday({tv_sec=1597390386, tv_usec=921726}, NULL) = 0
poll([{fd=4, events=POLLIN|POLLPRI|POLLHUP}, {fd=9, events=POLLIN|POLLPRI|POLLHUP}, {fd=11, events=POLLIN|POLLPRI|POLLHUP}], 3, 1000) = 0 (Timeout)
gettimeofday({tv_sec=1597390387, tv_usec=924476}, NULL) = 0
wait4(-1, 0x7ffc80915174, WNOHANG, NULL) = 0
gettimeofday({tv_sec=1597390387, tv_usec=924774}, NULL) = 0
poll([{fd=4, events=POLLIN|POLLPRI|POLLHUP}, {fd=9, events=POLLIN|POLLPRI|POLLHUP}, {fd=11, events=POLLIN|POLLPRI|POLLHUP}], 3, 1000) = 0 (Timeout)
gettimeofday({tv_sec=1597390388, tv_usec=930374}, NULL) = 0
wait4(-1, 0x7ffc80915174, WNOHANG, NULL) = 0
```

我们发现，`supervisord`一直在调用`poll`系统调用。这应该就是在监听`SIGCHLD`信号了。我们来给`cat`进程发送一个`kill`的信号试试：

```bash
poll([{fd=4, events=POLLIN|POLLPRI|POLLHUP}, {fd=9, events=POLLIN|POLLPRI|POLLHUP}, {fd=11, events=POLLIN|POLLPRI|POLLHUP}], 3, 1000) = 2 ([{fd=9, revents=POLLHUP}, {fd=11, revents=POLLHUP}])
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_KILLED, si_pid=201, si_uid=0, si_status=SIGTERM, si_utime=0, si_stime=0} ---
rt_sigreturn({mask=[]})                 = 2
read(9, "", 131072)                     = 0
read(11, "", 131072)                    = 0
gettimeofday({tv_sec=1597390506, tv_usec=625701}, NULL) = 0
wait4(-1, [{WIFSIGNALED(s) && WTERMSIG(s) == SIGTERM}], WNOHANG, NULL) = 201
gettimeofday({tv_sec=1597390506, tv_usec=627230}, NULL) = 0
gettimeofday({tv_sec=1597390506, tv_usec=627396}, NULL) = 0
stat("/etc/localtime", {st_mode=S_IFREG|0644, st_size=118, ...}) = 0
write(3, "2020-08-14 07:35:06,627 INFO exi"..., 79) = 79
lseek(3, 0, SEEK_CUR)                   = 2831
close(8)                                = 0
close(9)                                = 0
close(11)                               = 0
wait4(-1, 0x7ffc80914f64, WNOHANG, NULL) = -1 ECHILD (No child processes)
gettimeofday({tv_sec=1597390506, tv_usec=628325}, NULL) = 0
close(13)                               = 0
close(14)                               = 0
poll([{fd=4, events=POLLIN|POLLPRI|POLLHUP}], 1, 1000) = 0 (Timeout)
gettimeofday({tv_sec=1597390507, tv_usec=632307}, NULL) = 0
gettimeofday({tv_sec=1597390507, tv_usec=632405}, NULL) = 0
stat("/usr/local/sbin/cat", 0x7ffc80914d10) = -1 ENOENT (No such file or directory)
stat("/usr/local/bin/cat", 0x7ffc80914d10) = -1 ENOENT (No such file or directory)
stat("/usr/sbin/cat", 0x7ffc80914d10)   = -1 ENOENT (No such file or directory)
stat("/usr/bin/cat", {st_mode=S_IFREG|0755, st_size=54080, ...}) = 0
access("/usr/bin/cat", X_OK)            = 0
pipe([5, 6])                            = 0
pipe([8, 9])                            = 0
pipe([10, 11])                          = 0
fcntl(8, F_GETFL)                       = 0 (flags O_RDONLY)
fcntl(8, F_SETFL, O_RDONLY|O_NONBLOCK)  = 0
fcntl(10, F_GETFL)                      = 0 (flags O_RDONLY)
fcntl(10, F_SETFL, O_RDONLY|O_NONBLOCK) = 0
fcntl(6, F_GETFL)                       = 0x1 (flags O_WRONLY)
fcntl(6, F_SETFL, O_WRONLY|O_NONBLOCK)  = 0
open("/tmp/cat.log", O_WRONLY|O_CREAT|O_APPEND, 0666) = 12
lseek(12, 0, SEEK_END)                  = 0
fstat(12, {st_mode=S_IFREG|0644, st_size=0, ...}) = 0
open("/tmp/cat.err.log", O_WRONLY|O_CREAT|O_APPEND, 0666) = 13
lseek(13, 0, SEEK_END)                  = 0
fstat(13, {st_mode=S_IFREG|0644, st_size=0, ...}) = 0
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f161ad54a10) = 207
```

此时，`supervisord`收到了`SIGCHLD`信号，它知道`cat`进程挂了。然后，我们发现，这里调用了`clone`系统调用，创建了一个新的子进程，`pid`是`207`。我们可以来看看是不是：

```bash
[root@e2a14c00e7f6 ~]# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 05:26 pts/0    00:00:00 /usr/sbin/sshd -D
root         6     1  0 05:26 ?        00:00:01 sshd: root@pts/1
root         8     6  0 05:26 pts/1    00:00:00 -bash
root       153     1  0 07:17 ?        00:00:00 /usr/bin/python /usr/bin/supervisord -c /etc/supervis
root       172     1  0 07:19 ?        00:00:00 sshd: root@pts/2
root       174   172  0 07:19 pts/2    00:00:00 -bash
root       207   153  0 07:35 ?        00:00:00 cat
root       208   174  0 07:36 pts/2    00:00:00 ps -efs
[root@e2a14c00e7f6 ~]#
```

确实是`207`。

所以，`supervisord`这就实现了自动重启子进程的功能。

那么，为什么`supervisord`无法监控守护进程呢？我们来继续做实验。

这里有一个`Swoole Server`的例子：

```php
<?php

use Swoole\Server;

$serv = new Server('127.0.0.1', 9580);

$serv->set([
    'daemonize' => 1,
]);

$serv->on('Receive', function () {
});

$serv->start();
```

我们可以先来确认一下程序是否可以手动启动成功：

```bash
[root@e2a14c00e7f6 server]# php start.php
[root@e2a14c00e7f6 server]#

[root@e2a14c00e7f6 server]# netstat -antp | grep 9580
tcp        0      0 127.0.0.1:9580          0.0.0.0:*               LISTEN      337/php
[root@e2a14c00e7f6 server]#
```

我们发现，启动成功了。然后，我们需要杀死这个`server`进程：

```bash
[root@e2a14c00e7f6 server]# kill 337
[root@e2a14c00e7f6 server]#
[root@e2a14c00e7f6 server]# netstat -antp | grep 9580
[root@e2a14c00e7f6 server]#
```

确认没有问题之后，我们通过`supervisor`来启动`Swoole Server`（具体的`supervisor ini`配置大家可以自己配一下）：

```bash
[root@e2a14c00e7f6 ~]# supervisorctl restart server
server: ERROR (not running)
server: ERROR (spawn error)
```

然后我们发现启动失败了，查看日志可以看到：

```log
PHP Fatal error:  Uncaught Swoole\Exception: failed to listen server port[127.0.0.1:9580], Error: Address already in use[98] in /root/codeDir/phpCode/swoole/server/start.php:5
Stack trace:
#0 /root/codeDir/phpCode/swoole/server/start.php(5): Swoole\Server->__construct('127.0.0.1', 9580)
#1 {main}
  thrown in /root/codeDir/phpCode/swoole/server/start.php on line 5
```

说是端口被占用了。我们来看一下端口：

```bash
[root@e2a14c00e7f6 server]# netstat -antp | grep 9580
tcp        0      0 127.0.0.1:9580          0.0.0.0:*               LISTEN      234/php
[root@e2a14c00e7f6 server]#
```

我们发现，程序确实被我们的服务器给占用了。那么为什么会报这个错误呢？说明在`supervisor`的接管下，`server`被多次启动，并且是直接在`server`还没有退出的情况下启动。

首先，我们来看一下服务器进程状态：

```bash
[root@e2a14c00e7f6 server]# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 05:26 pts/0    00:00:00 /usr/sbin/sshd -D
root         6     1  0 05:26 ?        00:00:01 sshd: root@pts/1
root         8     6  0 05:26 pts/1    00:00:00 -bash
root       153     1  0 07:17 ?        00:00:00 /usr/bin/python /usr/bin/supervisord -c /etc/supervis
root       172     1  0 07:19 ?        00:00:00 sshd: root@pts/2
root       174   172  0 07:19 pts/2    00:00:00 -bash
root       234     1  0 07:48 ?        00:00:00 php start.php
root       235   234  0 07:48 ?        00:00:00 php start.php
root       238   235  0 07:48 ?        00:00:00 php start.php
root       239   235  0 07:48 ?        00:00:00 php start.php
root       250   174  0 07:52 pts/2    00:00:00 ps -ef
[root@e2a14c00e7f6 server]#
```

我们发现，因为`server`是以守护进程的方式启动的，所以`master`进程的`ppid`是`1`。（因为守护进程的实现原理是`fork + fork + exit`，所以，`master`进程自然就被`pid`为`1`的进程接管了）

正是因为`server`的父进程不是`supervisor`了，所以，`supervisor`此时不能正确的监控`server`的状态。（至于有没有其他的操作实现监控，这我没有过多的去研究它）

我们现在通过`strace`来看看。首先，`kill`掉这个`server`：

```bash
[root@e2a14c00e7f6 server]# kill 234
```

```bash
[root@e2a14c00e7f6 ~]# strace -p 153

```

然后我们另开一个终端，此时我们再次通过`supervisor`来启动`server`：

```bash
[root@e2a14c00e7f6 server]# supervisorctl restart server
server: ERROR (not running)
server: ERROR (spawn error)
```

在`strace`的终端，我们可以看到如下输出：

```bash
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f161ad54a10) = 324


START_RESTARTBLOCK (Interrupted by signal)
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=324, si_uid=0, si_status=0, si_utime=3, si_stime=3} ---

clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f161ad54a10) = 331

--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=331, si_uid=0, si_status=255, si_utime=3, si_stime=1} ---

clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f161ad54a10) = 332

--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=332, si_uid=0, si_status=255, si_utime=3, si_stime=1} ---
```

可以看到，收到了子进程退出的信息。因为我们的`server`进程因为守护进程化，最初的那个子进程是退出了的。所以，`supervisor`误认为`server`是不正常退出，它又对`server`进行了重启。但是实际上，我们的`server`已经监听了端口了，所以`supervisor`再次启动`server`，就会报错了。
