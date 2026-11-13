# http-demo

用 tap 写的一个小 HTTP 框架，外加一个演示服务。目标是「能讲清楚 HTTP 请求怎么进来、
响应怎么出去」，不是替代生产级框架。

```sh
tap build
./build/http-demo            # Windows: build\http-demo.exe
PORT=9000 ./build/http-demo  # 换端口
```

启动后：

```sh
curl -i http://127.0.0.1:8080/
curl -i http://127.0.0.1:8080/hello/%E4%BD%A0%E5%A5%BD
curl -i http://127.0.0.1:8080/users/42
curl -i 'http://127.0.0.1:8080/search?q=http%20server'
curl -i -X POST --data 'ping' http://127.0.0.1:8080/echo
curl -i -X DELETE http://127.0.0.1:8080/users/42
```

## 分层

| 文件 | 职责 | 依赖 |
|---|---|---|
| `src/text.tp` | 字节级字符串工具：split / trim / find / URL 解码 / 大小写 | std.vec、std.string_builder |
| `src/http.tp` | 报文层：`Request` / `Response` / `Connection`，解析与序列化 | std.net（只用到 send/recv） |
| `src/router.tp` | 路由表：方法 + 模式匹配 + 参数提取 | http |
| `src/server.tp` | 监听、accept、连接循环、日志 | http、router、std.net |
| `src/main.tp` | 演示应用：只写「有哪些路由」和「每个路由做什么」 | 全部 |

依赖是单向的：`main → server → router → http → text`。`http` 不认识 router，
所以报文层可以脱离网络单测 —— `tests/` 就是这么做的。

## 怎么写一个路由

处理函数签名固定为 `*fn(&Request, &mut Response)`：

```tap
fn hello(request: &Request, response: &mut Response) {
    let name: String = request.param("name");
    response.reply_plain(200, "hello, " + name + "!\n");
}

fn main(): i32 {
    let app: Router = router.new();
    app.get("/hello/:name", &hello);
    return server.serve("127.0.0.1", 8080, app, false);
}
```

- `:name` 是参数段，匹配到的值进 `request.params`，用 `request.param("name")` 取
- `*` 作为方法名表示「任意方法」
- 路径能匹配但方法不行 → **405 + `Allow` 头**；路径也不匹配 → **404**

## 报文层的 API

```tap
request.header("content-type")     // 头，大小写不敏感
request.param("id")                // 路由参数
request.query_value("q")           // query 参数，值已 URL 解码
request.wants_keep_alive()         // 这条连接要不要复用

response.header("allow")           // 响应头
response.set_header("Location", "/")
response.reply_plain(200, "ok")    // 就地填 status / Content-Type / body
response.reply_html(...)   response.reply_json(...)
response.build(keep_alive)         // 序列化成线上字节

conn.read_request()                // 读一个完整请求 → ReadOutcome
conn.send(&response, keep_alive)   // 写回去，处理「一次发不完」
conn.close()

// 没有接收者的构造入口仍是模块函数（tap 还不支持 Response.new()）
http.new_response(200)  http.plain(404, "...")  http.html(...)
http.json_response(...)  http.redirect("/")  http.empty(204)
http.parse_head(head)   http.connect(socket)   http.status_text(404)
```

`Content-Length` 一律由 `response.build()` 按 body 长度算，调用方不用也不能自己写。
204 / 304 不写 `Content-Length` 也不发 body（RFC 7230 §3.3.2）。

## 测试

```sh
tapm test        # 或 TAPM=/path/to/tapm TAP=/path/to/tap tapm test
```

用例是普通 tap 文件，断言库在 `tests/lib.tp`：

| 用例 | 覆盖 |
|---|---|
| `tests/text_utils.tp` | 切分（保留空段）、trim、查找、URL 解码、大小写 |
| `tests/http_request.tp` | 请求行/头解析、query 取值、keep-alive 判定、响应序列化、坏报文 |
| `tests/routing.tp` | 参数段匹配、段数不符、404 / 405 分派与 `Allow` 头 |

> 用例文件名不能和 `src/` 里的模块同名（`tests/text.tp` 会让 `import text;` 解析到用例
> 自己，报 `entry file cannot import itself`），所以是 `text_utils.tp` / `routing.tp`。

## 已知限制

- **不支持 chunked、100-continue、gzip、multipart**。请求体只按 `Content-Length` 收，
  其余每一种都要额外的状态机，这个「简单框架」没有实现。
- **keep-alive 默认关闭**。`std.net` 没有超时也没有 poll，`receive` 是纯阻塞的；单线程下
  一条「活着但不发请求」的连接会把整个服务锁死（浏览器常这么干）。要复用连接就用
  `KEEP_ALIVE=1` 启动，代价是并发客户端会互相排队。
- **没有并发**。一次处理一个连接，慢请求会挡住后面所有人。要做并发得先给 `std.net`
  加非阻塞 + 事件循环，不是在这里堆线程。
- **存进结构体的函数指针不能直接调**。`route.handler(req, res)` 编不过，要先
  `let h: *fn(&Request, &mut Response) = route.handler; h(req, res)`。
  字段类型是函数指针时，调用点只认标识符，不认字段访问。
- **没有接收者的关联函数调不了**。`impl Response` 里写 `fn new(...)` 也白写 ——
  `Response.new()` 这条语法还不存在，构造器只能当模块函数（`http.new_response`）。
  所以 impl 里只放带 `self` 的方法。
