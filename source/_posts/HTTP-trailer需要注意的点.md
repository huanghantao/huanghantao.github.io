---
title: HTTP trailer需要注意的点
date: 2020-07-31 23:36:03
tags:
- HTTP
---

`HTTP trailer`的例子如下：

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked
Trailer: Expires

7\r\n
Mozilla\r\n
9\r\n
Developer\r\n
7\r\n
Network\r\n
0\r\n
Expires: Wed, 21 Oct 2015 07:28:00 GMT\r\n
\r\n
```

`HTTP trailer`有如下需要注意的点：

1. `Transfer-Encoding`得是`chunked`

2. 最后一个`chunk`是`0\r\n`，只有`Transfer-Encoding: chunked`而没有`Trailer`的最后一个`chunk`是`0\r\n\r\n`

3. `trailer`的内容得在发完所有的`http body`之后附加
