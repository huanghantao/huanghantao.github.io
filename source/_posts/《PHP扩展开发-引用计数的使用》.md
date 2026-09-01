---
title: 《PHP扩展开发--引用计数的使用》
date: 2019-09-24 09:52:55
tags:
- PHP
- PHP扩展开发
- 引用计数
---

感谢`twosee`大佬的点拨。

这篇文章，我们来讲一下`PHP`扩展开发的常见问题，引用计数管理。因为现有的书籍有些老旧，跟不上`PHP`的发展，所以很有必要学习一下。

`PHP`版本是`7.3.5`。

我们来写一个测试扩展函数：

```c
ZEND_BEGIN_ARG_INFO_EX(arginfo_study_test, 0, 0, 1)
    ZEND_ARG_ARRAY_INFO(0, arr, 0)
ZEND_END_ARG_INFO()

PHP_FUNCTION(test)
{
    zval *arr;

    if(zend_parse_parameters(ZEND_NUM_ARGS(), "a", &arr) == FAILURE){
        RETURN_FALSE;
    }
    RETURN_ARR(Z_ARR_P(arr));
}
```

这个函数做的事情很简单，接收一个数组，然后直接返回回去。

(小伙伴们自己记得注册一下创建的这个测试函数)

`OK`。我们编译、安装扩展：

```shell
~/codeDir/cppCode/study # make clean ; make ; make install
----------------------------------------------------------------------

Build complete.
Don't forget to run 'make test'.

Installing shared extensions:     /usr/local/lib/php/extensions/no-debug-non-zts-20180731/
Installing header files:          /usr/local/include/php/
~/codeDir/cppCode/study #
```

然后编写测试脚本：

```php
<?php

$a = array("time" => time());
xdebug_debug_zval("a");
$b = test($a);
xdebug_debug_zval("a");
xdebug_debug_zval("b");

unset($a);

var_dump($b);
```

执行：

```php
~/codeDir/cppCode/study # php test.php
a: (refcount=1, is_ref=0)=array ('time' => (refcount=0, is_ref=0)=1569290383)
a: (refcount=1, is_ref=0)=array ('time' => (refcount=0, is_ref=0)=1569290383)
b: (refcount=1, is_ref=0)=array ('time' => (refcount=0, is_ref=0)=1569290383)
/root/codeDir/cppCode/study/test.php:11:
&array
~/codeDir/cppCode/study #
```

我们发现，没有打印出`$b`这个数组。出现了`bug`，我们分析一下。

因为`PHP`对于复杂类型，例如字符串、数组、对象都是通过引用计数来管理的，不会拷贝出一个副本。所以，当我们在`PHP`脚本传递一个数组进入`PHP`扩展的时候，实际上只是增加了这个数组的引用计数。我们可以用`gdb`来调试一下：

```shell
 34│ PHP_FUNCTION(test)
 35│ {
 36│     zval *arr;
 37│
 38│     if(zend_parse_parameters(ZEND_NUM_ARGS(), "a", &arr) == FAILURE){
 39│         RETURN_FALSE;
 40│     }
 41├───> RETURN_ARR(Z_ARR_P(arr));
 42│ }

(gdb) p *arr.value.arr
$3 = {gc = {refcount = 2, u = {type_info = 23}}, u = {v = {flags = 24 '\030', _unused = 0 '\000', nIteratorsCount = 0 '\000', _unused2 = 0 '\000'}, flags = 24}, nTableMask = 4294967280, arData = 0x7ffff76
66680, nNumUsed = 1, nNumOfElements = 1, nTableSize = 8, nInternalPointer = 0, nNextFreeElement = 0, pDestructor = 0x555555a46860 <zval_ptr_dtor>}
(gdb)
```

我们发现，在调用了`zend_parse_parameters`解析出`PHP`脚本传递过来的数组之后，这个数组的引用计数变成了`2`。这一步增加引用计数的操作是`PHP`底层自动帮我们做的。

然后，通过`xdebug`打印发现，从扩展里面返回数组到`PHP`脚本后，它的引用计数又变回了`1`。这一步也是`PHP`帮我们做的。

接下来，我们`unset`了`$a`，使得这个数组的引用计数变成了`0`。又因为`$b`和`$a`指向的是一个数组，所以，此时我们再使用`$b`，就会报错了。

以上`bug`会在动态生成数组时出现。如果我们初始的数组是一个不可变数组，那么，同样的代码是不会出现`bug`的。因为不可变数组的初始引用计数是`2`，而不是`1`。我们可以测试一下：

```php
<?php

$a = array("time" => 1111);
xdebug_debug_zval("a");
$b = test($a);
xdebug_debug_zval("a");
xdebug_debug_zval("b");

unset($a);

var_dump($b);
```

我们把`time`对应的`value`写死为`1111`。然后执行脚本：

```shell
~/codeDir/cppCode/study # php test.php
a: (refcount=2, is_ref=0)=array ('time' => (refcount=0, is_ref=0)=1111)
a: (refcount=2, is_ref=0)=array ('time' => (refcount=0, is_ref=0)=1111)
b: (refcount=2, is_ref=0)=array ('time' => (refcount=0, is_ref=0)=1111)
/root/codeDir/cppCode/study/test.php:11:
array(1) {
  'time' =>
  int(1111)
}
~/codeDir/cppCode/study #
```

我们发现，因为不可变数组初始的引用计数是`2`，当我们对`unset($a)`之后，它的引用计数变成了`1`。此时，这个数组还是可以用的，所以后面使用`$b`不会报错。

正是因为从`PHP`脚本传入到扩展，以及从扩展传出到`PHP`脚本的那个数组是同一个，所以，`$a`和`$b`都是指向同一个数组的，因此，我们需要在扩展层面手动为这个数组的引用计数`+1`。所以，扩展代码应该改为：

```c
PHP_FUNCTION(test)
{
    zval *arr;

    if(zend_parse_parameters(ZEND_NUM_ARGS(), "a", &arr) == FAILURE){
        RETURN_FALSE;
    }
    Z_TRY_ADDREF_P(arr);
    RETURN_ARR(Z_ARR_P(arr));
}
```

重新编译、安装扩展：

```shell
~/codeDir/cppCode/study # make clean ; make ; make install
----------------------------------------------------------------------

Build complete.
Don't forget to run 'make test'.

Installing shared extensions:     /usr/local/lib/php/extensions/no-debug-non-zts-20180731/
Installing header files:          /usr/local/include/php/
~/codeDir/cppCode/study #
```

测试脚本为：

```php
<?php

$a = array("time" => time());
xdebug_debug_zval("a");
$b = test($a);
xdebug_debug_zval("a");
xdebug_debug_zval("b");

unset($a);

var_dump($b);
```

指向脚本：

```shell
~/codeDir/cppCode/study # php test.php
a: (refcount=1, is_ref=0)=array ('time' => (refcount=0, is_ref=0)=1569291353)
a: (refcount=2, is_ref=0)=array ('time' => (refcount=0, is_ref=0)=1569291353)
b: (refcount=2, is_ref=0)=array ('time' => (refcount=0, is_ref=0)=1569291353)
/root/codeDir/cppCode/study/test.php:11:
array(1) {
  'time' =>
  int(1569291353)
}
~/codeDir/cppCode/study #
```

我们发现，从扩展传回到`PHP`的时候，这个数组的引用计数变成了`2`。因此，当我们`unset($a)`之后，这个数组的引用计数变成了`1`。此时，我们再次使用`$b`就不会出错了。

