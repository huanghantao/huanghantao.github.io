---
title: Swoole Library分析
date: 2020-02-07 14:40:43
tags:
- Swoole
---

> `Swoole`在`v4`版本后内置了`Library`模块，使用`PHP`代码编写内核功能，使得底层设施更加稳定可靠。

这篇文章我们就来分析一下如何使用以及编写`Library`。

首先，在`Swoole v4`的早期版本，`swoole-src`下面是有一个`library`目录的，里面就存放了很多的`PHP`代码，也就是`Library`库的代码。在后续版本，这个`library`就独立出了一个[仓库](https://github.com/swoole/library)。

简单介绍了背景之后，我们就来分析下这个`Library`如何工作以及如何使用的。

## 如何把Library作为Swoole扩展的内置库

### eval

首先，我们要明白`Library`可以作为`Swoole`内置库的工作原理。我们先来看一段代码：

```php
<?php

echo "start\n";
$code = '
    class Library
    {
        public function func1() {
            echo "codinghuang\n";
        }
    }
';
eval($code);
(new Library)->func1();
echo "end\n";

```

执行结果如下：

```shell
start
codinghuang
end
```

我们发现，通过`eval`可以把`$code`对应的字符串作为代码来执行。`Swoole`也是使用了`eval`的能力，把`PHP`代码作为`Swoole`扩展的内置库。在`Swoole`扩展层面，就是调用了`zend_eval_stringl`来执行`PHP`代码的。

既然如此，`Swoole`肯定就需要读取写好的`PHP Library`，然后在扩展加载的时候，执行`zend_eval_stringl`。然后，这些类、函数、变量等等就生成了。

### Swoole如何读取到写好的Library代码

其实，`Library`代码都在文件`swoole-src/php_swoole_library.h`里面。我们其中一个例子：

```cpp
static const char* swoole_library_source_constants =
    "\n"
    "/**\n"
    " * This file is part of Swoole.\n"
    " *\n"
    " * @link     https://www.swoole.com\n"
    " * @contact  team@swoole.com\n"
    " * @license  https://github.com/swoole/library/blob/master/LICENSE\n"
    " */\n"
    "\n"
    "declare(strict_types=1);\n"
    "\n"
    "define('SWOOLE_LIBRARY', true);\n";
```

可以发现，这段`PHP`代码作为`C++`的字符串存在。然后，在函数`php_swoole_load_library`里面就调用了：

```cpp
zend::eval(swoole_library_source_constants, "@swoole-src/library/constants.php");
```

来执行这段`PHP`代码。这样，常量`SWOOLE_LIBRARY`就被定义了。我们在来看看其他的`Library`代码（因为太长，我省略了部分）：

```cpp
static const char* swoole_library_source_core_constant =
    "\n"
    "/**\n"
    " * This file is part of Swoole.\n"
    " *\n"
    " * @link     https://www.swoole.com\n"
    " * @contact  team@swoole.com\n"
    " * @license  https://github.com/swoole/library/blob/master/LICENSE\n"
    " */\n"
    "\n"
    "declare(strict_types=1);\n"
    "\n"
    "namespace Swoole;\n"
    "\n"
    "class Constant\n"
    "{\n"
    "    public const OPTION_BUFFER_INPUT_SIZE = 'buffer_input_size';\n"
    "\n"
    "    public const OPTION_BUFFER_OUTPUT_SIZE = 'buffer_output_size';\n"
    "\n"
    "    public const OPTION_MESSAGE_QUEUE_KEY = 'message_queue_key';\n"
    "\n"
    "    public const OPTION_BACKLOG = 'backlog';\n"
    "\n"
    "    public const OPTION_KERNEL_SOCKET_RECV_BUFFER_SIZE = 'kernel_socket_recv_buffer_size';\n"
    "\n"
    "    public const OPTION_KERNEL_SOCKET_SEND_BUFFER_SIZE = 'kernel_socket_send_buffer_size';\n"
    "\n"
    "    public const OPTION_TCP_DEFER_ACCEPT = 'tcp_defer_accept';\n"
    "\n"
    "    public const OPTION_OPEN_TCP_KEEPALIVE = 'open_tcp_keepalive';\n"
    "\n"
    "    public const OPTION_OPEN_HTTP_PROTOCOL = 'open_http_protocol';\n"
    "\n"
    "    public const OPTION_OPEN_WEBSOCKET_PROTOCOL = 'open_websocket_protocol';\n"
    "\n"
    "    public const OPTION_WEBSOCKET_SUBPROTOCOL = 'websocket_subprotocol';\n"
    "\n"
    "    public const OPTION_OPEN_WEBSOCKET_CLOSE_FRAME = 'open_websocket_close_frame';\n"
    "\n"
    "    public const OPTION_OPEN_HTTP2_PROTOCOL = 'open_http2_protocol';\n"
    "\n"
    "    public const OPTION_OPEN_REDIS_PROTOCOL = 'open_redis_protocol';\n"
    "\n"
    "    public const OPTION_TCP_KEEPIDLE = 'tcp_keepidle';\n"
    "\n"
    "    public const OPTION_TCP_KEEPINTERVAL = 'tcp_keepinterval';\n"
    "\n"
    "    public const OPTION_TCP_KEEPCOUNT = 'tcp_keepcount';\n"
    "\n"
    "    public const OPTION_TCP_FASTOPEN = 'tcp_fastopen';\n"
    "\n"
    "    public const OPTION_PACKAGE_BODY_START = 'package_body_start';\n"
    "\n"
    "    public const OPTION_SSL_CLIENT_CERT_FILE = 'ssl_client_cert_file';\n"
    "\n"
    "    public const OPTION_SSL_PREFER_SERVER_CIPHERS = 'ssl_prefer_server_ciphers';\n"
    "\n"
    "    public const OPTION_SSL_CIPHERS = 'ssl_ciphers';\n"
    "\n"
    "    public const OPTION_SSL_ECDH_CURVE = 'ssl_ecdh_curve';\n"
    "\n"
    "    public const OPTION_SSL_DHPARAM = 'ssl_dhparam';\n"
    "\n"
    "    public const OPTION_OPEN_SSL = 'open_ssl';\n"
    "\n"
    "    /* }}} OPTION */\n"
    "}\n";
```

我们发现，这里有太多的`PHP`常量了，并且常量对应的值很多都是`Swoole`的配置项这种，例如`buffer_input_size`。所以，这些`php_swoole_library.h`文件里面的`Library`肯定不是手写的。

实际上，`php_swoole_library.h`里面的`Library`代码是通过`swoole-src/remake_library.sh`工具自动生成的。

但是，如果你在比较新的`swoole-src`代码里面直接跑`remake_library.sh`脚本，是会报错的：

```shell
[root@64fa874bf7d4 swoole-src]# ./remake_library.sh
rm swoole.lo
rm php_swoole_library.h
sh: line 0: cd: /root/codeDir/cppCode/swoole-src/library: No such file or directory
sh: line 0: cd: /root/codeDir/cppCode/swoole-src/library: No such file or directory
❌ Unable to get commit id of library in [/root/codeDir/cppCode/swoole-src/library]
[root@64fa874bf7d4 swoole-src]#
```

因为`library`代码已经不在`swoole-src`下面了。

### 如何生成Library代码

所以，我们需要先准备好`library`代码，我们可以直接在`swoole-src`下面`git clone`下`library`代码：

```shell
git clone git@github.com:swoole/library.git
```

然后，在目录`swoole-src`下执行`remake_library.sh`脚本即可把`library`的代码生成在`php_swoole_library.h`文件里面：

```shell
[root@64fa874bf7d4 swoole-src]# ./remake_library.sh
🚀🚀🚀Generated swoole php library successfully!🚀🚀🚀
remake...
done
[root@64fa874bf7d4 swoole-src]#
```

这样，就可以把`library`仓库最新的代码生成到`php_swoole_library.h`文件里面了。

### Swoole的那些常量如何编写的

我们发现，在`library`里面有一个文件`library/src/core/Constant.php`，这里面包含了很多`Swoole`内核的常量字符串，比如说：

```php
public const EVENT_RECEIVE = 'receive';
public const EVENT_CONNECT = 'connect';
public const OPTION_SSL_CERT_FILE = 'ssl_cert_file';
public const OPTION_SSL_KEY_FILE = 'ssl_key_file';
public const OPTION_BUFFER_INPUT_SIZE = 'buffer_input_size';
public const OPTION_BUFFER_OUTPUT_SIZE = 'buffer_output_size';
```

等等，这些要写起来是非常的繁琐，而且很容易漏了。并且，如果你在`Swoole`内核中修改了配置项，那么我们就需要在`Constant.php`里面去更新对应的常量，非常的麻烦。所以，官方提供了一个工具`swoole-src/tools/constant-generator.php`。只要我们跑这个`PHP`脚本，它就会通过正则匹配，把`Swoole`内核里面的那些常量字符串找出来，然后生成到文件`library/src/core/Constant.php`里面，大大的提高了工作效率。我们来演示一下：

```shell
[root@64fa874bf7d4 tools]# php constant-generator.php
🚀🚀🚀Constant generator successfully done!🚀🚀🚀
[root@64fa874bf7d4 tools]#
```

从这里我们发现，`library`的代码除了手动编写的之外，还有部分是工具生成的。

（未完）
