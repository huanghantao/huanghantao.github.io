---
title: 《就是要你懂swoole》-- 编程须知
date: 2018-07-28 11:41:13
tags:
- PHP
- swoole
---

小伙伴们，这篇博客对应的官方文档[在此处](https://wiki.swoole.com/wiki/page/p-instruction.html)。建议大家对照着官方文档来阅读我的博客，这样效果会更好。OK，开始我们的学习。

## 进程隔离

官网上有那么一句话：

```
进程隔离也是很多新手经常遇到的问题。修改了全局变量的值，为什么不生效，原因就是全局变量在不同的进程，内存空间是隔离的，所以无效。
```

如果小伙伴们之前对进程没有概念，要理解起来还是有一定难度的。所以在这里，我有必要说一下进程是什么东西。

简单来说，进程是一个**数据结构**，数据结构里面存放了很多成员变量，这个是最本质的。很多书说：

```
进程是资源的集合
```

那么这句话所说的资源指的就是这些变量存放的东西，集合指的就是进程这个数据结构。

如果还是有一些抽象，不要紧，我们去看看Linux中进程这个数据结构的定义是怎样的。数据结构的定义[在这里](https://github.com/torvalds/linux/blob/master/include/linux/sched.h)，大概是在593行的位置：

![](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E7%BC%96%E7%A8%8B%E9%A1%BB%E7%9F%A5/1.png)

我列举几个这个数据结构里面我们听的比较多的成员变量：

```c
struct task_struct {
    pid_t					pid;
    struct list_head		children;
    struct files_struct		*files;
}
```

其中，`pid`变量里面保存着这个进程的标识，也就是说给进程标一个号，区分一下各个进程。通过`children`变量，操作系统可以找到这个进程的所有子进程。变量`files`里面有一个变量`fdtab`，通过`fdtab`里面的`fd`指针，可以找到当前进程打开的所有文件，如下图所示：

![](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E7%BC%96%E7%A8%8B%E9%A1%BB%E7%9F%A5/2.png)

操作系统会为每一个进程分配一个`task_struct`数据结构，一旦CPU执行某个进程的代码的时候，操作系统把当前进程的这些变量提供给CPU。因为每个进程都有自己的这个`task_struct`数据结构，所以每个进程的变量是在各自的进程里面的，因此不同进程的变量是隔离的，**这些变量也包括全局变量、文件句柄（即上图中的fd）**。

紧接着，官网又给出那么一句话：

```
不同进程的文件句柄是隔离的，所以在A进程创建的Socket连接或打开的文件，在B进程内是无效，即使是将它的fd发送到B进程也是不可用的
```

这句话什么意思呢？为了搞清楚，我们要知道文件句柄的作用。

当用户调用open系统调用（或者其他打开文件的系统调用）的时候，内核会创建一个打开文件对象来表示该文件的一个打开实例。内核同时也会分配一个文件描述符（也就是上图中的fd）作为打开文件对象的句柄。如下图所示：

![](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E7%BC%96%E7%A8%8B%E9%A1%BB%E7%9F%A5/3.png)

open系统调用向进程返回这个文件描述符，然后这个文件描述符就被存放在`file array`里面了。然后进程就可以通过这个`fd1`来找到对应的文件。那么，图中的offset是什么意思呢（我们把offset叫做**打开文件对象**）？在Unix系统中，文件默认是顺序访问的。当用户打开文件的时候，内核初始化这个offset的偏移指针为0。举个例子，如果当前进程刚打开一个文件（文件内容是字符串`abcdefghijklnm`），那么此时offset的状态如下：

![](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E7%BC%96%E7%A8%8B%E9%A1%BB%E7%9F%A5/5.png)

因为offset的偏移指针为0，所以指向字母a。所以，当我们通过fd去读取文件内容的时候，读取到的第一个字符就是字母`a`。假设我连续读了3个字节，那么此时的状态应该是这个样子的：

![](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E7%BC%96%E7%A8%8B%E9%A1%BB%E7%9F%A5/6.png)

也就是说，当进程从文件里面读取3个字节之后，offset此时指向字母d。当进程下一次读取文件的时候，读取出来的第一个字母就是d了。这就是offset的概念。

同一个进程还可以多次打开同一个文件，如下图所示：

![](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E7%BC%96%E7%A8%8B%E9%A1%BB%E7%9F%A5/7.png)

可以看出，虽然fd1和fd2都是指向同一个文件，但是，如果进程通过fd1去读取文件的话，读到的第一个字母是d；如果进程通过fd2去读取文件的话，读取到的第一个字母是a。所以，每个文件描述符代表着与文件的一个独立会话，对应的打开文件对象（即图中的offset）保存者该会话的内容，这包括了被打开文件的模式（只读、只写、读写）以及下一次读取或者写入时的偏移指针。

我们通过代码来直观感受一下：

```php
<?php

$handle1 = fopen("file.txt", "r");
$handle2 = fopen("file.txt", "r");

$content = fread ($handle1 , 3);

echo "The process reads three bytes through handle1, the content is: " . $content . PHP_EOL;

$content1 = fread ($handle1 , 1);
$content2 = fread ($handle2 , 1);

echo "The process reads one bytes through handle1, the content is: " . $content1 . PHP_EOL;
echo "The process reads one bytes through handle2, the content is: " . $content2 . PHP_EOL;
```

结果如下：

![](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E7%BC%96%E7%A8%8B%E9%A1%BB%E7%9F%A5/8.png)

验证了我们的说法。

正是因为offset这个打开文件对象对同一个文件的不同会话进行了隔离，所以，才有了官网的这句话：

```
不同进程的文件句柄是隔离的，所以在A进程创建的Socket连接或打开的文件，在B进程内是无效，即使是将它的fd发送到B进程也是不可用的
```

也就是说，**就算fd是一样的，但是offset是不一样的**。

其实到这里算是讲完了隔离，但是，我还想再讲一点其他的东西。

即，同一个进程是否可以让多个fd指向同一个offset，从而达到多个fd共享同一个offset的效果呢？答案是可以做到。

进程可以通过dup系统调用来复制一个文件描述符fd，这样一来，两个文件描述符共享着相同的会话：

![](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E7%BC%96%E7%A8%8B%E9%A1%BB%E7%9F%A5/9.png)

代码如下：

```php
<?php

$handle1 = fopen("file.txt", "r");
$handle2 = dup($handle1);

$content = fread ($handle1 , 3);

echo "The process reads three bytes through handle1, the content is: " . $content . PHP_EOL;

$content1 = fread ($handle1 , 1);
$content2 = fread ($handle2 , 1);

echo "The process reads one bytes through handle1, the content is: " . $content1 . PHP_EOL;
echo "The process reads one bytes through handle2, the content is: " . $content2 . PHP_EOL;
```

（注意，这段代码运行不了，因为PHP没有提供dup这个函数）

假设，这个函数是可以运行，那么输出的结果应该是

```
The process reads three bytes through handle1, the content is: abc
The process reads one bytes through handle1, the content is: d
The process reads one bytes through handle2, the content is: e
```



---

下面是更新的内容，经过热心网友 [@许怀远](https://www.zhihu.com/people/0521957489f668a5ea4b848d93eb2494)的指正，unix domain socket可以用来在不同的进程之间传递fd，而且传过去后还可以正常使用。至于如何使用unix domain socket，小伙伴们可以在《UNIX网络编程》卷一的第15章找到答案。

































