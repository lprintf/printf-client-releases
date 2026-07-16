# 错误与兼容性

FastAPI validation error 通常返回：

```json
{
  "detail": [
    {
      "type": "...",
      "loc": ["body", "field"],
      "msg": "...",
      "input": "..."
    }
  ]
}
```

业务错误通常返回：

```json
{"detail": "Invalid token"}
```

## 常见状态码

| HTTP | 含义 |
| --- | --- |
| `200` | 成功。 |
| `400` | 请求语义错误、unsupported capability 或受限操作。 |
| `401` | join/heartbeat token 无效。 |
| `403` | header token 无效或资源不属于当前 Client。 |
| `404` | Mapping/Node 不存在或不可见。 |
| `409` | token 正在使用、capability 不满足或状态冲突。 |
| `422` | JSON/schema validation 失败。 |
| `500` | 控制面内部数据或部署错误。 |

不要根据错误消息文本做唯一业务分支；v0 阶段文本可能变化。优先判断 HTTP status，并把已脱敏的 response body用于诊断。

## 重试

建议重试：

- 网络超时；
- DNS 临时失败；
- `502`、`503`、`504`；
- heartbeat 连接错误。

不要自动无限重试：

- `400`；
- `401`；
- `403`；
- `409` token conflict；
- `422`。

join 会触发 Node peer 更新，不应按毫秒级高频重试。官方 Client 当前最多尝试三次，每次间隔约两秒；持续失败后退出。

## 日志脱敏

可以记录：

- endpoint path；
- HTTP status；
- request ID（如果服务端提供）；
- 已截断错误消息；
- config/route version。

禁止记录：

- token；
- private key；
- 完整 WireGuard config；
- 包含 token 的 request body/header。
