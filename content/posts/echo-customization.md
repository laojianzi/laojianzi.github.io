+++
title = "【文档翻译】Echo - 定制化"
date = "2023-09-19"
author = "Jeff"
cover = "https://image.0xbb.dev/2023/09/customization_640.png"
description = "一文介绍 Echo 中的定制化设置"
+++

翻译自 Echo 文档 [Customization](https://echo.labstack.com/docs/customization)

## 定制化

## Debug 模式

`Echo#Debug` 可用于启用/禁用调试模式，需要启用调试模式可以将日志级别设置为 `DEBUG`。

## Logging 日志

日志的默认格式是 JSON，可通过修改 header 进行变更。

### Log Header

`Echo#Logger.SetHeader(string)` 可以为 logger 设置 header，默认值:

```js
{"time":"${time_rfc3339_nano}","level":"${level}","prefix":"${prefix}","file":"${short_file}","line":"${line}"}
```

*示例*

```go
import "github.com/labstack/gommon/log"

/* ... */

if l, ok := e.Logger.(*log.Logger); ok {
  l.SetHeader("${time_rfc3339} ${level}")
}
```

```sh
2018-05-08T20:30:06-07:00 INFO info
```

#### 可用的 Tags

- `time_rfc3339`
- `time_rfc3339_nano`
- `level`
- `prefix`
- `long_file`
- `short_file`
- `line`

### Log Output

`Echo#Logger.SetOutput(io.Writer)` 可以为 logger 设置输出地址， 默认值是 `os.Stdout`。

如果想要完全禁用输出，可以使用 `Echo#Logger.SetOutput(io.Discard)` 或者 `Echo#Logger.SetLevel(log.OFF)`

### Log Level

`Echo#Logger.SetLevel(log.Lvl)` 可以为 logger 设置日志等级，默认值是 `ERROR`， 你可以使用以下几种等级的值:

- `DEBUG`
- `INFO`
- `WARN`
- `ERROR`
- `OFF`

### 自定义 Logger

Echo 提供了接口定义 `echo.Logger`，所以用户只需要按照 `Echo#Logger` 注册自定义的日志 logger。

## 启动 Banner

`Echo#HideBanner` 可以用于隐藏启动 banner。

## 自定义 Listener

`Echo#*Listener` 可以用于运行我们自定义的 listener 监听器。

*示例*

```go
l, err := net.Listen("tcp", ":1323")
if err != nil {
  e.Logger.Fatal(err)
}
e.Listener = l
e.Logger.Fatal(e.Start(""))
```

## 禁用 HTTP/2

`Echo#DisableHTTP2` 可以用于禁用 HTTP/2 协议。

## Read Timeout

`Echo#*Server#ReadTimeout` 可以用于设置请求读取的最大超时时间。

## Write Timeout

`Echo#*Server#WriteTimeout` 可以用于设置响应写入的最大超时时间。

## 参数校验器 Validator

`Echo#Validator` 可以用于注册校验器 validator，validator 可以为所有的 payload 做数据验证。

[查看更多](https://echo.labstack.com/docs/request#validate-data)

## 自定义 Binder

`Echo#Binder` 可以用于注册自定义的 binder，binder 可以将所有的 payload 数据绑定到变量中。

[查看更多](https://echo.labstack.com/docs/request#custom-binder)

## 自定义 JSON 序列化器

`Echo#JSONSerializer` 可以用于注册自定义的 JSON 序列化器。

请参阅 [json.go](https://github.com/labstack/echo/blob/master/json.go) 上的 `DefaultJSONSerializer`。

## 渲染器

`Echo#Renderer` 可以为模版渲染注册一个渲染器。

[查看更多](https://echo.labstack.com/docs/templates)

## HTTP 异常处理

`Echo#HTTPErrorHandler` 可以用于注册一段 http 错误时执行的处理逻辑。

[查看更多](https://echo.labstack.com/docs/error-handling)
