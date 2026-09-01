---
title: gqlgen/graphql自定义标量
date: 2019-08-30 09:38:19
tags:
- Golang
- GraphQL
- gqlgen
---

昨天，我们在使用`gqlgen`的时候，发现它默认没有`int64`类型的标量，只有`int`类型的标量。所以需要自定义一个`int64`标量。

先说一下自定义标量的原理吧，这个在文档里面没有去解释，只是给出了一段代码，其他的就要自己去理解了。

自定义标量的原理就是，前端传递一个字符串，然后`gqlgen`会自动调用我们实现的解析函数，去解析这个字符串，得到我们想要的类型。

现在，我们来实战一下。我们初始化项目：

```shell
~/codeDir/golangCode # mkdir scalars ; cd scalars ; go mod init scalars
go: creating new go.mod: module scalars
~/codeDir/golangCode/scalars # 
```

这里，我们是通过`go module`这个包依赖管理工具来管理的。

因为`gqlgen`是先定义`schema`的然后再生成代码的，所以，我们需要先定义好我们的`schema`。我们先创建文件：

```shell
~/codeDir/golangCode/scalars # touch schema.graphql
```

内容如下：

```typescript
type Article {
  id: ID!
  text: String!
}
type Query {
  article: Article!
}
```

这里，我们简单的定义了一个类型`Article`和一个查询`article`。

`OK`，现在我们来生成一下我们的代码：

```shell
~/codeDir/golangCode/scalars # gqlgen init
Exec "go run ./server/server.go" to start GraphQL server
~/codeDir/golangCode/scalars # tree
.
├── generated.go
├── go.mod
├── go.sum
├── gqlgen.yml
├── models_gen.go
├── resolver.go
├── schema.graphql
└── server
    └── server.go

1 directory, 8 files
~/codeDir/golangCode/scalars # 
```

我们可以在`resolver.go`的`Article`函数里面写我们的查询，这里，我们简单的返回一条记录即可：

```go
func (r *queryResolver) Article(ctx context.Context) (*Article, error) {
	return &Article{
		ID:   "1",
		Text: "I am codinghuang",
	}, nil
}
```

然后，我们启动服务器：

```shell
~/codeDir/golangCode/scalars # go run server/server.go 
2019/08/30 02:04:46 connect to http://localhost:8080/ for GraphQL playground

```

然后，我们在浏览器里面做如下请求：

```typescript
# Write your query or mutation here
query {
  article {
    id
    text
  }
}
```

我们将会得到如下结果：

```typescript
{
  "data": {
    "article": {
      "id": "1",
      "text": "I am codinghuang"
    }
  }
}
```

现在，我们来给`article`增加一个查询参数，比如说时间，我们这里假定是`int64`这个标量。我们修改`schema`：

```typescript
scalar Int64

type Article {
  id: ID!
  text: String!
  time: Int64!
}

type Query {
  article (time: Int64): Article!
}
```

然后，我们需要去实现这个`Int64`标量。我们创建一个新的文件，叫做`int64.go`：

```shell
~/codeDir/golangCode/scalars # touch int64.go
```

内容如下：

```go
package scalars

import (
	"io"
)

// Int64 is int64
type Int64 int64

// UnmarshalGQL implements the graphql.UnMarshaler interface
func (i *Int64) UnmarshalGQL(v interface{}) error {
	return nil
}

// MarshalGQL implements the graphql.Marshaler interface
func (i Int64) MarshalGQL(w io.Writer) {
	return
}
```

我们需要去实现这里的`UnmarshalGQL`函数和`MarshalGQL`函数。这里，我们先简单的做个小测试，实现如下：

```go
package scalars

import (
	"io"
)

// Int64 is int64
type Int64 int64

// UnmarshalGQL implements the graphql.UnMarshaler interface
func (i *Int64) UnmarshalGQL(v interface{}) error {
	*i = 10
	return nil
}

// MarshalGQL implements the graphql.Marshaler interface
func (i Int64) MarshalGQL(w io.Writer) {
	w.Write([]byte("5"))
	return
}
```

然后，我们需要去修改`gqlgen.yml`文件，指明我们的这个自定义的标量：

```yaml
# .gqlgen.yml example
#
# Refer to https://gqlgen.com/config/
# for detailed .gqlgen.yml documentation.

schema:
- schema.graphql
exec:
  filename: generated.go
model:
  filename: models_gen.go
resolver:
  filename: resolver.go
  type: Resolver
models:
  Int64:
    model: scalars.Int64
```

然后，我们需要把`resolver.go`文件删除：

```shell
~/codeDir/golangCode/scalars # rm resolver.go 
```

然后重新生成：

```shell
~/codeDir/golangCode/scalars # gqlgen
```

我们会发现，此时`Article`这个`queryResolver`多了一个参数`time *Int64`，注意，这里的`Int64`是我们`scalars`包下自定义的那个`Int64`，而不是`golang`自带的那个`int64`。

`OK`，我们重新写一下这个`Article`函数：

```go
func (r *queryResolver) Article(ctx context.Context, time *Int64) (*Article, error) {
	fmt.Println(*time)
	return &Article{
		ID:   "1",
		Text: "I am codinghuang",
		Time: *time,
	}, nil
}
```

然后，我们重新启动服务器：

```shell
~/codeDir/golangCode/scalars # go run server/server.go 
2019/08/30 02:23:25 connect to http://localhost:8080/ for GraphQL playground

```

我们发起如下请求，我们传递了一个时间参数`"1"`：

```typescript
# Write your query or mutation here
query {
  article (time: "1") {
    id
    text
    time
  }
}
```

会得到如下的结果：

```typescript
{
  "data": {
    "article": {
      "id": "1",
      "text": "I am codinghuang",
      "time": 5
    }
  }
}
```

我们发现，返回的这个时间`5`实际上就是我们在`int64.go`文件里面实现的：

```go
func (i Int64) MarshalGQL(w io.Writer) {
	w.Write([]byte("5"))
	return
}
```

也就是说，我们`w io.Writer`里面写了什么内容，就会返回给前端。

我们再看看终端的输出：

```shell
~/codeDir/golangCode/scalars # go run server/server.go 
2019/08/30 02:28:40 connect to http://localhost:8080/ for GraphQL playground
10

```

打印出了`10`，而不是客户端传递给服务器的`1`。这个`10`其实就是我们再`int64.go`文件里面实现的：

```go
// UnmarshalGQL implements the graphql.UnMarshaler interface
func (i *Int64) UnmarshalGQL(v interface{}) error {
	*i = 10
	return nil
}
```

也就是说，我们在`*i`里面填写的值，可以被`Article queryResolver`的`time *Int64`获取到。

`OK`，那我们如何获取到前端传递过来的`time`呢？

我们对`int.go`里面的`UnmarshalGQL`函数做如下修改：

```go
// UnmarshalGQL implements the graphql.UnMarshaler interface
func (i *Int64) UnmarshalGQL(v interface{}) error {
	str, ok := v.(string)
	if !ok {
		return errors.New("time must be string")
	}
	n, err := strconv.ParseInt(str, 10, 64)

	*i = Int64(n)
	return err
}
```

也就是说，前端传递给服务器的参数，我们可以在`v interface{}`里面获取到。

然后重新启动服务器：

```shell
~/codeDir/golangCode/scalars # go run server/server.go 
2019/08/30 02:36:26 connect to http://localhost:8080/ for GraphQL playground

```

然后做如下请求：

```typescript
# Write your query or mutation here
query {
  article (time: 1) {
    id
    text
    time
  }
}
```

结果：

```typescript
{
  "errors": [
    {
      "message": "time must be string",
      "path": [
        "article"
      ]
    }
  ],
  "data": null
}
```

因为我们在`UnmarshalGQL`里面限定了`time`必须为`string`。

我们修改请求如下：

```typescript
# Write your query or mutation here
query {
  article (time: "100") {
    id
    text
    time
  }
}

```

结果：

```typescript
{
  "data": {
    "article": {
      "id": "1",
      "text": "I am codinghuang",
      "time": 5
    }
  }
}
```

我来看看终端的输出：

```shell
~/codeDir/golangCode/scalars # go run server/server.go 
2019/08/30 02:36:26 connect to http://localhost:8080/ for GraphQL playground
100

```

说明我们在`reoslver`里面获取到了客户端传递的这个时间字符串`100`，并且成功的转换为了我们自定义的`Int64`类型。

所以，对于自定义标量，总结下了实际上就是：

```
1、先定义好我们的两个解析函数
2、客户端传递一个自定义标量的参数，那么就会调用我们的UnmarshalGQL解析函数，把前端传递过来的值转化为我们自定义的标量类型（至于这里我们需不需要去限定客户端传递过来的是字符串还是整数，看个人情况吧，在我的例子里面，其实不可以不对传递的参数做字符串的要求，完全可以传递一个整数过来）
	然后，resolver就可以通过参数获取到我们在UnmarshalGQL解析函数里面设置的那个值
3、服务器返回给客户端的值，首先会经过MarshalGQL解析函数处理，然后再通过io.Writer写入我们需要返回给前端的值。
4、这一切，gqlgen都会帮我们自动的调用
```

最后，`GraphQL`好用。

