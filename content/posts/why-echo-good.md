+++
title = "为什么我觉得 labstack/echo 好用？"
date = "2023-09-05"
author = "Jeff"
cover = "https://image.0xbb.dev/2023/09/202309191939866.png"
description = "在新手刚使用这两个库的时候，在 `HandlerFunc` 这一层上我觉得 echo 做的会好一些，它可以帮助新手避免一些 return 问题。"
+++

本文主要是个人在使用体验上的一些对比和偏爱，讲讲为什么我觉得 echo 好用

## Echo and Gin

用 go 写 web 应该都知道 echo 和 gin 是相似度非常高的

- Echo https://github.com/labstack/echo
- Gin https://github.com/gin-gonic/gin

## Echo 哪里更好

对于我在这两个库的使用体验来说，我认为 echo 在 http handler 设计上做的比较好

对比一下它们的 `HandlerFunc`

```go
// Echo HandlerFunc
type HandlerFunc func(c Context) error

// Gin Handler func
type HandlerFunc func(*Context)
```

当我们以 echo 和 gin 完成一个 API 的时候，从例子上看看差异

```go
// Echo
e.POST("/users", func(c echo.Context) error {
    u := new(User)
    if err := c.Bind(u); err != nil {
        return err
    }

    return c.JSON(http.StatusCreated, u)
})

// Gin
e.POST("/users", func(c *gin.Context) {
    u := new(User)
    if err := c.ShouldBind(u); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusCreated, u)
})
```

差异点
- echo 要求 handelr 必须 return 一个 error，允许直接 return response handler
- gin 没有要求返回值，并且大部分 response handler 并不会直接中断请求流程

(**个人关点**) 对于一个新手来说，我两个库都并没有熟悉的情况下，在使用 gin 时很容易发生忘记 return 会出现 response 被多次处理，特别是在逻辑中有很多种 response 返回处理的情况。但 echo 可以很好的避免这个问题，因为它要求你必须 return 一个 error，除非你每次在使用 echo 的 response handler 时都不处理 func return 的 err 值。

## 总结

在新手刚使用这两个库的时候，在 `HandlerFunc` 这一层上我觉得 echo 做的会好一些，它可以帮助新手避免一些 return 问题。
