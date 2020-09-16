---
title: 通过__call()来减少你的代码
date: 2019-03-19 17:01:35
tags:
- PHP
---

假设有这么一个场景：

```
已经定义好了一个类，然后，因为某些原因需要对这个类进行一个封装。此时，就会出现一个问题：编写大量的重复代码。
```

这样讲可能会有些抽象，现在通过代码来分析一下。

假设我们已经有了一个Redis类（需要安装redis扩展），此时我想对它进行一个封装（文件名：Redis.php）：

```php
<?php

namespace App;

class Redis
{
    use Singleton;

    private $redis;

    public function __construct()
    {
        if (!extension_loaded('redis')) {
            throw new \Exception('redis.so文件不存在');
        }
        try {
            $this->redis = new \Redis();
            $result = $this->redis->connect('127.0.0.1', 6379, 5);
        } catch (\Throwable $th) {
            throw new \Exception('redis服务异常');
        }

        if ($result === false) {
            throw new \Exception('连接redis服务器异常');
        }
    }
}
```

其中，`Singleton`如下：

```php
trait Singleton
{
    private static $instance;

    static function getInstance(...$args)
    {
        if(!isset(self::$instance)){
            self::$instance = new static(...$args);
        }
        return self::$instance;
    }
}
```

这个封装过后的Redis类很简单，就是在实例化它的时候，连接Redis服务器。并且通过`$redis`这个成员变量可以获取到这个连接。并且它是**单例**的。

OK，此时，我们来使用一下这个子类：

```php
<?php
    
    require 'Redis.php';

	$redis = new App\Redis();
```

此时，如果我要使用redis的set方法，就必须在这个封装后的Redis类中实现一个set方法：

```php
<?php

namespace App;

class Redis
{
    use Singleton;

    private $redis;

    public function __construct()
    {
        if (!extension_loaded('redis')) {
            throw new \Exception('redis.so文件不存在');
        }
        try {
            $this->redis = new \Redis();
            $result = $this->redis->connect('127.0.0.1', 6379, 5);
        } catch (\Throwable $th) {
            throw new \Exception('redis服务异常');
        }

        if ($result === false) {
            throw new \Exception('连接redis服务器异常');
        }
    }
    
    public function set($key, $value)
    {
        return $this->redis->set($key, $value);
    }
}
```

然后，我才能使用：

```php
<?php
    
    require 'Redis.php';

	$redis = new App\Redis();
	$redis->set('name', 'codinghuang');
```

然后，此时我需要使用redis的get方法，那么又需要在封装后的Redis类中实现get方法：

```php
<?php

namespace App;

class Redis
{
    use Singleton;

    private $redis;

    public function __construct()
    {
        if (!extension_loaded('redis')) {
            throw new \Exception('redis.so文件不存在');
        }
        try {
            $this->redis = new \Redis();
            $result = $this->redis->connect('127.0.0.1', 6379, 5);
        } catch (\Throwable $th) {
            throw new \Exception('redis服务异常');
        }

        if ($result === false) {
            throw new \Exception('连接redis服务器异常');
        }
    }
    
    public function set($key, $value)
    {
        return $this->redis->set($key, $value);
    }
    
    public function get($key)
    {
        return $this->redis->get($key);
    }
}
```

这样是不是很麻烦呢？每次使用一个`\Redis`提供的方法，我都需要在封装后的Redis类中实现它。这样会增加很多啥都没做的代码。

此时，我们可以通过魔术方法`__call`来减少这些啥都没做的代码：

```php
<?php

namespace App;

class Redis
{
    use Singleton;

    private $redis;

    public function __construct()
    {
        if (!extension_loaded('redis')) {
            throw new \Exception('redis.so文件不存在');
        }
        try {
            $this->redis = new \Redis();
            $result = $this->redis->connect('127.0.0.1', 6379, 5);
        } catch (\Throwable $th) {
            throw new \Exception('redis服务异常');
        }

        if ($result === false) {
            throw new \Exception('连接redis服务器异常');
        }
    }
    
    public function __call(string $methodName , array $args)
    {
        if (method_exists($this->redis, $methodName)) {
            return call_user_func_array(array($this->redis, $methodName), $args);
        }
    }
}
```

至于`__call`的作用是什么，PHP文档说的很详细了。