---
title: PHP8 Constructor parameter promotion
date: 2020-06-15 11:27:09
tags:
- PHP
---

`PHP8`增加了一个新的功能，叫做`Constructor parameter promotion`。它可以在初始化对象属性的时候帮我们省不少代码。在`PHP7`中，如果我们要初始化对象的属性，我们得这么写：

```php
class Point {
    public float $x;
    public float $y;
 
    public function __construct(
        float $x,
        float $y
    ) {
        $this->x = $x;
        $this->y = $y;
    }
}

var_dump(new Point(1, 2));
```

而现在，使用`PHP8`我们可以这么写：

```php
<?php

class Point {
    public function __construct(
        public float $x,
        public float $y
    ) {}
}

var_dump(new Point(1, 2));
```
