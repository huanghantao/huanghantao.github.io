---
title: 在PHP代码里面通过ini_set设置upload_max_filesize不生效
date: 2019-01-10 19:01:18
tags:
- PHP
---

今天在测试项目的时候，遇到了一个bug，如题所说的。经过排查发现，ini_set不是对所有的配置项都可以进行设置的。例如`upload_max_filesize`这个值。我们来进行一个测试：

```php
<?php

// test memory_limit
$value = ini_get('memory_limit');
echo 'old memory_limit value: ' . $value . PHP_EOL;

ini_set('memory_limit', '256M');
$value = ini_get('memory_limit');
echo 'new memory_limit value: ' . $value . PHP_EOL;

// test upload_max_filesize
$value = ini_get('upload_max_filesize');
echo 'old upload_max_filesize value: ' . $value . PHP_EOL;

ini_set('upload_max_filesize', '20M');
$value = ini_get('upload_max_filesize');
echo 'new upload_max_filesize value: ' . $value . PHP_EOL;

```

执行后输出：

```shell
old memory_limit value: 128M
new memory_limit value: 256M
old upload_max_filesize value: 2M
new upload_max_filesize value: 2M
```

可以发现`memory_limit`的值是可以在php脚本运行时候修改的，但是`upload_max_filesize`却不可以。所以，当我们遇到`ini_set`函数不生效的时候，我们需要去查一查这个配置值是否可以在脚本运行的时候进行动态的修改。这里我给出两个链接：

1、[ini list](http://www.php.net/manual/en/ini.list.php)

2、[configuration changes modes](http://php.net/manual/en/configuration.changes.modes.php)