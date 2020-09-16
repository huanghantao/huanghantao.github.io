---
title: Swoole创建多个Process的时候报错打开太多文件
date: 2020-08-12 17:16:02
tags:
- Swoole
---

例如如下代码：

```php
$files = array(<URLs of over 500 files>);

foreach($files as $file) {
    $processes[$file] = new \Swoole\Process(function () use ($file) {
        $data = file_get_contents($file);
        file_put_contents("dir/path/to/new/file.abc", $data);
    }

    $processes[$data]->start();
}
```

可能会报如下错误：

```bash
WARNING swPipeUnsock_create(:83): socketpair() failed, Error: Too many open files[24]
```

这是因为`Swoole`在`new Process`的时候，默认会创建一对管道，这样就会消耗两个`fd`（每创建一个进程，都会消耗两个`fd`）。如果创建的`Process`过多的话，就会出现“打开文件过多的错误”。

解决这个问题的方法有两个。

第一，我们配置`ulimit -n`，调大最大打开文件的上限。例如：

```bash
ulimit -n 100000
```

第二，因为我们这个程序并没有涉及到进程间的通信，所以完全可以不创建这`pipe`。此时，我们只需要设置`new Process`的第三个参数为`0`即可。例如：

```php
new Swoole\Process($fn, false, 0);
```
