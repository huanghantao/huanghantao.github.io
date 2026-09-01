---
title: PHP内核pemalloc的persistent硬编码为1，我们该如何去进行释放
date: 2020-06-06 15:49:51
tags:
- PHP
---

举个例子，在`PHP`内核初始化`interned string`的时候，`PHP`内核使用的是`pemalloc`，并且`persistent`设置为了`1`：

```cpp
ZEND_API void zend_interned_strings_init(void)
{
	// 省略其他代码
	/* known strings */
	zend_known_strings = pemalloc(sizeof(zend_string*) * ((sizeof(known_strings) / sizeof(known_strings[0]) - 1)), 1);
	for (i = 0; i < (sizeof(known_strings) / sizeof(known_strings[0])) - 1; i++) {
		str = zend_string_init(known_strings[i], strlen(known_strings[i]), 1);
		zend_known_strings[i] = zend_new_interned_string_permanent(str);
	}
}
```

我们看到`zend_known_strings`这个地方是通过`pemalloc`来分配内存的，并且`persistent`设置为`1`了。但是释放的时候，我们发现用的是`free`，而不是`pefree`：

```cpp
ZEND_API void zend_interned_strings_dtor(void)
{
	zend_hash_destroy(&interned_strings_permanent);

	free(zend_known_strings);
	zend_known_strings = NULL;
}
```

实际上，当`persistent`为`1`的时候，`pemalloc`就是`malloc`。所以，这里可以用`free`来进行释放。但是，当时我觉得，为了对应`pemalloc`，释放的时候用`pefree`会更好一点。于是我改成了`pefree(zend_known_strings, 1)`，但是，`nikic`给我的解释如下：

```
通常，只有在动态传递persistent时才使用pefree()。

我们有时使用pemalloc(x, 1)和硬编码的persistent=1，但这是有原因的:pemalloc(x, 1)是一个“可靠的分配器”，永远不会返回NULL。因此，它与malloc(x)不同。
```

我们可以看一下`pemalloc`会做什么：

```cpp
#define pemalloc(size, persistent) ((persistent)?__zend_malloc(size):emalloc(size))

ZEND_API void * __zend_malloc(size_t len)
{
	void *tmp = malloc(len);
	if (EXPECTED(tmp || !len)) {
		return tmp;
	}
	zend_out_of_memory();
}

static ZEND_COLD ZEND_NORETURN void zend_out_of_memory(void)
{
	fprintf(stderr, "Out of memory\n");
	exit(1);
}
```

所以，当`pemalloc`的`persistent`为`1`的时候，实际上还是调用的`malloc`，但是会去判断是否成功的分配到了内存，如果没有分配到内存，直接退出进程。所以，我们可以认为，`pemalloc`的返回值一定不是`NULL`，如果是`NULL`，直接退出进程。

所以，我们可以得出一个结论，如果我们确定了内存一定是通过`malloc`分配的，那么，我们可以直接使用`free`来释放内存（用不用`pefree`都无所谓了）；但是如果不能够确定是不是`malloc`来分配的，那么，我们应该使用`pefree`来释放内存（此时`persistent`应该是一个变量，而不是硬编码）
