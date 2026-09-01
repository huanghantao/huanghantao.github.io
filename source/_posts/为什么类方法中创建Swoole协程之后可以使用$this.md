---
title: 为什么类方法中创建Swoole协程之后可以使用$this
date: 2020-10-15 11:09:12
tags:
- PHP
- Swoole
- Hyperf
---

前几天在写公司代码的时候，使用`Hyperf`写了大概这么一段代码：

```php
class IndexController
{
    public function test1()
    {
        Coroutine::create(function () {
            $this->request->getHeaders();
        });
    }
}
```

然后，在执行`$this->request->getHeaders();`这一行报错了，说是某某某接口没有实现。但是，如果我把这一行代码直接放在创建协程的外面，也就是这么写：

```php
class IndexController
{
    public function test1()
    {
        $this->request->getHeaders();
    }
}
```

就不会报错了。

具体要怎么解决这个问题不是我们这篇文章讨论的重点。这个问题一开始让我产生了一个疑问，以为不能在创建的子协程里面使用`$this`。

然后，我写了这么一段代码来进行测试：

```php
<?php

class Foo
{
    public function test1()
    {
        var_dump('foo1');
        go(function () {
            $this->test2();
        });
    }

    public function test2()
    {
        var_dump('foo2');
    }
}

$foo = new Foo;
$foo->test1();
```

输出结果如下：

```bash
string(4) "foo1"
string(4) "foo2"
```

发现，在子协程里面使用`$this`完全没有问题。

然后，回想到了编译原理的面向对象的语义特征里面作用域角度（我在我的[这篇博客](https://huanghantao.github.io/2020/09/07/面向对象的语义特征/)有讲解），我看了看`Swoole`的实现，果然发现有一部分代码是把`$this`复制到了子协程栈里面，实际上对应的就是`EX(This)`：

```cpp
call = zend_vm_stack_push_call_frame(
            ZEND_CALL_TOP_FUNCTION | ZEND_CALL_ALLOCATED, func, argc, fci_cache.called_scope, fci_cache.object);
```

如果`fci_cache.object`改成传`nullptr`的话，执行我们的脚本，就会报这个错误：

```bash
string(4) "foo1"
[1]    60865 segmentation fault  php test.php
```

说明，我们获取`$this`的时候出了问题。

实际上，`PHP`对`$this->test1()`的生成的`opcode`是`INIT_METHOD_CALL`，对应的`handler`是`ZEND_INIT_METHOD_CALL_SPEC_UNUSED_CONST_HANDLER`。我们一定会在这个`handler`里面看到从作用域里面去获取`$this`的过程。

如果我们传递的`fci_cache.object`是`nullptr`，意味着子协程作用域的`EX(This)`是`nullptr`，那么`EX(This)`对`foo1`的调用必然会段错误了。
