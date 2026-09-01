---
title: golang-migrate命令行的使用
date: 2019-06-17 14:00:57
tags:
- golang
- migrate
---

最近项目要用`Go`写。受`Laravel`的启发，想找一个`Go`语言的数据库版本迁移工具，然后就找到了一个大家用的比较多的工具：[migrate](https://github.com/golang-migrate/migrate/tree/master/cmd/migrate)。因为官方文档以及网上资源太少，所以这里写一篇文章记录下。

（安装这个工具的方法省略）

我们创建一个目录`migrations`，里面存放我们需要迁移的对象。

然后，创建迁移文件：

```shell
migrate create -ext sql -dir migrations create_users_table
```

执行这条命令会创建两个文件，分别是：

```
~/codeDir/golangCode/echo-demo # ls migrations/
20190617061102_create_users_table.down.sql  20190617061102_create_users_table.up.sql
```

`20190617061102`是和时间有关的一个标识，用来区分我们的`migration`的版本。

这两个文件都是空的，需要我们自己去填充。很明显，这是两个相反的`sql`操作。

我们在`20190617061102_create_users_table.up.sql`中写入创建`users`表的操作：

```sql
CREATE TABLE IF NOT EXISTS users(
   id INT,
   name VARCHAR(100) NOT NULL,
   password VARCHAR(40) NOT NULL,
   PRIMARY KEY ( id )
);
```

对应的我们就需要在`20190617061102_create_users_table.down.sql`中删除这张表：

```sql
DROP table IF EXISTS users;
```

然后，我们在终端执行命令：

```shell
migrate -verbose -source file://migrations -database postgres://postgres:postgres用户的密码@host.docker.internal:5432/postgres?sslmode=disable up 1
```

结果如下：

```shell
~/codeDir/golangCode/echo-demo # migrate -verbose -source file://migrations -database postgres://postgres:postgres用户的密码@host.docker.internal:5432/postgres?sslmode=disable up 1
2019/06/17 06:17:20 Start buffering 20190617061102/u create_users_table
2019/06/17 06:17:20 Read and execute 20190617061102/u create_users_table
2019/06/17 06:17:20 Finished 20190617061102/u create_users_table (read 11.0414ms, ran 12.0132ms)
2019/06/17 06:17:20 Finished after 28.7139ms
2019/06/17 06:17:20 Closing source and database
~/codeDir/golangCode/echo-demo # 
```

此时，我们看看数据库的表：

```sql
postgres=# \dt
               List of relations
 Schema |       Name        | Type  |  Owner   
--------+-------------------+-------+----------
 public | schema_migrations | table | postgres
 public | users             | table | postgres
(2 rows)

postgres=# 
```

发现，此时有两张表，分别是`schema_migrations`以及我们创建的`users`。其中，`schema_migrations`里面存放的是当前`migration`的版本以及状态。

我们来看看：

```sql
postgres=# select * from schema_migrations;
    version     | dirty 
----------------+-------
 20190617061102 | f
(1 row)

postgres=# 
```

此时，`migration`的版本是`20190617061102`，这个就是我们文件的前缀。`dirty`代表当前版本是干净的还是脏的。`f`表示干净（即没有出问题）。

然后，我来演示一下`migrate`失败的操作。

我们再创建一个`migration`文件：

```shell
migrate create -ext sql -dir migrations create_emails_table
```

```shell
~/codeDir/golangCode/echo-demo # ls migrations/
20190617061102_create_users_table.down.sql   20190617061102_create_users_table.up.sql     20190617062220_create_emails_table.down.sql  20190617062220_create_emails_table.up.sql
~/codeDir/golangCode/echo-demo # 
```

然后，在`up`的那个文件里面写下：

```sql
CREATE TABLE IF NOT EXISTS emails(
   id error INT,
   account VARCHAR(100) NOT NULL,
   PRIMARY KEY ( id )
);
```

（我们在`id`的那一行故意写错了，加了一个会导致报错的字符串`error`）

在`down`的那个文件里面写下：

```sql
DROP table IF EXISTS emails;
```

然后，执行迁移操作：

```shell
~/codeDir/golangCode/echo-demo # migrate -verbose -source file://migrations -database postgres://postgres:postgres用户的密码@host.docker.internal:5432/postgres?sslmode=disable up 1
2019/06/17 06:24:49 Start buffering 20190617062220/u create_emails_table
2019/06/17 06:24:49 Read and execute 20190617062220/u create_emails_table
2019/06/17 06:24:49 error: migration failed: syntax error at or near "INT" (column 13) in line 2: CREATE TABLE IF NOT EXISTS emails(
   id error INT,
   account VARCHAR(100) NOT NULL,
   PRIMARY KEY ( id )
); (details: pq: syntax error at or near "INT")
~/codeDir/golangCode/echo-demo # 
```

完美报错。

此时，我们把`error`这个字符串删除：

```sql
CREATE TABLE IF NOT EXISTS emails(
   id INT,
   account VARCHAR(100) NOT NULL,
   PRIMARY KEY ( id )
);
```

然后再执行迁移操作：

```shell
~/codeDir/golangCode/echo-demo # migrate -verbose -source file://migrations -database postgres://postgres:postgres用户的密码@host.docker.internal:5432/postgres?sslmode=disable up 1
2019/06/17 06:26:36 error: Dirty database version 20190617062220. Fix and force version.
~/codeDir/golangCode/echo-demo # 
```

（开始的时候，我是吃惊的，因为我按照`Laravel migrate`的思路，不应该会报错）

报错提示说`database version`是脏的。所以，我们看看`schema_migrations`表的内容：

```sql
postgres=# select * from schema_migrations;
    version     | dirty 
----------------+-------
 20190617062220 | t
(1 row)

postgres=# 
```

我们发现，尽管`migrate up`失败了，但是这里还是会`up`这个`version`。这是我用不习惯的一个地方。

此时，我们需要按照报错的提示：

```shell
Fix and force version.
```

我们已经把`error`字符串删除了，所以此时我们只需要`force version`一下：

```shell
~/codeDir/golangCode/echo-demo # migrate -verbose -source file://migrations -database postgres://postgres:postgres用户的密码@host.docker.internal:5432/postgres?sslmode=disable force 20190617062220
2019/06/17 06:32:30 Finished after 18.7548ms
2019/06/17 06:32:30 Closing source and database
```

`force`后面带上`dirty`的那个版本。

此时，我们再看看`schema_migrations`表的内容：

```sql
postgres=# select * from schema_migrations;
    version     | dirty 
----------------+-------
 20190617062220 | f
(1 row)

postgres=# 
```

`dirty`已经是`f`了，代表这是干净的了。

然后，我们再回退到上一个版本：

```shell
~/codeDir/golangCode/echo-demo # migrate -verbose -source file://migrations -database postgres://postgres:postgres用户的密码@host.docker.internal:5432/postgres?sslmode=disable down 1
2019/06/17 06:34:04 Start buffering 20190617062220/d create_emails_table
2019/06/17 06:34:04 Read and execute 20190617062220/d create_emails_table
2019/06/17 06:34:04 Finished 20190617062220/d create_emails_table (read 13.2303ms, ran 10.691ms)
2019/06/17 06:34:04 Finished after 30.1489ms
2019/06/17 06:34:04 Closing source and database
~/codeDir/golangCode/echo-demo # 
```

回退成功：

```sql
postgres=# select * from schema_migrations;
    version     | dirty 
----------------+-------
 20190617061102 | f
(1 row)

postgres=# 
```

然后，我们现在可以正常的执行`up`操作了：

```shell
~/codeDir/golangCode/echo-demo # migrate -verbose -source file://migrations -database postgres://postgres:postgres用户的密码@host.docker.internal:5432/postgres?sslmode=disable up 1
2019/06/17 06:35:15 Start buffering 20190617062220/u create_emails_table
2019/06/17 06:35:15 Read and execute 20190617062220/u create_emails_table
2019/06/17 06:35:15 Finished 20190617062220/u create_emails_table (read 12.715ms, ran 16.2358ms)
2019/06/17 06:35:15 Finished after 34.5774ms
2019/06/17 06:35:15 Closing source and database
~/codeDir/golangCode/echo-demo # 
```

```sql
postgres=# \dt
               List of relations
 Schema |       Name        | Type  |  Owner   
--------+-------------------+-------+----------
 public | emails            | table | postgres
 public | schema_migrations | table | postgres
 public | users             | table | postgres
(3 rows)

postgres=# 
```

`emails`表创建完成。

（感觉还是不如`Laravel`的`migrate`好用）

