---
title: docker-php-ext-enable可以开启扩展的原理
date: 2018-12-21 21:36:44
tags:
- docker
---

（我们以docker镜像`php:7.2-cli-alpine`为例进行讲解）

我们知道，当我们编译完一个PHP扩展的时候，执行命令：

```shell
docker-php-ext-enable 扩展名
```

就可以开启这个扩展。起初我并没有觉得很奇怪，我以为肯定是在`php.ini`文件里面增加了一行：

```ini
extension=扩展名
```

但是，今天我由于某些原因想要去寻找这个容器里面的`php.ini`文件却发现并没有找到。于是我就很纳闷了。查看`docker-php-ext-enable`的源码才发现，原来它是这样开启扩展的：

```shell
ini="/usr/local/etc/php/conf.d/${iniName:-"docker-php-ext-$ext.ini"}"
if ! grep -q "$line" "$ini" 2>/dev/null; then
	echo "$line" >> "$ini"
fi
```

然后我就明白了，开启扩展的那一行是写在文件`/usr/local/etc/php/conf.d/docker-php-ext-扩展名.ini`这个配置文件里面的。查看命令`php -i`的输出，得到如下内容：

```shell
Scan this dir for additional .ini files => /usr/local/etc/php/conf.d
Additional .ini files parsed => /usr/local/etc/php/conf.d/docker-php-ext-mysqli.ini,
/usr/local/etc/php/conf.d/docker-php-ext-opcache.ini,
/usr/local/etc/php/conf.d/docker-php-ext-pcntl.ini,
/usr/local/etc/php/conf.d/docker-php-ext-pdo_mysql.ini,
/usr/local/etc/php/conf.d/docker-php-ext-sockets.ini,
/usr/local/etc/php/conf.d/docker-php-ext-sodium.ini,
/usr/local/etc/php/conf.d/docker-php-ext-tinyswoole.ini,
```

焕然大悟。

