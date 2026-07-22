# Client 自动化接口

普通用户应优先使用 Printf 控制面或仓库提供的 AI Agent skill，不需要直接调用接口。

| 文档 | 状态 | 用途 |
| --- | --- | --- |
| [Panel API](panel-api-experimental.md) | Experimental | 使用 Client token 管理自己的 Mapping。 |
| [错误处理](errors.md) | 当前行为 | 识别认证、冲突、输入和服务错误。 |

Panel API 使用控制面 HTTPS 地址，并通过 `X-Printf-Token` header 认证。不要把 token 放入 URL、日志或提交记录。

官方 Client 的运行时通信不是稳定的第三方集成接口。除非 Printf 明确发布兼容性承诺，否则应直接使用对应版本的官方 Client，不要依赖其内部请求顺序、传输实现或未版本化字段。
