---
title: Swoole基础课程第三讲：如何正确搭建HTTP服务
date: 2019-10-22 08:10:19
tags:
- PHP
- Swoole
---

大部分的传统`FPM`项目性能瓶颈在于每次请求重新创建`ZendVM`的开销、IO阻塞导致的上下文频繁切换。`Swoole`解决的就是这类问题。

这篇文章教大家如何让`Swoole`的`HTTP`服务器性能达到最大。

压测脚本如下，机器的配置是单核、`2G`内存、`50G`硬盘：

```php
<?php

use Swoole\Http\Request;
use Swoole\Http\Response;

$process = new Swoole\Process(function (Swoole\Process $process) {
    $server = new Swoole\Http\Server('127.0.0.1', 9501, SWOOLE_BASE);
    $server->set([
        'log_file' => '/dev/null',
        'log_level' => SWOOLE_LOG_INFO,
        'worker_num' => swoole_cpu_num() * 2,
        // 'hook_flags' => SWOOLE_HOOK_ALL,
    ]);
    $server->on('workerStart', function () use ($process, $server) {
        $process->write('1');
    });
    $server->on('request', function (Request $request, Response $response) use ($server) {
        try {
            $redis = new Redis;
            $redis->connect('127.0.0.1', 6379);
            $greeter = $redis->get('greeter');
            if (!$greeter) {
                throw new RedisException('get data failed');
            }
            $response->end("<h1>{$greeter}</h1>");
        } catch (\Throwable $th) {
            $response->status(500);
            $response->end();
        }
    });
    $server->start();
});

if ($process->start()) {
    register_shutdown_function(function () use ($process) {
        $process::kill($process->pid);
        $process::wait();
    });
    $process->read(1);
    System('ab -c 256 -n 10000 -k http://127.0.0.1:9501/ 2>&1');
}
```

首先，我们创建了一个`Swoole\Process`对象，这个对象会开启一个子进程，在子进程中，我创建了一个`HTTP Server`，这个服务器是`BASE`模式的。除了`BASE`模式之外，还有一种`PROCESS`模式。在`PROCESS`模式下，套接字连接是在`Master`进程维持的，`Master`进程和`Worker`进程会多一层`IPC`通信的开销，但是，当`Worker`进程奔溃的时候，因为连接是在`Master`进程维持的，所以连接不会被断开。所以，`Process`模式适用于维护大量长连接的场景。`BASE`模式是在每个工作进程维持自己的连接，所以性能会比`Master`更好。并且，在`HTTP Server`下，`BASE`模式会更加的适用。

这里，我们将`worker_num`，也就是进程的数量设置为当前机器`CPU`核数的两倍。但是，在实际的项目中，我们需要不断的压测，来调整这个参数。

在`workerStart`的时候，也就是工作进程启动的时候，我们让子进程向管道中随意写入一个数据给父进程，父进程此时会读到一点数据，读到数据后，父进程才开始压测。

此时，压测的请求会进入`onRequest`回调。在这个回调中，我们创建了一个`Redis`客户端，这个客户端会连接`Redis`服务器，并请求一条数据。得到数据后，我们调用`end`方法来响应压测的请求。当错误时，我们返回一个错误码为`500`的响应。

在开始压测前，我们需要安装`Redis`扩展：

```shell
pecl install redis
```

然后`php.ini`配置中开启`redis`扩展即可。

我们还需要在`Redis`服务器里面插入一条数据：

```shell
127.0.0.1:6379> SET greeter swoole
OK
127.0.0.1:6379> GET greeter
"swoole"
127.0.0.1:6379>
```

`OK`，我们现在进行压测：

```shell
~/codeDir/phpCode/swoole/server # php server.php
Concurrency Level:      256
Time taken for tests:   2.293 seconds
Complete requests:      10000
Failed requests:        0
Keep-Alive requests:    10000
Total transferred:      1680000 bytes
HTML transferred:       150000 bytes
Requests per second:    4361.00 [#/sec] (mean)
Time per request:       58.702 [ms] (mean)
Time per request:       0.229 [ms] (mean, across all concurrent requests)
Transfer rate:          715.48 [Kbytes/sec] received
```

我们发现，现在的`QPS`比较低，只有`4361.00`。

因为，我们目前使用的`Redis`扩展是`PHP`官方的同步阻塞客户端，没有利用到协程（或者说异步的特性）。当进程去连接`Redis`服务器的时候，可能会阻塞整个进程，导致进程无法处理其他的连接，这样，这个`HTTP Server`处理请求的速度就不可能快。但是，这个压测结果会比`FPM`下好，因为`Swoole`是常驻进程的。

现在，我们来开启`Swoole`提供的`RuntimeHook`机制，也就是在运行时动态的将`PHP`同步阻塞的方法全部替换为异步非阻塞的协程调度的方法。我们只需要在`server->set`配置中加入一行即可：

```php
'hook_flags' => SWOOLE_HOOK_ALL
```

此时，我们再来运行这个脚本：

```shell
Concurrency Level:      256
Time taken for tests:   1.643 seconds
Complete requests:      10000
Failed requests:        0
Keep-Alive requests:    10000
Total transferred:      1680000 bytes
HTML transferred:       150000 bytes
Requests per second:    6086.22 [#/sec] (mean)
Time per request:       42.062 [ms] (mean)
Time per request:       0.164 [ms] (mean, across all concurrent requests)
Transfer rate:          998.52 [Kbytes/sec] received
```

我们发现，此时的`QPS`还是有一定的提升的。（这里，视频中压测的时候，会夯住请求，导致`QPS`非常的低，但是我实际测试的时候没有发生这个情况，估计是和`Redis`服务器本身对连接个数的的配置有关）

但是，为了避免请求数量过多，导致创建连接个数过多的问题，我们可以使用一个`Redis`连接池来解决。（同步阻塞是没有`Redis`连接过多的问题的，因为一旦`worker`进程阻塞住了，那么后面的请求就不会继续执行了，也就不会创建新的`Redis`连接了。因此，在同步阻塞的模式下，`Redis`的连接数量最大是`worker`进程的个数）

现在，我们来实现一下`Redis`连接池：

```php
class RedisQueue
{
    protected $pool;

    public function __construct()
    {
        $this->pool = new SplQueue;
    }

    public function get(): Redis
    {
        if ($this->pool->isEmpty()) {
            $redis = new \Redis();
            $redis->connect('127.0.0.1', 6379);
            return $redis;
        }
        return $this->pool->dequeue();
    }

    public function put(Redis $redis)
    {
        $this->pool->enqueue($redis);
    }

    public function close()
    {
        $this->pool = null;
    }
}
```

这里通过`spl`的队列实现的连接池。如果连接池中没有连接的时候，我们就新建一个连接，并且把创建的这个连接返回；如果连接池里面有连接，那么我们获取队列中前面的一个连接。当我们用完连接的时候，，就可以调用`put`方法归还连接。这样，我们就可以在一定程度上复用`Redis`的连接，缓解`Redis`服务器的压力，以及频繁创建`Redis`连接的开销也会降低。

我们现在使用这个连接池队列：

```php
<?php

use Swoole\Http\Request;
use Swoole\Http\Response;

$process = new Swoole\Process(function (Swoole\Process $process) {
    $server = new Swoole\Http\Server('127.0.0.1', 9501, SWOOLE_BASE);
    $server->set([
        'log_file' => '/dev/null',
        'log_level' => SWOOLE_LOG_INFO,
        'worker_num' => swoole_cpu_num() * 2,
        'hook_flags' => SWOOLE_HOOK_ALL,
    ]);
    $server->on('workerStart', function () use ($process, $server) {
        $server->pool = new RedisQueue;
        $process->write('1');
    });
    $server->on('request', function (Request $request, Response $response) use ($server) {
        try {
            $redis = $server->pool->get();
            // $redis = new Redis;
            // $redis->connect('127.0.0.1', 6379);
            $greeter = $redis->get('greeter');
            if (!$greeter) {
                throw new RedisException('get data failed');
            }
            $server->pool->put($redis);
            $response->end("<h1>{$greeter}</h1>");
        } catch (\Throwable $th) {
            $response->status(500);
            $response->end();
        }
    });
    $server->start();
});

if ($process->start()) {
    register_shutdown_function(function () use ($process) {
        $process::kill($process->pid);
        $process::wait();
    });
    $process->read(1);
    System('ab -c 256 -n 10000 -k http://127.0.0.1:9501/ 2>&1');
}

class RedisQueue
{
    protected $pool;

    public function __construct()
    {
        $this->pool = new SplQueue;
    }

    public function get(): Redis
    {
        if ($this->pool->isEmpty()) {
            $redis = new \Redis();
            $redis->connect('127.0.0.1', 6379);
            return $redis;
        }
        return $this->pool->dequeue();
    }

    public function put(Redis $redis)
    {
        $this->pool->enqueue($redis);
    }

    public function close()
    {
        $this->pool = null;
    }
}
```

我们在`worker`进程初始化的时候，创建了这个`RedisQueue`。然后在`onRequest`的阶段，从这个`RedisQueue`里面获取一个`Redis`连接。

现在，我们来进行压测：

```shell
Concurrency Level:      256
Time taken for tests:   1.188 seconds
Complete requests:      10000
Failed requests:        0
Keep-Alive requests:    10000
Total transferred:      1680000 bytes
HTML transferred:       150000 bytes
Requests per second:    8416.18 [#/sec] (mean)
Time per request:       30.418 [ms] (mean)
Time per request:       0.119 [ms] (mean, across all concurrent requests)
Transfer rate:          1380.78 [Kbytes/sec] received
```

`QPS`提升到了`8416.18`。

但是，通过`splqueue`实现的连接池是有缺陷的，因为这个队列是可以无限长的。这样，当并发量特别大的时候，还是会有可能创建非常多的连接，因为连接池里面可能始终都是空的。

这个时候，我们可以使用`Channel`来实现连接池。代码如下：

```php
class RedisPool
{
    protected $pool;

    public function __construct(int $size = 100)
    {
        $this->pool = new \Swoole\Coroutine\Channel($size);
        for ($i = 0; $i < $size; $i++)
        {
            while (true) {
                try {
                    $redis = new \Redis();
                    $redis->connect('127.0.0.1', 6379);
                    $this->put($redis);
                    break;
                } catch (\Throwable $th) {
                    usleep(1 * 1000);
                    continue;
                }
            }
        }
    }

    public function get(): \Redis
    {
        return $this->pool->pop();
    }

    public function put(\Redis $redis)
    {
        $this->pool->push($redis);
    }

    public function close()
    {
        $this->pool->close();
        $this->pool = null;
    }
}
```

可以看到，我们在这个构造方法中，将这个`Channel`的`size`设置为这个传入的参数。并且，创建`size`个连接。这些连接会在初始化连接池的时候就被创建，处于就就绪状态。这个有好处也有坏处，坏处就是在每个进程初始化的时候，就会占用一些连接，但是此时的进程并不会接收连接。好处就是提前创建好了`Redis`连接，这样服务器响应的延迟就会降低。

虽然，其他地方的代码其实和`RedisQueue`的实现一样。但是，底层是和`RedisQueue`大有不同的。因为当`Channel`里面没有`Redis`连接的时候，会让当前的协程挂起，让其他的协程继续被执行。等有协程把`Redis`连接还回到连接池里面的时候，这个被挂起的协程才会继续执行。这就是协程协作的原理。

现在，我们修改服务器的代码：

```php
<?php

use Swoole\Http\Request;
use Swoole\Http\Response;

$process = new Swoole\Process(function (Swoole\Process $process) {
    $server = new Swoole\Http\Server('127.0.0.1', 9501, SWOOLE_BASE);
    $server->set([
        'log_file' => '/dev/null',
        'log_level' => SWOOLE_LOG_INFO,
        'worker_num' => swoole_cpu_num() * 2,
        'hook_flags' => SWOOLE_HOOK_ALL,
    ]);
    $server->on('workerStart', function () use ($process, $server) {
        $server->pool = new RedisPool(64);
        $process->write('1');
    });
    $server->on('request', function (Request $request, Response $response) use ($server) {
        try {
            $redis = $server->pool->get();
            // $redis = new Redis;
            // $redis->connect('127.0.0.1', 6379);
            $greeter = $redis->get('greeter');
            if (!$greeter) {
                throw new RedisException('get data failed');
            }
            $server->pool->put($redis);
            $response->end("<h1>{$greeter}</h1>");
        } catch (\Throwable $th) {
            $response->status(500);
            $response->end();
        }
    });
    $server->start();
});

if ($process->start()) {
    register_shutdown_function(function () use ($process) {
        $process::kill($process->pid);
        $process::wait();
    });
    $process->read(1);
    System('ab -c 256 -n 10000 -k http://127.0.0.1:9501/ 2>&1');
}

class RedisPool
{
    protected $pool;

    public function __construct(int $size = 100)
    {
        $this->pool = new \Swoole\Coroutine\Channel($size);
        for ($i = 0; $i < $size; $i++)
        {
            while (true) {
                try {
                    $redis = new \Redis();
                    $redis->connect('127.0.0.1', 6379);
                    $this->put($redis);
                    break;
                } catch (\Throwable $th) {
                    usleep(1 * 1000);
                    continue;
                }
            }
        }
    }

    public function get(): \Redis
    {
        return $this->pool->pop();
    }

    public function put(\Redis $redis)
    {
        $this->pool->push($redis);
    }

    public function close()
    {
        $this->pool->close();
        $this->pool = null;
    }
}
```

只需要修改`workerStart`里面的部分即可，其他地方不需要做修改。这样，每个进程最多只能创建`64`个`Redis`连接。

我们继续压测：

```shell
Concurrency Level:      256
Time taken for tests:   0.817 seconds
Complete requests:      10000
Failed requests:        0
Keep-Alive requests:    10000
Total transferred:      1680000 bytes
HTML transferred:       150000 bytes
Requests per second:    12234.30 [#/sec] (mean)
Time per request:       20.925 [ms] (mean)
Time per request:       0.082 [ms] (mean, across all concurrent requests)
Transfer rate:          2007.19 [Kbytes/sec] received
```

我们发现`QPS`还是有所提升的，为什么我的`QPS`没有视频里面的提升明显呢？这个和测试环境有关。我自己的机器已经无法再提升了。就好像学霸最高只可以考`100`分一样的道理。（实际上，经过我的测试，如果我调整连接池的最大连接数，`QPS`会有所提升）

