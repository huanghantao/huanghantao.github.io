---
title: C语言通过宏来生成代码
date: 2020-06-28 10:36:53
tags:
- C语言
---

在`C`语言里面，没有`map`结构，只有一个简单的数组。因此，如果我们无法实现如下的映射：

```c
header["Content-Type"] = "text/html";
header["Connection"] = "close";
header["Host"] = "www.host.com";
```

我们只可以这样写：

```c
header[1] = "text/html";
header[2] = "close";
header[3] = "www.host.com";
```

但是，这样写可读性太差了。

好在`C`语言提供了宏，我们可以这么做：

```c
#define CONTENT_TYPE    1
#define CONNECTION      2
#define HOST            3

header[CONTENT_TYPE] = "text/html";
header[CONNECTION] = "close";
header[HOST] = "www.host.com";
```

但是这么做也有一个问题，就是`map`的`name`和`value`分散了。如果我们要修改或者增加新的`name value`，那么就容易搞错位置。

所以我们有如下技巧：

```c
#define HTTP_HEADER_MAP(XX) \
    XX(CONTENT_TYPE, "text/html") \
    XX(CONNECTION, "close") \
    XX(HOST, "www.host.com") \

#define HTTP_HEADER_VARS_GEN(name, value) char name[] = value;
    HTTP_HEADER_MAP(HTTP_HEADER_VARS_GEN)
#undef HTTP_HEADER_VARS_GEN
```

我们来给一个完整的例子：

```c
#include <stdio.h>

#define HTTP_HEADER_MAP(XX) \
    XX(CONTENT_TYPE, "text/html") \
    XX(CONNECTION, "close") \
    XX(HOST, "www.host.com") \

enum http_header_e
{
#define HTTP_HEADER_GEN(name, value) HTTP_HEADER_##name,
    HTTP_HEADER_MAP(HTTP_HEADER_GEN)
#undef HTTP_HEADER_GEN
};

int main(int argc, char const *argv[])
{
    char *header[3];
#define HTTP_HEADER_VARS_GEN(name, value) header[HTTP_HEADER_##name] = value;
    HTTP_HEADER_MAP(HTTP_HEADER_VARS_GEN)
#undef HTTP_HEADER_VARS_GEN

    printf("%s\n", header[HTTP_HEADER_CONTENT_TYPE]);
    printf("%s\n", header[HTTP_HEADER_CONNECTION]);
    printf("%s\n", header[HTTP_HEADER_HOST]);

    return 0;
}
```

这个技巧的大概思路是，我们通过宏定义一个伪`map`，即`HTTP_HEADER_MAP`。这个宏我们需要传递一个`XX`，而这个`XX`就根据我们的需求，来取`HTTP_HEADER_MAP`里面的内容。

例如，在`main`函数里面，我们定义了一个`HTTP_HEADER_VARS_GEN`来替换`XX`，而`HTTP_HEADER_VARS_GEN`就是去取`HTTP_HEADER_MAP`的东西，来初始化我们的`header`数组。

明白了这个思想之后，我们可以定一个新的宏来生成`printf`代码，从而继续简化我们的代码：

```c
#include <stdio.h>

#define HTTP_HEADER_MAP(XX) \
    XX(CONTENT_TYPE, "text/html") \
    XX(CONNECTION, "close") \
    XX(HOST, "www.host.com") \

enum http_header_e
{
#define HTTP_HEADER_GEN(name, value) HTTP_HEADER_##name,
    HTTP_HEADER_MAP(HTTP_HEADER_GEN)
#undef HTTP_HEADER_GEN
};

int main(int argc, char const *argv[])
{
    char *header[3];
#define HTTP_HEADER_VARS_GEN(name, value) header[HTTP_HEADER_##name] = value;
    HTTP_HEADER_MAP(HTTP_HEADER_VARS_GEN)
#undef HTTP_HEADER_VARS_GEN

#define HTTP_HEADER_PRINTF_GEN(name, value) printf("%s\n", header[HTTP_HEADER_##name]);
    HTTP_HEADER_MAP(HTTP_HEADER_PRINTF_GEN)
#undef HTTP_HEADER_PRINTF_GEN

    return 0;
}
```

可以发现，代码非常的简洁了。

总结一下套路：

> 1. 定义一个伪`map`
> 2. 定义一个`GEN`宏
> 3. 把这个`GEN`宏传递进伪`map`里面，从这个伪`map`里面取我们需要的内容
