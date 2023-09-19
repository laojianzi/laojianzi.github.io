+++
title = "【文档翻译】Echo - 快速开始"
date = "2023-09-17"
author = "Jeff"
cover = "https://image.0xbb.dev/2023/09/quick-start-echo.jpeg"
description = "一文教你如何快速开始使用 Echo"
+++

翻译自 Echo 文档 [Quick Start](https://echo.labstack.com/docs/quick-start)

## 安装
### 要求
安装 Echo 需要 [Go](https://go.dev/doc/install) 1.13 或更高版本，Go 1.12 支持有限，某些中间件无法使用。

确保您的项目文件夹位于 $GOPATH 以外。

```shell
$ mkdir myapp && cd myapp
$ go mod init myapp
$ go get github.com/labstack/echo/v4
```

如果您使用 Go v1.14 或更早版本，请使用:

```shell
$ GO111MODULE=on go get github.com/labstack/echo/v4
```

### Hello, World!
创建 `server.go`

```go
package main

import (
    "net/http"

    "github.com/labstack/echo/v4"
)

func main() {
    e := echo.New()
    e.GET("/", func(c echo.Context) error {
        return c.String(http.StatusOK, "Hello, World!")
    })
    e.Logger.Fatal(e.Start(":1323"))
}
```

启动服务器

```shell
$ go run server.go
```

浏览器访问 [http://localhost:1323](http://localhost:1323) 您应该会看到 Hello, World! 在页面上。

### 路由

```go
e.POST("/users", saveUser)
e.GET("/users/:id", getUser)
e.PUT("/users/:id", updateUser)
e.DELETE("/users/:id", deleteUser)
```

### Path 参数

```go
// e.GET("/users/:id", getUser)
func getUser(c echo.Context) error {
    // 从 path 参数 `users/:id` 中读取 user id
    id := c.Param("id")
    return c.String(http.StatusOK, id)
}
```

浏览器访问 [http://localhost:1323/users/joe](http://localhost:1323/users/joe) 您应该在页面上看到 'joe'。

### Query 参数

`/show?team=x-men&member=wolverine`

```go
//e.GET("/show", show)
func show(c echo.Context) error {
    // 从 query 参数中获取 team 和 member
    team := c.QueryParam("team")
    member := c.QueryParam("member")
    return c.String(http.StatusOK, "team:" + team + ", member:" + member)
}
```

浏览器访问 [http://localhost:1323/show?team=x-men&member=wolverine](http://localhost:1323/show?team=x-men&member=wolverine) 您应该在页面上看到 'team:x-men, member:wolverine'。

### Form 表单 application/x-www-form-urlencoded

`POST` `/save`

| name  | value            |
| ----- | ---------------- |
| name  | Joe Smith        |
| email | joe@labstack.com |

```go
// e.POST("/save", save)
func save(c echo.Context) error {
    // 从 form 表单中获取 name 和 email
    name := c.FormValue("name")
    email := c.FormValue("email")
    return c.String(http.StatusOK, "name:" + name + ", email:" + email)
}
```

运行以下命令:

```shell
$ curl -d "name=Joe Smith" -d "email=joe@labstack.com" http://localhost:1323/save
// => name:Joe Smith, email:joe@labstack.com
```

### Form 表单 multipart/form-data

`POST` `/save`

| name   | value     |
| ------ | --------- |
| name   | Joe Smith |
| avatar | avatar    |

```go
func save(c echo.Context) error {
    // 从 form 表单中获取 name 和 avatar
    name := c.FormValue("name")
    avatar, err := c.FormFile("avatar")
    if err != nil {
        return err
    }

    // 读取上传的文件
    src, err := avatar.Open()
    if err != nil {
        return err
    }
    defer src.Close()

    // 服务器创建接收文件
    dst, err := os.Create(avatar.Filename)
    if err != nil {
        return err
    }
    defer dst.Close()

    // 将文件内容写入服务器文件
    if _, err = io.Copy(dst, src); err != nil {
        return err
    }

    return c.HTML(http.StatusOK, "<b>Thank you! " + name + "</b>")
}
```

运行以下命令:

```shell
$ curl -F "name=Joe Smith" -F "avatar=@/你的路径/avatar.png" http://localhost:1323/save
// => <b>Thank you! Joe Smith</b>
```

运行以下命令可以检查上传的图像:

```shell
cd <项目文件夹>
ls avatar.png
// => avatar.png
```

### 处理请求

- 根据 Content-Type 请求头将 `json`, `xml`, `form` 或 `query` 的请求参数绑定到 Go 结构体中。
- 以 `json` 或 `xml` 格式显示响应，并返回状态码。

```go
type User struct {
    Name  string `json:"name" xml:"name" form:"name" query:"name"`
    Email string `json:"email" xml:"email" form:"email" query:"email"`
}

e.POST("/users", func(c echo.Context) error {
    u := new(User)
    if err := c.Bind(u); err != nil {
        return err
    }
    return c.JSON(http.StatusCreated, u)
    // 或者
    // return c.XML(http.StatusCreated, u)
})
```

### 静态资源

添加静态资源访问路径 `/static/*` 可以访问到 static 文件夹下的任何文件。

```go
e.Static("/static", "static")
```

[查看更多](https://echo.labstack.com/docs/static-files)

### 模版渲染

[查看更多](https://echo.labstack.com/docs/templates)

### Middleware 中间件

```go
// 全局有效
e.Use(middleware.Logger())
e.Use(middleware.Recover())

// Group 下有效
g := e.Group("/admin")
g.Use(middleware.BasicAuth(func(username, password string, c echo.Context) (bool, error) {
  if username == "joe" && password == "secret" {
    return true, nil
  }
  return false, nil
}))

// 只有使用的路由有效
track := func(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        println("request to /users")
        return next(c)
    }
}
e.GET("/users", func(c echo.Context) error {
    return c.String(http.StatusOK, "/users")
}, track)
```
