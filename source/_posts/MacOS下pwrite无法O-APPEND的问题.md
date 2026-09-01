---
title: MacOS下pwrite无法O_APPEND的问题
date: 2020-11-17 10:19:02
tags:
- MacOS
---

这个问题来自`Swoole`的一个[`issue`](https://github.com/swoole/swoole-src/issues/3839)。

有如下代码：

```cpp
#include <sys/file.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>
#include <unistd.h>

int main(int argc, char const *argv[]) {
    int flags = 0;
    int fd;

    flags = O_CREAT | O_WRONLY;
    fd = open("test.txt", flags, 0644);
    pwrite(fd, "first line\n", strlen("first line\n"), 0);

    flags = O_APPEND | O_CREAT | O_WRONLY;
    fd = open("test.txt", flags, 0644);
    pwrite(fd, "second line\n", strlen("second line\n"), 0);

    flags = O_APPEND | O_CREAT | O_WRONLY;
    fd = open("test.txt", flags, 0644);
    pwrite(fd, "third line\n", strlen("third line\n"), 0);

    return 0;
}
```

此时，`test.txt`文件里面的内容是：

```txt
third line


```

我们发现，这实际上没有追加，而是覆盖了之前写入的内容。也就意味着`pwrite`的`offset`和`O_APPEND`没有一起起到作用。

我们换成`write`来测试追加：

```cpp
#include <sys/file.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>
#include <unistd.h>

int main(int argc, char const *argv[]) {
    int flags = 0;
    int fd;

    flags = O_CREAT | O_WRONLY;
    fd = open("test.txt", flags, 0644);
    write(fd, "first line\n", strlen("first line\n"));

    flags = O_APPEND | O_CREAT | O_WRONLY;
    fd = open("test.txt", flags, 0644);
    write(fd, "second line\n", strlen("second line\n"));

    flags = O_APPEND | O_CREAT | O_WRONLY;
    fd = open("test.txt", flags, 0644);
    write(fd, "third line\n", strlen("third line\n"));

    return 0;
}
```

此时，`test.txt`文件里面的内容是：

```txt
first line
second line
third line

```

追加成功了。
