---
title: 'PHP7中到底可不可以通过::来调用对象的非静态方法？'
date: 2019-04-01 21:58:49
tags:
- PHP
---

直接上代码：

```php
<?php

class Test
{
    public function say()
    {
        echo "hello world\n";
    }
}

$obj = new Test();
$obj::say();
```

结果如下：

```shell
~/codeDir/phpCode/test # php test.php 
hello world
```

奇怪，不是说好的`::`是用来调用类的静态方法的吗？为啥还可以调用类的非静态方法？

实际上，用`::`访问类的非静态方法不符合语法，只不过PHP默认没有开始对应的报错提示。

我们修改PHP的配置，增加如下配置：

```php
<?php

// error_reporting = E_ALL | E_STRICT
// display_errors = On

class Test
{
    public function say()
    {
        echo "hello world\n";
    }
}

ini_set('error_reporting', E_ALL | E_STRICT);
ini_set('display_errors', 'on');

$obj = new Test();
$obj::say();
```

结果如下：

```shell
~/codeDir/phpCode/test # php test.php 

Deprecated: Non-static method Test::say() should not be called statically in /root/codeDir/phpCode/test/test.php on line 15
hello world
```

可以发现报错了对吧。

所以，**在开发环境，应该设置最严格的报错的级别，生产环境，可以适当降低报错级别**。