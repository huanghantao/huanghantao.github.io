---
title: golang实现chunk方式的查询
date: 2019-09-17 14:10:42
tags:
---

有一个需求，是把表里面所有的数据都查询出来，并且生成`json`文件。因为一张表里面的数据很多，所以不可能一次性全部查询出来，所以需要用到`chunk`。之前用的`gorm`，但是发现`gorm`没有`chunk`方式的查询。如果要自己去实现这种操作，就需要去管理偏移量，而且还容易出现`bug`，所以就找了一个库，叫做`gorose`。用起来挺舒服的。

代码如下：

```go
package main

import (
    "fmt"

    "github.com/gohouse/gorose"
)

// User struct
// type User struct {
//  ID       int
//  UserName string
// }

const (
    dbHost     = "tcp(host.docker.internal:3306)"
    dbName     = "test"
    dbUser     = "root"
    dbPassword = "123456"
)

func main() {
    dsn := dbUser + ":" + dbPassword + "@" + dbHost + "/" + dbName + "?charset=utf8"
    var dbConfig = gorose.DbConfigSingle{
        Driver: "mysql",
        Dsn:    dsn,
    }

    connection, err := gorose.Open(&dbConfig)
    if err != nil {
        fmt.Println(err)
        return
    }

    session := connection.NewSession()

    user := session.Table("users")
    user.Fields("id", "username", "number").Chunk(2, func(data []map[string]interface{}) {
    fmt.Println(data)
    })
}

```

执行结果如下：

```shell
~/codeDir/golangCode/test # go run main.go
[map[id:1 username:a number:1] map[id:2 username:b number:2]]
[map[number:3 id:3 username:c] map[id:4 username:d number:4]]
[map[id:5 username:e number:5]]
~/codeDir/golangCode/test #
```

可以看出，每次都会查询出2条记录。

这个框架一个缺点就是文档不是很清楚，报错也有点不习惯。但是先用这个库解决一下`chunk`查询的问题吧。
