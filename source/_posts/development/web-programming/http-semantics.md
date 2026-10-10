---
title: 语义化 HTTP 笔记
date: 2026-10-11 01:39:17
tags:
  - http
categories:
  - Development
  - Web Programming
---

## 方法与语义

HTTP 方法决定请求语义（RFC 9110 §9.1）。先分清两个属性：

- safe 方法只读：GET、HEAD、OPTIONS、TRACE。
- idempotent 方法重复请求的效果与一次相同：PUT、DELETE 和所有 safe 方法。

safe 方法一定也是 idempotent 方法，反过来不成立。idempotent 只约束效果，响应可以不同（§9.2.2）。cacheable 与这两个属性无关：规范只为 GET、HEAD、POST 定义缓存语义，绝大多数缓存实现也只支持 GET 和 HEAD（§9.2.3）。

POST 和 PATCH 不是 idempotent，代理不得自动重试，客户端也不应自动重试。需要重试写操作时可以使用 Idempotency-Key 请求头字段，但它目前仍是 Internet-Draft（rev-07，2026 年 4 月过期），不是标准。

| 方法 | safe | idempotent | cacheable | 语义 |
| --- | --- | --- | --- | --- |
| GET | 是 | 是 | 是 | 读取目标资源的当前表示 |
| HEAD | 是 | 是 | 是 | 与 GET 相同，但不返回 body |
| POST | 否 | 否 | 有条件 | 由目标资源按自身语义处理请求 body |
| PUT | 否 | 是 | 否 | 创建资源，或用请求 body 整体替换资源状态 |
| DELETE | 否 | 是 | 否 | 删除资源与其当前功能的关联 |
| CONNECT | 否 | 否 | 否 | 建立隧道 |
| OPTIONS | 是 | 是 | 否 | 查询通信选项 |
| TRACE | 是 | 是 | 否 | 回环测试 |
| PATCH | 否 | 否 | 有条件 | 按补丁文档增量修改 |
| QUERY | 是 | 是 | 是 | 用请求 body 执行服务端查询 |

QUERY（RFC 10008，2026 年）是 safe 且 idempotent 的查询方法。请求必须带 Content-Type 请求头字段，缺失或不一致时服务器必须失败。

## 请求 body

- 有定义语义：POST、PUT、PATCH、QUERY。
- 无定义语义：GET、HEAD、DELETE、OPTIONS。
- 其中 GET、HEAD、DELETE 客户端不应生成 body。OPTIONS 可以带 body，但必须同时带 Content-Type 请求头字段。
- 禁止携带：TRACE。
- 没有 body：CONNECT。

带 body 的 GET、HEAD、DELETE 还可能被实现直接拒绝并关闭连接。各层行为并不一致：

- 浏览器 API：fetch 会抛 TypeError，XHR 丢弃 body，axios 不发送。
- 命令行与语言客户端：curl（`-X GET -d`）、Python requests 和 Go 允许发送，Java HttpClient 是否受限取决于实现。
- 服务端、代理、CDN：处理方式没有统一结论。

结论是不要依赖 GET 或 DELETE 的 body。

## PATCH 与补丁格式

PATCH 的 body 是补丁文档，格式由媒体类型决定。服务器必须原子应用整套变更，任何一条指令失败则全部不生效（RFC 5789 §2）。常用补丁格式（媒体类型）：

- JSON Patch：`application/json-patch+json`（RFC 6902）。body 是一个操作数组，每项用 `op` 说明动作（add、remove、replace、move、copy、test），用 `path` 指定位置，需要时再用 `value` 给出新值。
- JSON Merge Patch：`application/merge-patch+json`（RFC 7396，取代 RFC 7386）。直接给一份目标片段：要改的字段写新值，要删的字段写 null。
- XML：`application/xml-patch+xml`（RFC 7351，Informational）。

一个 JSON Patch 的例子：

```json
[
  {"op": "test", "path": "/version", "value": 3},
  {"op": "replace", "path": "/title", "value": "新标题"},
  {"op": "add", "path": "/tags/-", "value": "http"}
]
```

先确认版本号是 3，然后替换标题，并在 tags 数组末尾追加一个值。任何一步失败，整个 PATCH 都不会生效。

错误码：

- 400：补丁文档格式错误。
- 415：媒体类型不支持。
- 422：语义上无法处理。

支持 PATCH 的资源应当在 OPTIONS 响应里用 Accept-Patch 响应头字段声明支持的格式（§2.2、§3.1）。

## 状态码

| 方法 | 成功码 | 典型错误码 |
| --- | --- | --- |
| GET | 200、206、304 | 401、403、404、406、414 |
| HEAD | 200、304 | 同 GET |
| POST | 201（创建资源，应当带 Location 响应头字段）、200、204、303 | 400、409、412、413、415、422、428 |
| PUT | 201（创建）、200 或 204（修改） | 409、415、412、428 |
| DELETE | 202、204 或 200 | 404、409、412 |
| CONNECT | 任意 2xx | 非法端口通常 400 |
| OPTIONS | 200 | 401、403、405 |
| TRACE | 200 | 405 |
| PATCH | 200 或 204 | 400、404、409、412、415、422 |
| QUERY | 200 | Content-Type 请求头字段缺失或不一致时必须失败 |

这些状态码的 phrase 与含义：

| 状态码 | phrase | 含义 |
| --- | --- | --- |
| 200 | OK | 成功，响应带表示 |
| 201 | Created | 创建成功，通常带 Location 响应头字段 |
| 202 | Accepted | 请求已接受，尚未处理完 |
| 204 | No Content | 成功，没有 body |
| 206 | Partial Content | 只返回部分内容，通常配合 Range 请求头字段 |
| 301 | Moved Permanently | 资源永久迁移，POST 可能被改成 GET |
| 302 | Found | 资源临时迁移，POST 可能被改成 GET |
| 303 | See Other | 指向结果资源，客户端通常改用 GET 获取 |
| 304 | Not Modified | 条件请求命中，使用本地缓存 |
| 307 | Temporary Redirect | 临时重定向，保留方法和 body |
| 308 | Permanent Redirect | 永久重定向，保留方法和 body |
| 400 | Bad Request | 请求格式或参数有问题 |
| 401 | Unauthorized | 缺少有效认证 |
| 403 | Forbidden | 已认证但没有权限 |
| 404 | Not Found | 资源不存在 |
| 405 | Method Not Allowed | 方法不被目标资源支持 |
| 406 | Not Acceptable | 无法满足 Accept 请求头字段 |
| 409 | Conflict | 与资源当前状态冲突 |
| 412 | Precondition Failed | 前置条件不成立 |
| 413 | Content Too Large | 请求 body 过大 |
| 414 | URI Too Long | URI 过长 |
| 415 | Unsupported Media Type | 请求的媒体类型不支持 |
| 422 | Unprocessable Content | 语法正确，但语义无法处理 |
| 428 | Precondition Required | 要求请求带前置条件 |
| 429 | Too Many Requests | 触发限流 |
| 501 | Not Implemented | 服务器不认识这个方法 |
| 504 | Gateway Timeout | 网关等待上游响应超时 |
| 511 | Network Authentication Required | 需要先完成网络认证 |

- 405 响应必须携带 Allow 响应头字段，例如 `Allow: GET, POST`。
- 需要保留 POST 和 body 时用 307 或 308。
- 304 响应要带上与 200 一致的 ETag、Vary 等响应头字段。
- 415 响应可以用 Accept 或 Accept-Encoding 响应头字段提示可接受的格式。
- 限流用 429，可以带 Retry-After 响应头字段。429 和 511 都不能被缓存。
- 422 曾叫 Unprocessable Entity。判断状态码时只看数字，不要依赖原因短语（HTTP/1.1 状态行里的说明文字）。

## 条件请求与并发控制

两个人同时修改同一个资源时，后提交的人会覆盖前一个人的改动。解决办法是让请求带上“我改的是哪个版本”，服务器先对比版本，不一致就拒绝（RFC 9110 §13）。版本标识放在响应头字段 ETag 里，条件通过请求头字段传递。

- ETag（响应头字段）是服务器给某个版本的标识。它是一个字符串，客户端不需要理解内容，只要在下次请求时原样带回来。
- If-Match（请求头字段）用在 PUT、PATCH、DELETE 上：版本一致才执行，不一致返回 412。
- If-None-Match（请求头字段）用在 GET、HEAD 上：资源没变化时返回 304，客户端继续用本地副本。
- If-None-Match 用在写请求上时，条件不成立返回 412。值 `*` 表示“只有资源不存在才创建”。
- If-Match 要求 ETag 精确一致。
- If-None-Match 允许弱匹配，语义等价的表示也算匹配。
- 多个条件请求头字段同时出现时，服务器按固定顺序判断：If-Match、If-Unmodified-Since、If-None-Match、If-Modified-Since、If-Range。

更新一个资源时的例子（先 GET 获取资源，后 PUT 修改资源）：

```http
GET /articles/post1
→ 200 OK
→ ETag: "v1-e19f9d4daa28eedf"

PUT /articles/post1
If-Match: "v1-e19f9d4daa28eedf"
→ 版本一致：200 或 204
→ 期间被别人改过：412
```

如果服务器对表示做了转换（例如重新压缩），就不要在成功响应里返回 ETag（§9.3.4），否则客户端拿到的版本标识与实际内容对不上，后续条件请求会不断失败。

要强制客户端带条件写（RFC 6585 §3），可以用 428 走一个回环：

1. 服务器发现写请求没有条件请求头字段，返回 428，并说明要带 If-Match。
2. 客户端读取资源，拿到 ETag，带上 If-Match 重试。
3. 如果这期间资源被改过，服务器返回 412，客户端重新读取后再改。

## 缓存

缓存让重复的 GET 请求不必每次都回源（RFC 9111 §4）。一个响应能不能直接复用，要看几件事：

- 请求 URI 相同。
- 方法允许：GET、HEAD 可以，POST 有条件。
- Vary 响应头字段列出的请求头字段一致。例如响应声明 `Vary: Accept-Language`，语言不同的请求就不能共用缓存。
- 响应没有 no-cache 指令，或者已经回源验证过。
- 响应还在有效期内，或者允许使用旧副本，或者刚刚验证通过。

“验证”就是缓存带着 ETag 回源问一句“资源还是这个版本吗”。服务器返回 304 时，缓存继续用本地副本，并用响应头字段更新副本里的元数据。

请求带 Authorization 请求头字段时，它的响应默认不能被 CDN 这类共享缓存复用，除非明确写了 public、s-maxage 或 must-revalidate。

下面的指令都写在 Cache-Control 头字段里，请求和响应都可以带。例如 `Cache-Control: max-age=60, private` 表示：60 秒内可以直接用缓存，但只允许浏览器这类私有缓存保存。

| 指令 | 作用 |
| --- | --- |
| no-store | 完全不存储请求或响应 |
| no-cache | 可以存，但每次用之前都要回源确认 |
| max-age=N | N 秒内可以直接使用，不必回源 |
| s-maxage=N | 只对共享缓存生效，覆盖 max-age |
| public | 允许 CDN 等共享缓存保存 |
| private | 只允许浏览器等私有缓存保存 |
| must-revalidate | 过期后必须先回源确认，不能用旧副本 |
| no-transform | 中间件不得修改 body |
| only-if-cached | 只接受缓存里的响应，没有就返回 504 |

写操作会让缓存失效：PUT、POST、DELETE 成功后，目标 URI 的缓存必须作废（§4.4）。

POST、PATCH 的响应要缓存，条件更严格：必须有显式的过期信息，Content-Location 响应头字段与目标 URI 一致，并且只能用于后续的 GET、HEAD（RFC 9110 §9.3.3、RFC 5789 §2）。

QUERY 的响应可以缓存，但缓存键（缓存区分请求的依据）必须包含请求 body，否则不同查询会混进同一条缓存（RFC 10008 §2.7）。

## 错误响应

API 错误建议使用 RFC 9457 定义的 `application/problem+json`，对象成员如下：

- type：问题类型标识，缺省 about:blank。
- status：必须与 HTTP 状态码一致。
- title：描述问题类型，不应随实例变化。
- detail：供人阅读，不应被程序解析。
- instance：标识本次问题。
- 扩展成员：适合放字段级错误，但不要暴露堆栈或内部实现细节。

一个例子：

```json
{
  "type": "https://example.com/errors/out-of-credit",
  "status": 403,
  "title": "余额不足",
  "detail": "当前余额 30，需要 50"
}
```

## 认证与授权

客户端没带凭证，或者凭证无效时，服务器返回 401，并在 WWW-Authenticate 响应头字段里说明该用哪种方式认证。已经认证、但当前账号没有权限时返回 403，这种响应不需要带 WWW-Authenticate。

使用 OAuth 的 API，token 放在 Authorization 请求头字段里：`Authorization: Bearer <token>`。请求被拒绝时，可以在 WWW-Authenticate 里带上错误码：invalid_request 表示请求本身有问题（400），invalid_token 表示 token 过期或无效（401），insufficient_scope 表示权限不够（403）。收到 invalid_token 后，客户端可以刷新 token 再试一次（RFC 6750 §3.1）。

授权码流程要配合 PKCE，客户端能支持时必须使用 S256（RFC 7636）。OAuth 2.0 的安全基线见 RFC 9700（BCP 240）。

## REST 与成熟度模型

REST 是架构风格，没有标准文档。判断 API 风格成熟度时，常用一个四级模型：

- L0：所有请求都发到同一个地址，用 body 里的字段说明要做什么，HTTP 只是传输通道。
- L1：每个资源有自己的 URI，但操作方式还是千篇一律。
- L2：用 HTTP 方法和状态码表达操作，比如 GET 查询、POST 创建、PUT 更新、DELETE 删除。多数自称 REST 的 API 停在这一层。
- L3：响应里带链接，告诉客户端下一步能做什么、该请求哪个 URI。客户端不需要把流程写死，跟着链接走就行，这就是 HATEOAS。

Fowler 引用 Fielding 的观点：只有 L3 才算 REST。链接通常放在 Link 响应头字段里（RFC 8288）。

## 数据格式与 URI

- JSON 在网络传输中必须使用 UTF-8，且不加 BOM（RFC 8259 §8.1）。
- 时间戳使用 RFC 3339 的 date-time 格式。
- 自定义 JSON 媒体类型应当使用 `+json` 后缀（RFC 6838 §4.2.8）。
- 资源标识使用 URI（RFC 3986），URI 模板遵循 RFC 6570。

## 常见坑

- 在 GET 上执行变更（例如 `?do=delete`）：爬虫和预取会真的触发动作。只要 URI 里有动作参数，就必须在 safe 方法访问下禁用它。
- 所有错误都返回 200，再在 body 里放业务错误码：代理、缓存和监控无法识别失败，重试和告警都会失灵。
- 把敏感数据放进 GET 查询参数：数据会留在日志、浏览器历史和 Referer 请求头字段中，规范建议改用 POST 承载。

> 本笔记使用 Deepseek-Flash 进行辅助，于时间 2026-10-11 时完成。
> 我已进行详细严格的审查，修改其错误的地方，润色不讲人话的语句。
> 我可确保现有内容准确无误。
