---
title: 使用B+树索引来模拟Hash索引
date: 2018-04-06 18:53:19
tags:
- MySQL
---

MySQL对于B+树索引的长度是有限制的。如果我们想要在一个很长字符串（例如`varchar(255)`）上进行查找，就得使用前缀索引。但是有时这样还是会使得索引列键值的可选择性很差。

例如：

```
abcd
abce
abcf
abcg
```

不管长度为1还是2还是3，都无法很好的筛选出数据。

要解决这个问题，可以使用Hash索引。但是，并不是所有的储存引擎都支持Hash索引。此时，我们可以**通过B+树索引来模拟Hash索引**。

举个例子：

这里有一张表：

```mysql
CREATE TABLE `article` (  
  `id` int NOT NULL AUTO_INCREMENT,
  `title` varchar(255) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_title` (`title`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

表中有一个字段是一个很长的字符串类型：

```mysql
`title` varchar(255) NOT NULL
```

OK，现在我们开始使用B+树索引来模拟Hash索引。

我们先来往表中插入一些数据：

```mysql
INSERT INTO article
(title)
VALUES
('STL源码剖析'),
('深入理解Nginx'),
('Linux高性能服务器编程'),
('Linux内核设计与实现'),
('PHP7内核剖析'),
('Linux多线程服务端编程'),
('Unix环境高级编程'),
('TCP/IP详解'),
('深度探索C++对象模型');
```

![](http://oklbfi1yj.bkt.clouddn.com/%E4%BD%BF%E7%94%A8B+%E6%A0%91%E7%B4%A2%E5%BC%95%E6%9D%A5%E6%A8%A1%E6%8B%9FHash%E7%B4%A2%E5%BC%95/1.png)

OK，因为InnoDB储存引擎是不能够人为的去建立Hash索引。所以我们需要在表中新加一个列：

```mysql
alter table article add title_md5 char(32);
```

然后为表article的每条记录增加字段`title_md5`的值：

```mysql
update article set title_md5 = md5(title);
```

![](http://oklbfi1yj.bkt.clouddn.com/%E4%BD%BF%E7%94%A8B+%E6%A0%91%E7%B4%A2%E5%BC%95%E6%9D%A5%E6%A8%A1%E6%8B%9FHash%E7%B4%A2%E5%BC%95/2.png)

因为我们要使用B+树索引来模拟Hash索引，所以，我们还需要为这个`title_md5`列建立一个索引：

```mysql
create index idx_md5 on article(title_md5);
```

![](http://oklbfi1yj.bkt.clouddn.com/%E4%BD%BF%E7%94%A8B+%E6%A0%91%E7%B4%A2%E5%BC%95%E6%9D%A5%E6%A8%A1%E6%8B%9FHash%E7%B4%A2%E5%BC%95/3.png)

接下来，使用这个B+树索引来模拟Hash索引：

```mysql
select * from article where title_md5 = md5('STL源码剖析');
```

结果：

![](http://oklbfi1yj.bkt.clouddn.com/%E4%BD%BF%E7%94%A8B+%E6%A0%91%E7%B4%A2%E5%BC%95%E6%9D%A5%E6%A8%A1%E6%8B%9FHash%E7%B4%A2%E5%BC%95/4.png)

然后我们再来使用：

```mysql
explain select * from article where title_md5 = md5('STL源码剖析')\G
```

看看结果：

![](http://oklbfi1yj.bkt.clouddn.com/%E4%BD%BF%E7%94%A8B+%E6%A0%91%E7%B4%A2%E5%BC%95%E6%9D%A5%E6%A8%A1%E6%8B%9FHash%E7%B4%A2%E5%BC%95/5.png)

但是，我们知道，哈希函数是会发生碰撞的，也就是说不同的key可能得到的Hash值相同。即，可能会得到多条数据。所以，我们还需要再增加一个条件：

```mysql
select * from article where title_md5 = md5('STL源码剖析') and title = 'STL源码剖析';
```

![](http://oklbfi1yj.bkt.clouddn.com/%E4%BD%BF%E7%94%A8B+%E6%A0%91%E7%B4%A2%E5%BC%95%E6%9D%A5%E6%A8%A1%E6%8B%9FHash%E7%B4%A2%E5%BC%95/6.png)

然后我们再来执行：

```mysql
explain select * from article where title_md5 = md5('STL源码剖析') and title = 'STL源码剖析'\G
```

结果：

![](http://oklbfi1yj.bkt.clouddn.com/%E4%BD%BF%E7%94%A8B+%E6%A0%91%E7%B4%A2%E5%BC%95%E6%9D%A5%E6%A8%A1%E6%8B%9FHash%E7%B4%A2%E5%BC%95/7.png)

我们会发现，在key这一行，是`idx_title`，也就是说，使用的索引不是`idx_md5`。其实我这里也有点纳闷，目前还不知道为什么不是使用`idx_md5`索引。希望知道的小伙伴能够告诉我一下！！

所以我想到的一个方式是为字段`title_md5`和`title`建立一个联合索引。

## 局限性

- 和Hash索引一样，只能处理键值的**全值匹配**查找

- 所使用的Hash函数决定着索引键的大小

  如果我们使用的Hash函数生成的Hash值太大，就会造成索引比较大的情况。所以我们需要选取合适的Hash函数，既不能使生成的Hash值太大，也不能造成太多的Hash冲突。

既要在`title_md5`上做过滤也要在`title`上做过滤的原因：

为了避免Hash冲突的出现。因为只对`title_md5`做过滤掉，那么很可能得不到我们真正想要的结果（因为不同的key可能得到的Hash值相同）。