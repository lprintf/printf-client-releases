# 多服务唯一出口

当前端、后端、数据库、缓存或认证服务同时存在时，使用本流程。先完整读取 [compose.gateway.yml](../compose.gateway.yml)，再修改用户现有 Compose；示例只表达网络和服务边界，不覆盖项目已有构建、环境变量、卷或健康检查。

## 架构规则

| 服务 | 私有 `default` 网络 | Printf `gateway` 网络 | 公网角色 |
| --- | --- | --- | --- |
| 唯一出口，例如 Nginx 或认证网关 | 是 | 是，声明唯一 alias | 提供前端并代理 API、认证等公网路径 |
| 后端 API | 是 | 否 | 只接受唯一出口的内部请求 |
| 数据库、Redis、队列 | 是 | 否 | 仅供内部服务访问 |
| 构建任务、迁移任务 | 按项目需要 | 否 | 不接受公网请求 |

禁止因为后端已有健康检查或监听端口就直接发布后端。用户要求部署完整网站时，公网入口必须同时覆盖前端页面以及页面依赖的 API、认证和静态资源。

## 选择唯一出口

按以下优先级选择且只选择一个入口：

1. 项目已有认证网关时使用认证网关。
2. 前端由 Nginx 或其他反向代理提供时使用该前端服务，并让它代理 API 路径。
3. 项目只有独立前端运行时和后端时，增加或调整一个反向代理作为统一入口。

如果现有架构无法通过一个入口完整提供网站，停止并说明缺少的路由，不要把前端和后端都加入 `gateway` 网络作为补救。

## Compose 约束

- 显式设置 DNS 安全且唯一的 `COMPOSE_PROJECT_NAME`，只允许小写 `a-z0-9-`，不能以 `-` 开头或结尾。
- 唯一出口同时加入 `default` 和 external `gateway`，alias 使用 `${COMPOSE_PROJECT_NAME:-app}`。
- `default` 使用 `${COMPOSE_PROJECT_NAME:-app}-private`，承载出口、后端和数据服务之间的内部通信。
- 只有唯一出口加入 external `gateway`。后端、数据库、Redis 和任务服务只加入 `default`。
- 唯一出口通过 Compose service name 访问内部服务，例如 `api:8000`、`database:5432`，不使用宿主机地址。
- Mapping 的 `target_host` 必须等于解析后的 `COMPOSE_PROJECT_NAME`，`target_port` 必须是唯一出口的容器内部监听端口。
- 不为 Printf 新增宿主机 `ports`。已有 `ports` 若承担独立本机用途，先确认再决定是否保留。
- 保留项目原生健康检查。只有服务确实声明 healthcheck 时，才使用 `condition: service_healthy`。

## 路由完整性

读取前端请求、反向代理配置和后端路由，确认至少覆盖：

- `/`、前端静态资源和客户端路由回退。
- 页面实际调用的 API 前缀，例如 `/api/`。
- 登录、回调、上传、WebSocket 或 SSE 路径（如果项目使用）。
- 唯一出口自身的健康检查路径。

优先使用同源路径，避免为了发布而引入额外公网后端域名和 CORS 配置。不要绕过现有认证层。

## 验收

启动 Mapping 前必须确认：

1. `docker compose config --quiet` 成功。
2. 只有一个服务连接 external `gateway`，并且该服务同时连接私有 `default`。
3. 后端和数据服务没有连接 external `gateway`，也没有新增公网端口。
4. 唯一出口能通过私有 service name 访问后端，后端能访问数据库或缓存。
5. 从唯一出口请求前端、API 和认证关键路径均符合预期。
6. `PRINTF_TARGET_HOST` 与唯一 alias 完全一致，`PRINTF_TARGET_PORT` 与出口容器内部端口一致。

任一条件失败时报告实际配置或请求错误并停止，不创建公网 Mapping。
