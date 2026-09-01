---
title: PHP使用递归迭代器来遍历指定目录下的指定文件
date: 2020-07-17 18:59:56
tags:
- PHP
---

目录结构如下：

```bash
 hantaohuang@HantaodeMBP  ~/codeDir/phpCode/library/iterator  tree
.
├── dir1
│   ├── dir11
│   │   ├── file11.js
│   │   └── file11.php
│   ├── file1.php
│   └── file1.py
├── dir2
│   ├── file2.css
│   └── file2.php
└── iterator.php

3 directories, 7 files
 hantaohuang@HantaodeMBP  ~/codeDir/phpCode/library/iterator 
```

测试代码如下：

```php
<?php

$files = new RecursiveIteratorIterator(new RecursiveDirectoryIterator(__DIR__));
$files = new RegexIterator($files, '/\.php$/');

foreach ($files as $file) {
    /** @var SplFileInfo $file */
    var_dump($file);
}
```

执行结果如下：

```bash
hantaohuang@HantaodeMBP  ~/codeDir/phpCode/library/iterator  php iterator.php
object(SplFileInfo)#7 (2) {
  ["pathName":"SplFileInfo":private]=>
  string(66) "/Users/hantaohuang/codeDir/phpCode/library/iterator/dir2/file2.php"
  ["fileName":"SplFileInfo":private]=>
  string(9) "file2.php"
}
object(SplFileInfo)#10 (2) {
  ["pathName":"SplFileInfo":private]=>
  string(66) "/Users/hantaohuang/codeDir/phpCode/library/iterator/dir1/file1.php"
  ["fileName":"SplFileInfo":private]=>
  string(9) "file1.php"
}
object(SplFileInfo)#7 (2) {
  ["pathName":"SplFileInfo":private]=>
  string(73) "/Users/hantaohuang/codeDir/phpCode/library/iterator/dir1/dir11/file11.php"
  ["fileName":"SplFileInfo":private]=>
  string(10) "file11.php"
}
object(SplFileInfo)#10 (2) {
  ["pathName":"SplFileInfo":private]=>
  string(64) "/Users/hantaohuang/codeDir/phpCode/library/iterator/iterator.php"
  ["fileName":"SplFileInfo":private]=>
  string(12) "iterator.php"
}
hantaohuang@HantaodeMBP  ~/codeDir/phpCode/library/iterator 
```

用起来还是比较舒服的。
