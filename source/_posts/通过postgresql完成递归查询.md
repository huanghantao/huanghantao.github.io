---
title: 通过postgresql完成递归查询
date: 2019-08-26 15:37:48
tags:
- PostgreSQL
---

我们业务里有一张表存储的是图关系结构，今天，我们需要查询出和某个节点有联系的所有节点。图是有向图。

业务抽象出的表结构很简单：

```sql
create table if not exists tests(
   id serial not null,
   point1 int not null,
   point2 int not null,

   primary key(id)
);
```

其中，我们定义`point1`是起始点，`point2`是终点。举个例子：`(1, 2)`代表从`1`节点到`2`节点。

然后我们模拟一些测试数据：

```shell
insert into tests (point1, point2) values(1, 4),
(1, 5),
(4, 6),
(5, 6),
(2, 5),
(2, 7),
(7, 8),
(3, 9);
```

这些节点对应的图如下：

```shell
           1                            2                                      3
           +                            +                                      +
           |                            |                                      |
+----------+--------+                   |                                      |
|                   |                   |                                      |
|                   |                   |                                      |
|                   |                   |                                      |
v                   v                   |                                      |
4                   5<------------------+---------->7            9<------------+
+                   +                               +
+--------->6<-------+                         8<----+
```

也就是说，我这里产生了`9`个节点，`8`条边。

如果我想查询节点`1`有联系的其他节点，那么可以通过如下`SQL`语句查出来：

```shell
WITH RECURSIVE results AS
(
  SELECT
    id,
    point1,
    point2
  FROM  tests
  WHERE point1 = 1
  UNION ALL
  SELECT
    origin.id,
    origin.point1,
    origin.point2
  FROM results
  JOIN tests origin
  ON results.point2 = origin.point1
)
SELECT
  id,
  point1,
  point2
FROM results;
```

其中，

`WITH`被称为通用表表达式(`Common Table Expressions`)，作用是把复杂查询语句拆分成多个简单的部分。用法比较简单，大家可以搜一下这个的用法。

`RECURSIVE`修饰符来引入它自己，从而实现递归。

`WITH RECURSIVE`语句包含了非递归部分：

```sql
SELECT
id,
point1,
point2
FROM  tests
WHERE point1 = 1
```

和递归部分：

```sql
SELECT
origin.id,
origin.point1,
origin.point2
FROM results
JOIN tests origin
ON results.point2 = origin.point1
```

然后`UNION ALL`用来结合这两部分的结果，最终得到我们的图：

```sql
postgres=# WITH RECURSIVE results AS (   SELECT     id,     point1,     point2   FROM  tests   WHERE point1 = 1   UNION ALL   SELECT     origin.id,     origin.point1,     origin.point2   FROM results   JOIN tests origin   ON results.point2 = origin.point1 ) SELECT   id,   point1,   point2 FROM results;

 id | point1 | point2
----+--------+--------
  1 |      1 |      4
  2 |      1 |      5
  3 |      4 |      6
  4 |      5 |      6
(4 rows)
```

然后，我们也可以根据有向图的反方向进行搜索，`SQL`语句如下：

```sql
WITH RECURSIVE results AS
(
  SELECT
    id,
    point1,
    point2
  FROM  tests
  WHERE point2 = 6
  UNION ALL
  SELECT
    origin.id,
    origin.point1,
    origin.point2
  FROM results
  JOIN tests origin
  ON origin.point2 = results.point1
)
SELECT
  id,
  point2,
  point1
FROM results;
```

这样我们就会得到图：

```sql
postgres=# WITH RECURSIVE results AS (   SELECT     id,     point1,     point2   FROM  tests   WHERE point2 = 6   UNION ALL   SELECT     origin.id,     origin.point1,     origin.point2   FROM results   JOIN tests origin   ON origin.point2 = results.point1 ) SELECT   id,   point2,   point1 FROM results;

 id | point2 | point1
----+--------+--------
  3 |      6 |      4
  4 |      6 |      5
  1 |      4 |      1
  2 |      5 |      1
  5 |      5 |      2
(5 rows)
```

