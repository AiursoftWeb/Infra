# Observer 应用设计

## 功能概括

负责基于 AppCenter 颁发的 Token，对应用程序进行监控的服务。

## 数据库

Observer 的数据库中只下面几个表：

* Traces，用于存储 App 的日志。

### Traces

Traces 表有几个重要列：

* TraceId, 也就是此 Trace 的唯一标识符。
* LogLevel, 也就是此 Trace 的日志等级。
* EnvTime, 也就是此 Trace 的发生时间。
* Scope, 也就是此 Trace 的作用范围。
* IP, 也就是此 Trace 的来源 IP。
* Server, 也就是此 Trace 的来源服务器。
* Method, 也就是此 Trace 的来源方法。
* Path, 也就是此 Trace 的来源路径。
* Message, 也就是此 Trace 的日志内容。
* StackTrace, 也就是此 Trace 的堆栈信息。
* AppId, 也就是此 Trace 的所属 App 的 AppId。

## API 接口设计

Observer 应用只提供下面几个接口：

Send:

* App (普通权限 `traces:send`)

Query:

* App (普通权限 `traces:query`)
* AnyApp (限制权限 `traces:query-any`，注意操作的 AppId 是取自 Scope 的而不是 Token 本身的。)

上面两个 Query 的 API，需要增加一些特性进行筛选，例如：

* where scope is
* where log level between\higher\lower
* where time between\after\before
* where message contains
* where stack trace contains
* where IP is
* where server is
* where method is
* where app id is
* where path is
* where trace id is

为了将上述筛选器可以被序列化，其最终的 HTTP Body 应该是一个 JSON 对象，其结构如下：

(注意，这里AppId是从Scope中取出的，而不是从Filter中取出的。)

```json
{
    "scope": "Front-end",
    "logLevel": "Error",
    "time": {
        "after": "2020-01-01T00:00:00Z",
        "before": "2020-01-02T00:00:00Z"
    },
    "message": "Exception",
    "stackTrace": "Exception",
    "IP": "10.0.0.0",
    "server": "Server",
    "method": "GET",
    "path": "/",
    "traceId": "TraceId"
}
```

对于上述的筛选器，其最终的 SQL 语句应该是这样的：

```sql
SELECT * FROM Traces
WHERE Scope = 'Front-end'
AND LogLevel = 'Error'
AND EnvTime > '2020-01-01T00:00:00Z'
AND EnvTime < '2020-01-02T00:00:00Z'
AND Message LIKE '%Exception%'
AND StackTrace LIKE '%Exception%'
AND IP = '10.0.0.0‘
AND Server = 'Server'
AND Method = 'GET'
AND Path = '/'
AND TraceId = 'TraceId'
```

## 关键类

### ExceptionSender

SDK 方法，用于将异常发送到 Observer。

### ObserverMiddleware

SDK 方法，ASP.NET Core 中间件，用于将异常 HTTP 请求发送到 Observer。

## 实际场景

当一个应用需要将自己的日志发布到 Observer 时，需要做下面几个步骤：

* 安装 Observer SDK
* 在 AppCenter 中注册 App
* 设置 App 具有 `traces:send` 权限
* 将 AppCenter 中的 AppId 和 AppSecret 配置到应用程序中
* 配置使用 Observer SDK 发布日志

### 示例：前端 JavaScript 调用 API

首先应用需要向 AppCenter 申请一个 Token。注意这里需要增加一个Scope：Front-end，这是考虑到这个 Token 是给前端使用的，可能会被黑客盗用，所以需要限制其只能用于前端。

其上传的数据将全部只能展现在 Scope 为 Front-end 的 Trace 中。因此，不需要担心黑客会上传后端的日志。

在后端通过 API 申请到 Token 后，将其返回给前端。前端将 Token 保存在 LocalStorage 中，以便后续使用。

在页面崩溃而需要上传日志时，前端将 Token 从 LocalStorage 中取出，然后调用 Observer SDK 的 `Send` 方法，将日志发送到 Observer。

### 示例2：应用程序将 HTTP 请求日志发送到 Observer

后端通过 API 直接申请 Scope 为 `backend` 的 Token，然后将其保存在内存中。

在 ASP.NET Core 中间件中，将 Observer SDK 的 `ObserverMiddleware` 注册到管道中。这样，当一个 HTTP 请求发生时，将会自动将其日志发送到 Observer。
