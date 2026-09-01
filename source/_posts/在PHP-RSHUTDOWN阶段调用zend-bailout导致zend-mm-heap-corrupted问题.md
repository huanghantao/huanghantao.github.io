---
title: 在PHP RSHUTDOWN阶段调用zend_bailout导致zend_mm_heap corrupted问题
date: 2021-01-07 18:01:50
tags:
- PHP
---

测试脚本如下：

```php
<?php

date_default_timezone_set('Asia/Shanghai');
```

并且开启`opcache`。

然后，我们编写如下代码：

```cpp
PHP_RSHUTDOWN_FUNCTION(yasd) {
    zend_bailout();

    return SUCCESS;
}
```

接着，使用`php-cgi`来启动服务：

```bash
php-cgi -b 0.0.0.0:8000
```

然后，请求两次我们的脚本。第一次是正常的，第二次就会出现`zend_mm_heap corrupted`的问题。

并且，我发现，关闭`opcache`之后，这个错误就会消失。当然，我们还是不要在`RSHUTDOWN`阶段去调用`zend_bailout`。
