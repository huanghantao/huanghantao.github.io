---
title: PHP8官方注解实战
date: 2020-06-09 19:05:04
tags:
- PHP
---

`PHP8`在前几天开始支持注解了，我们可以体验下，[RFC在这里](https://wiki.php.net/rfc/attributes_v2)。

注解的使用在很多的`PHP`框架里面已经具备了，所以我们不对注解做过多的解释，我们直接通过一个例子来看看注解可以怎么玩。

我们打算通过注解来实现一个功能：只要这个类被打上了`Bean`注解，那么我们就需要对这个类进行实例化，并且放在`DI`容器里面。

首先创建一个`beans.php`文件：

```php
<?php

namespace App;

<<\Bean>>
class Foo
{
    <<Value(1.1)>>
    public $x;

    <<Value(1.2)>>
    public $y;
}

<<\Bean>>
class Bar
{
    <<Value('2')>>
    public $x;
}

<<\Bean>>
class Tar
{
    <<Value(3)>>
    public $x;
}
```

这里有一个地方和民间版本的注解[doctrine/annotations](https://github.com/doctrine/annotations)有点区别。在民间版本里面，注解是写在`PHP`注释里面的，而官方支持的注解直接定义了新的语法（即注解写在了`<< >>`里面）。

然后实现扫描的功能：

```php
<?php

require 'beans.php';

$classes = get_declared_classes();

$container = [];

$scanNamespace = 'App';
$scanAnno = 'Bean';

foreach ($classes as $key => $class) {
    if (str_contains($class, $scanNamespace)) {
        $refClass = new \ReflectionClass($class);
        $classAttrs = $refClass->getAttributes();
        foreach ($classAttrs as $key => $classAttr) {
            $value = $classAttr->getName();
            if ($value === $scanAnno) {
                $refProperties =  $refClass->getProperties();
                $obj = $refClass->newInstance();
                foreach ($refProperties as $key => $refProperty) {
                    $refProperty = $refClass->getProperty($refProperty->getName());
                    $propertyAttrs = $refProperty->getAttributes();
                    $value = $propertyAttrs[0]->getArguments();
                    $refProperty->setValue($obj, $value[0]);
                    $container[$class] = $obj;
                }
            }
        }
    }
}

var_dump($container);
```

执行结果如下：

```bash
array(3) {
  ["App\Foo"]=>
  object(App\Foo)#5 (2) {
    ["x"]=>
    float(1.1)
    ["y"]=>
    float(1.2)
  }
  ["App\Bar"]=>
  object(App\Bar)#4 (1) {
    ["x"]=>
    string(1) "2"
  }
  ["App\Tar"]=>
  object(App\Tar)#1 (1) {
    ["x"]=>
    int(3)
  }
}
```

我们可以测试一下，把`Tar`类的`Bean`注解删除，那么就不会对这个类进行实例化：

```php
<?php

namespace App;

<<\Bean>>
class Foo
{
    <<Value(1.1)>>
    public $x;

    <<Value(1.2)>>
    public $y;
}

<<\Bean>>
class Bar
{
    <<Value('2')>>
    public $x;
}

class Tar
{
    <<Value(3)>>
    public $x;
}
```

执行结果：

```bash
array(2) {
  ["App\Foo"]=>
  object(App\Foo)#5 (2) {
    ["x"]=>
    float(1.1)
    ["y"]=>
    float(1.2)
  }
  ["App\Bar"]=>
  object(App\Bar)#4 (1) {
    ["x"]=>
    string(1) "2"
  }
}
```
