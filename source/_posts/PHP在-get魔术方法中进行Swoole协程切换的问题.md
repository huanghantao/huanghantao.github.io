---
title: PHP在__get魔术方法中进行Swoole协程切换的问题
date: 2020-08-09 16:57:51
tags:
- PHP
- Swoole
---

我们有如下代码：

```php
<?php

use Swoole\Coroutine;

class Foo
{
    public function __get($name)
    {
        Coroutine::sleep(1);
    }
}


$foo = new Foo;

for ($i=0; $i < 2; $i++) {
    go(function () use ($foo) {
        $foo->aaa;
    });
}
```

执行结果如下：

```php
[root@e2a14c00e7f6 get]# php get.php
PHP Notice:  Undefined property: Foo::$aaa in /root/codeDir/phpCode/swoole/coroutine/get/get.php on line 18

Notice: Undefined property: Foo::$aaa in /root/codeDir/phpCode/swoole/coroutine/get/get.php on line 18
[root@e2a14c00e7f6 get]#
```

我们会发现，这里会有警告，说是使用了没有定义的属性。

为了理解这个问题，我们可以先来了解一下`__get`这个魔术方法。首先是这么一段代码：

```php
<?php

use Swoole\Coroutine;

class Foo
{
    public function __get($name)
    {
        var_dump($name);
    }
}


$foo = new Foo;
$foo->aaa;
```

执行结果如下：

```bash
[root@e2a14c00e7f6 get]# php get.php
string(3) "aaa"
[root@e2a14c00e7f6 get]#
```

我们发现，因为`aaa`这个属性是类`Foo`的动态属性，所以默认会去调用`Foo`类的`__get`魔术方法，并且，传递给魔术方法的参数就是这个动态属性的名字。

好的，我们现在再来写一段代码：

```php
<?php

use Swoole\Coroutine;

class Foo
{
    public function __get($name)
    {
        $this->$name;
    }
}


$foo = new Foo;
$foo->aaa;
```

执行结果如下：

```bash
[root@e2a14c00e7f6 get]# php get.php
PHP Notice:  Undefined property: Foo::$aaa in /root/codeDir/phpCode/swoole/coroutine/get/get.php on line 9

Notice: Undefined property: Foo::$aaa in /root/codeDir/phpCode/swoole/coroutine/get/get.php on line 9
[root@e2a14c00e7f6 get]#
```

我们发现，当我们在`__get`魔术方法里面去读取动态属性`aaa`的时候，报的错误和我们在`__get`方法中进行协程切换是一模一样的。我们可以来理解一下为什么`PHP`要这么警告？

如果不这么警告的话，那么我们在`__get`函数里面读取动态属性`aaa`是不是又会继续调用`__get`魔术方法呢？那么这就会导致无限的递归了。所以`PHP`是禁止这种行为的。

那么，`PHP`底层是如何做到这种限制的呢？我们引用[《PHP7内核剖析》](https://github.com/pangudashu/php7-internal/blob/40645cfe087b373c80738881911ae3b178818f11/3/zend_object.md)的内容来解释一下：

```txt
Note: 如果类存在 get () 方法，则在实例化对象分配属性内存 (即:properties_table) 时会多分配一个 zval，类型为 HashTable，每次调用 get ($var) 时会把输入的 $var 名称存入这个哈希表，这样做的目的是防止循环调用，举个例子：

public function __get($var) { return $this->$var; }

这种情况是调用 get () 时又访问了一个不存在的属性，也就是会在 get () 方法中递归调用，如果不对请求的 $var 作判断则将一直递归下去，所以在调用 get () 前首先会判断当前 $var 是不是已经在 get () 中了，如果是则不会再调用 get ()，否则会把 $var 作为 key 插入那个 HashTable，然后将哈希值设置为：*guard |= IN_ISSET，调用完 get () 再把哈希值设置为：*guard &= ~IN_ISSET。

这个 HashTable 不仅仅是给 get () 用的，其它魔术方法也会用到，所以其哈希值类型是 zend_long，不同的魔术方法占不同的 bit 位；其次，并不是所有的对象都会额外分配这个 HashTable，在对象创建时会根据 zend_class_entry.ce_flags 是否包含 ZEND_ACC_USE_GUARDS 确定是否分配，在类编译时如果发现定义了 get()、set()、unset ()、__isset () 方法则会将 ce_flags 打上这个掩码。
```

所以，总结起来就是，当调用了`__get`时，会对这个动态属性做一个`IN_ISSET`标记，直到结束了这次`__get`调用，才会取消这个`IN_ISSET`标记。如果在有`IN_ISSET`的时候，再次对这个动态属性进行访问，那么就会报这个警告了。

所以，如下写法就是可以的了：

```php
<?php

use Swoole\Coroutine;

class Foo
{
    public function __get($name)
    {
        var_dump($name);
    }
}

$foo = new Foo;
$foo->aaa;
$foo->aaa;
```

执行结果如下：

```bash
[root@e2a14c00e7f6 get]# php get.php
string(3) "aaa"
string(3) "aaa"
[root@e2a14c00e7f6 get]#
```

因为我们是在退出第一次`__get`魔术方法调用之后再次访问动态属性`aaa`的，这个时候`IN_ISSET`标记已经没了。

好的，我们现在来修改一下之前的协程切换的代码：

```php
<?php

use Swoole\Coroutine;

class Foo
{
    public function __get($name)
    {
        Coroutine::sleep(1);
    }
}


$foo = new Foo;

go(function () use ($foo) {
    $foo->aaa;
});

go(function () use ($foo) {
    $foo->aaa;
});
```

这样或许会更加直观一点。首先，当第一个协程读取动态属性`aaa`的时候，对象`$foo`第一次调用了`__get`魔术方法。然后，因为`Coroutine::sleep`，协程被挂起了，`__get`魔术方法还没有退出，此时`IN_ISSET`标记还在。这个时候，轮到第二个协程进行动态属性`aaa`的读取，此时，因为`IN_ISSET`还在，所以此时访问动态属性`aaa`就是禁止的了：

```bash
[root@e2a14c00e7f6 get]# php get.php
PHP Notice:  Undefined property: Foo::$aaa in /root/codeDir/phpCode/swoole/coroutine/get/get.php on line 21

Notice: Undefined property: Foo::$aaa in /root/codeDir/phpCode/swoole/coroutine/get/get.php on line 21
[root@e2a14c00e7f6 get]#
```

我们发现报错的地方是第`21`行，是第二个协程报出的警告。
