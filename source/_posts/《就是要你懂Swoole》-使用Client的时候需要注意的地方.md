---
title: 《就是要你懂Swoole》—使用Swoole\Client的时候需要注意的地方
date: 2019-05-10 10:52:46
tags:
- Swoole
- PHP
---

今早看Swoole修复了一个创建`Swoole\Client`的bug，我记得之前是没有的，应该是用C++重构Swoole的时候出现的。Issue在这里[#2568](https://github.com/swoole/swoole-src/issues/2568)，修复在这里 [[e9fb220](https://github.com/swoole/swoole-src/commit/e9fb220a1e835b967d643d88398d3d28246ea45a)]。

原来的代码如下：

```c++
if (cli->async)
    {
        client_callback *cb = (client_callback *) swoole_get_property(getThis(), 0);
        if (!cb)
        {
            swoole_php_fatal_error(E_ERROR, "no event callback function");
            RETURN_FALSE;
        }

        if (swSocket_is_stream(cli->type))
        {
            if (!cb->cache_onConnect.function_handler)
            {
                swoole_php_fatal_error(E_ERROR, "no 'onConnect' callback function");
                RETURN_FALSE;
            }
            if (!cb->cache_onError.function_handler)
            {
                swoole_php_fatal_error(E_ERROR, "no 'onError' callback function");
                RETURN_FALSE;
            }
            if (!cb->cache_onClose.function_handler)
            {
                swoole_php_fatal_error(E_ERROR, "no 'onClose' callback function");
                RETURN_FALSE;
            }
            cli->onConnect = client_onConnect;
            cli->onClose = client_onClose;
            cli->onError = client_onError;
            cli->onReceive = client_onReceive;
            cli->reactor_fdtype = PHP_SWOOLE_FD_STREAM_CLIENT;
            if (cb->cache_onBufferFull.function_handler)
            {
                cli->onBufferFull = client_onBufferFull;
            }
            if (cb->cache_onBufferEmpty.function_handler)
            {
                cli->onBufferEmpty = client_onBufferEmpty;
            }
        }
        else
        {
            if (!cb || !cb->cache_onReceive.function_handler)
            {
                swoole_php_fatal_error(E_ERROR, "no 'onReceive' callback function");
                RETURN_FALSE;
            }
            if (cb->cache_onConnect.function_handler)
            {
                cli->onConnect = client_onConnect;
            }
            if (cb->cache_onClose.function_handler)
            {
                cli->onClose = client_onClose;
            }
            if (cb->cache_onError.function_handler)
            {
                cli->onError = client_onError;
            }
            cli->onReceive = client_onReceive;
            cli->reactor_fdtype = PHP_SWOOLE_FD_DGRAM_CLIENT;
        }

        zval *zobject = getThis();
        cli->object = zobject;
        sw_copy_to_stack(cli->object, cb->_object);
        Z_TRY_ADDREF_P(zobject);
    }
```

现在代码如下：

```c++
if (cli->async)
    {
        client_callback *cb = (client_callback *) swoole_get_property(getThis(), 0);
        if (!cb)
        {
            swoole_php_fatal_error(E_ERROR, "no event callback function");
            RETURN_FALSE;
        }
        if (!cb->cache_onReceive.function_handler)
        {
            swoole_php_fatal_error(E_ERROR, "no 'onReceive' callback function");
            RETURN_FALSE;
        }
        if (swSocket_is_stream(cli->type))
        {
            if (!cb->cache_onConnect.function_handler)
            {
                swoole_php_fatal_error(E_ERROR, "no 'onConnect' callback function");
                RETURN_FALSE;
            }
            if (!cb->cache_onError.function_handler)
            {
                swoole_php_fatal_error(E_ERROR, "no 'onError' callback function");
                RETURN_FALSE;
            }
            if (!cb->cache_onClose.function_handler)
            {
                swoole_php_fatal_error(E_ERROR, "no 'onClose' callback function");
                RETURN_FALSE;
            }
            cli->onConnect = client_onConnect;
            cli->onClose = client_onClose;
            cli->onError = client_onError;
            cli->onReceive = client_onReceive;
            cli->reactor_fdtype = PHP_SWOOLE_FD_STREAM_CLIENT;
            if (cb->cache_onBufferFull.function_handler)
            {
                cli->onBufferFull = client_onBufferFull;
            }
            if (cb->cache_onBufferEmpty.function_handler)
            {
                cli->onBufferEmpty = client_onBufferEmpty;
            }
        }
        else
        {
            if (cb->cache_onConnect.function_handler)
            {
                cli->onConnect = client_onConnect;
            }
            if (cb->cache_onClose.function_handler)
            {
                cli->onClose = client_onClose;
            }
            if (cb->cache_onError.function_handler)
            {
                cli->onError = client_onError;
            }
            cli->onReceive = client_onReceive;
            cli->reactor_fdtype = PHP_SWOOLE_FD_DGRAM_CLIENT;
        }

        zval *zobject = getThis();
        cli->object = zobject;
        sw_copy_to_stack(cli->object, cb->_object);
        Z_TRY_ADDREF_P(zobject);
    }
```

之前是如果`socket`不是`stream`类型的时候才会判断是否定义了`onReceive`回调函数。现在把对是否定义了`onReceive`的判断放在了

```c++
if (swSocket_is_stream(cli->type))
```

的上面，因此就变成了无论是不是`stream`类型的，都需要定义`onReceive`回调函数。

另外，`Swoole`创建的`Client`如果是`Stream`类型的，那么必须实现`onConnect`、`onReceive`、`onError`、`onClose`回调函数函数。如果不是`Stream`类型的，则可以不实现。当然，实现了这些回调函数也是可以的。从最近的一次修复来看。必须实现`onReceive`回调函数。（以上是在调用`client->connect`的时候进行判断的）