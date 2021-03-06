---
title: PHP扩展开发中如何处理错误码
date: 2020-06-28 15:25:32
tags:
- PHP
---

> `PHP`扩展开发中，一种高效的开发方式是，库的内核层和`PHP`的`API`层分离。例如`Swoole`，和`PHP`相关的`API`层放在了项目的根目录里面，和`Swoole`内核有关的代码放在了`src`目录里面。

我们在开发扩展的时候，不可避免的要去抛出异常或者`Error`等错误。那么，我们如何去处理库的内核代码返回的错误码呢？比如说，我们有操作系统返回的错误码`ECHILD`、`EMFILE`等等。或者还有我们自己定义的一些操作不当的错误码，例如`EMISUSE`。

一般遵循如下规则：操作系统返回的`errno`（例如`EMFILE`）我们抛出的是`Exception`这种可捕获的错误，因为这是运行时的，不确定的错误（运行时错误，即不知道这么做会不会出问题）。还有一种是使用上的错误，我们希望抛出`Error`, 不希望被捕获（也就是说，你这样做了，一定会错，所以没必要抛异常让你捕获了，直接让程序崩掉），我们可以让库的内核代码返回自定义的错误码`EMISUSE`，方便上层（例如PHP扩展层面）识别。
