---
title: PHP __debugInfo魔术方法
date: 2020-07-05 11:07:04
tags:
- PHP
---

> 该方法在`PHP 5.6.0`及其以上版本可以用

我们直接来看一段代码：

```php
<?php

class Foo
{
    public function __debugInfo()
    {
        return [
            'one' => 1,
            'two' => 2,
        ];
    }
}

var_dump(new Foo);
```

输出结果如下：

```bash
object(Foo)#1 (2) {
  ["one"]=>
  int(1)
  ["two"]=>
  int(2)
}
```

可以发现，使用`var_dump`的时候，会去调用`__debugInfo`魔术方法。这个方法对于调试还是比较有用，比如我们写`PHP`的`C`扩展，如果自定义了类对象，我们如果想要输出`struct`里面的信息，就可以去实现`__debugInfo`方法，然后在这个方法中去获取`struct`里面的信息，作为数组返回。
