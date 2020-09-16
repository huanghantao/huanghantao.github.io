---
title: PHP使用json_encode正确的处理浮点数负零的情况
date: 2020-07-29 00:27:24
tags:
- PHP
---

例如这段代码：

```php
<?php

$array = [-0.0, 0.0];
var_dump($array);
$json = json_encode($array);
var_dump($json);
$array = json_decode($json);
var_dump($array);
```

执行结果如下：

```php
array(2) {
  [0]=>
  float(-0)
  [1]=>
  float(0)
}
string(6) "[-0,0]"
array(2) {
  [0]=>
  int(0)
  [1]=>
  int(0)
}
```

我们发现，先`json_encode`再`json_decode`是不能够还原的。而且`value`也从`float`变成了`int`。这是因为我们在`json_encode`没有去保留`ZERO FRACTION`。所以，正确的做法应该是这样的：

```php
<?php

$array = [-0.0, 0.0];
var_dump($array);
$json = json_encode($array, JSON_PRESERVE_ZERO_FRACTION);
var_dump($json);
$array = json_decode($json);
var_dump($array);
```

执行结果如下：

```php
array(2) {
  [0]=>
  float(-0)
  [1]=>
  float(0)
}
string(10) "[-0.0,0.0]"
array(2) {
  [0]=>
  float(-0)
  [1]=>
  float(0)
}
```

我们在`encode`的时候加上`JSON_PRESERVE_ZERO_FRACTION`，就会让得到的`json`字符串保留浮点数的`.0`，这样在`decode`的时候，就可以顺利的还原了。
