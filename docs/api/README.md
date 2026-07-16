# Client API

公开 Client API 分为两个层次：

| 文档 | 状态 | 受众 |
| --- | --- | --- |
| [Agent Protocol v0](agent-protocol-v0.md) | 官方 Client 使用 | Client 实现者、审计和故障排查 |
| [Panel API](panel-api-experimental.md) | Experimental | 用户自动化、Client Panel 集成 |
| [错误与兼容性](errors.md) | 当前行为 | 所有 API 使用者 |
| [OpenAPI](client-openapi.yaml) | Agent Protocol v0 | 工具和类型生成 |

Admin API、Node SSH 部署 API、证书 API 和 token 刷新 API 不属于公共 Client 协议。

## Base URL

示例：

```text
https://moon.example.com
```

Client API prefix：

```text
/api/client
```

## Token

当前协议有两种 token 传输形式：

| 接口 | 形式 |
| --- | --- |
| `POST /join` | JSON body 的 `token` |
| `POST /heartbeat` | JSON body 的 `token` |
| 其他 Client API | `X-Printf-Token` header |

不要把 token 放在 URL query string。

## 版本状态

当前 HTTP path 尚未包含 `/v1`，因此公开文档称为 Protocol v0。第三方 Client 应：

- 严格校验必需字段；
- 忽略响应中的未知新增字段；
- 不依赖 JSON 字段顺序；
- 对非 2xx 响应保留 status 和已脱敏 body；
- 在正式 v1 前准备处理不兼容升级。
