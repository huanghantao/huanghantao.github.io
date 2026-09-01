---
title: 《就是要你懂swoole》-- 起步
date: 2018-07-27 22:41:12
tags:
- PHP
- swore
---

我想开一个swoole系列的文章。为什么？第一，我个人认为如果一个PHPer不懂swoole这个东西，就是计算机基础知识不扎实的体现。其次，因为很多PHPer以前没有接触过这方面的知识，所以学习起来会比较吃力，成本很大，所以，我希望能够尽我所能降低大家的学习成本。最后，因为是swoole让我知道了Unix网络编程，我很感激swoole。

emmm，小伙伴们无需担心自己的基础不够，因为我会讲的很细，细到操作系统底层原理。所以小伙伴们大可放心。而且我给出的代码例子都是经过我测试的，并且我会把运行结果以截图和gif的形式给大家展现，绝对不会只写代码不运行。

然后呢，我讲解知识的方式是按照官方文档来的，目的是过一遍官方文档，争取不落下任何一个知识点。

OK，今天这篇博客我教大家如何搭建swoole的环境。我的操作系统是MacOS，如果不是MacOS的也不要紧张，问题不是很大。

我选择的PHP版本是7.2.8，swoole版本是4.0.3。

## 安装PHP

（如果你之前没有安装swoole，那么尽量和我的环境保持一致）

从PHP官网下载源码，然后，按照下面的命令执行（执行命令的时候，如果遇到问题，自行百度解决）：

解压：

```
tar -xzf php-7.2.8.tar.gz
```

进入源码目录：

```
cd php-7.2.8
```

编译配置检测：

```
./configure --prefix=/usr/local/php7.2.8 --with-config-file-path=/usr/local/php7.2.8/etc --with-mcrypt=/usr/include --enable-mysqlnd --with-mysqli --with-pdo-mysql --enable-fpm --with-zlib --enable-xml --with-openssl --enable-pcntl --enable-sockets --enable-session --with-curl --enable-opcache
```

编译：

```
make
```

显示如下结果，说明编译成功：

![img](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E8%B5%B7%E6%AD%A5/1.png)

安装：

```
make install
```

成功安装完之后，你是可以在/usr/local目录下看到目录php7.2.8：

![img](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E8%B5%B7%E6%AD%A5/2.png)

## 把PHP加入到环境变量中

添加完之后，执行命令：

```
php -v
```

如果显示如下结果：

![img](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E8%B5%B7%E6%AD%A5/3.png)

说明添加成功。

## 安装swoole扩展

下载源码：

```
wget https://github.com/swoole/swoole-src/archive/v4.0.3.tar.gz
```

解压：

```
tar -xzf v4.0.3.tar.gz
```

进入源码目录：

```
cd swoole-src-4.0.3
```

执行命令phpize，生成编译检测脚本：

```
phpize
```

![img](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E8%B5%B7%E6%AD%A5/4.png)

编译配置检测：

```
./configure --with-php-config=/usr/local/php7.2.8/bin/php-config
```

编译：

```
make
```

安装：

```
make install
```

![img](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E8%B5%B7%E6%AD%A5/5.png)

我们可以在目录/usr/local/php7.2.8/lib/php/extensions/no-debug-non-zts-20170718/

中找到编译好的扩展：

![img](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E8%B5%B7%E6%AD%A5/6.png)

然后，我们需要在php.ini配置文件中添加这个扩展。首先，我们需要找到这个配置文件所在的路径：

```
php -i | grep php.ini
```

![img](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E8%B5%B7%E6%AD%A5/7.png)

可以看出，配置文件是要放在目录/usr/local/php7.2.8/etc中，我们去看看：

```
cd /usr/local/php7.2.8/etc
ls
```

![img](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E8%B5%B7%E6%AD%A5/8.png)

我们发现，并没有php.ini文件。不要担心，这种现象很正常，例如Mongodb你下载下来也是没有配置文件的。这说明需要我们自己手动创建这个配置文件，我们可以去PHP的源码目录下面拷贝一份配置文件：

```
cd /downloads/php-7.2.8
ls
```

![img](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E8%B5%B7%E6%AD%A5/9.png)

```
cp php.ini-development /usr/local/php7.2.8/etc
```

重新命名一下配置文件：

```
cd /usr/local/php7.2.8/etc
mv php.ini-development php.ini
```

![img](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E8%B5%B7%E6%AD%A5/10.png)

然后，我们修改配置文件。找一块空白的地方，添加上：

```
[swoole]
extension=swoole.so
```

![img](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E8%B5%B7%E6%AD%A5/11.png)

## 判断swoole是否安装成功

```
php -m | grep swoole
```

如果看到了swoole，那么，安装成功：

![img](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E8%B5%B7%E6%AD%A5/12.png)

到这里，我们的环境算是搭建起来了。

如果你还是担心没安装起来，在终端执行命令：

```
php -r "new swoole_server('0.0.0.0', 9501, SWOOLE_BASE, SWOOLE_SOCK_TCP);"
```

如果没有任何报错，那么说明安装成功：

![img](http://oklbfi1yj.bkt.clouddn.com/%E3%80%8A%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82swoole%E3%80%8B--%20%E8%B5%B7%E6%AD%A5/13.png)