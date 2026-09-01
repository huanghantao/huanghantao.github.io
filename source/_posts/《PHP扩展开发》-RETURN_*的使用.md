---
title: 《PHP扩展开发》--RETURN_*宏的使用
date: 2019-05-28 18:54:33
tags:
- PHP
- PHP扩展
---

以`RETURN_FALSE`宏为例，我们展开后得到：

```c
#define RETURN_FALSE  					{ RETVAL_FALSE; return; }
#define RETVAL_FALSE  					ZVAL_FALSE(return_value)
#define ZVAL_FALSE(z) do {				\
		Z_TYPE_INFO_P(z) = IS_FALSE;	\
	} while (0)
```

所以，`RETURN_FALSE`的作用就是把`return_value`这个扩展函数的返回值设置为`false`，然后再执行`C`语言的`return;`，从而跳出扩展函数。所以，`RETURN_FALSE`后面是不需要分号结尾的。（当然，写了也没事）