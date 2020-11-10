---
title: Swoole的writeVector和readVector功能介绍
date: 2020-11-10 14:10:38
tags:
- PHP
- Swoole
---

在`Swoole v4.5.7`版本中，我们新增了两个`API`，分别是`writeVector`和`readVector`。这篇文章，我们来介绍下这两个方法。

## writeVector

> 该方法在\Swoole\Coroutine\Socket类里面

这个方法用来把分散的字符串一块发送给对端，例如：

```php
$conn = new \Swoole\Coroutine\Socket(AF_INET, SOCK_STREAM, IPPROTO_IP);
$conn->connect('127.0.0.1', 9501);
$data1 = 'hello';
$data2 = 'world';
$ret = $conn->writeVector([$data1, $data2]);
```

那么，`writeVector`就会`hello`和`world`一块发送给对端。那在没有`writeVector`之前，我们是如何发送这两个字符串的呢？

我们有以下两种方式。

方式一：

```php
$conn = new \Swoole\Coroutine\Socket(AF_INET, SOCK_STREAM, IPPROTO_IP);
$conn->connect('127.0.0.1', 9501);
$data1 = 'hello';
$data2 = 'world';

$data3 = $data1 . $data2;
$ret = $conn->send($data3);
```

方式二：

```php
$conn = new \Swoole\Coroutine\Socket(AF_INET, SOCK_STREAM, IPPROTO_IP);
$conn->connect('127.0.0.1', 9501);
$data1 = 'hello';
$data2 = 'world';

$ret = $conn->send($data1);
$ret = $conn->send($data2);
```

方式一和方式二都会有它特有的性能问题。

其中，方式一的思路是，只想调用一次`send`方法，所以，就先拼接两个字符串。但是，这样会产生内存拷贝。拷贝的字节数是`strlen($data1) + strlen(data2)`。具体怎么拷贝的，我们可以去查看`PHP`内核`ZEND_CONCAT`对应的`handler`。

方式二的思路是，调用两次`send`方法来发送`$data1`和`$data2`。

因为，`send`方法只是把字符串从我们的应用空间拷贝到内核空间，不会立马发送字符串给对端（意味着两次`send`实际上只有一次网络发包的时间），所以，方式一和方式二实际上是在系统调用时间和拷贝字节数之间做权衡。

所以，我们需要`writeVector`这么一个方法，直接把分散在多个地方的字符串，一块发送出去。这样，应用层不存在字符串拼接，也只需要一次系统调用就行了。

## readVector

> 该方法在\Swoole\Coroutine\Socket类里面

`readVector`和`writeVector`的优化目的是一样的。

使用方法如下：

```php
$conn = new \Swoole\Coroutine\Socket(AF_INET, SOCK_STREAM, IPPROTO_IP);
$conn->connect('127.0.0.1', 9501);
$ret = $conn->readVector([5, 5]);
```

如果对端发来了`helloworld`，那么，`$ret`就会得到对应的数组`['hello', 'world']`。

那么，在没有`readVector`之前，我们也有两种方式来读取并分成两个字符串。

方式一：

```php
$conn = new \Swoole\Coroutine\Socket(AF_INET, SOCK_STREAM, IPPROTO_IP);
$conn->connect('127.0.0.1', 9501);
$data = $conn->recv(5 + 5);

$data1 = substr($data, 0, 5);
$data2 = substr($data, 5);
```

此时，我们通过一次系统调用来把字符串读取出来。但是，如果我们要分开来拿到这两个字符串，就需要调用`substr`来进行字符串截取了。此时，也会发生内存拷贝，拷贝的总字节数是`strlen($data1) + strlen($data2)`。

方式二：

```php
$conn = new \Swoole\Coroutine\Socket(AF_INET, SOCK_STREAM, IPPROTO_IP);
$conn->connect('127.0.0.1', 9501);
$data1 = $conn->recv(5);
$data2 = $conn->recv(5);
```

此时，我们通过两次`recv`系统调用来读取出两个字符串。
