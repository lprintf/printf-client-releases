# 使用 AI Agent 发布 Printf

本文件是可以直接交给本地 AI Agent 的任务入口。Agent 必须能够读取本仓库、读取目标项目并执行 Docker 和 HTTPS API 命令。

## 给 Agent 的任务

请完整读取并严格执行 [`skills/printf-client-release/SKILL.md`](skills/printf-client-release/SKILL.md)，把用户当前工作区中的 Docker Compose HTTP 服务发布为 Printf 公网 HTTPS 地址。

执行时遵守以下边界：

- 优先使用 Docker Compose Bridge，不在宿主机安装 `wg`、`wg-quick` 或其他 Client 辅助工具。
- 从目标项目现有 Compose、Dockerfile、网关和健康检查中确认真正的 HTTP 入口及容器内部端口，不使用 `.env.example` 中的示例 alias 或端口冒充真实配置。
- 只把认证网关、前端 Nginx 或单体应用入口加入共享 external network；数据库、Redis 和内部 API 保持私有。
- 使用 Docker alias 作为 Client API 的 `target_host`，使用容器内部端口作为 `target_port`，不为 Printf 添加宿主机 `ports`。
- 默认使用通用 `lprintf/printf:v0.2`。只有用户明确使用 Windows Docker Desktop 时才选择 `lprintf/printf:alpine-v0.2`。
- 不回显、不提交、不复制真实 token 到命令输出。发现 token 已被其他在线 Client 使用时停止，不启动第二个实例。
- 完成 API Mapping 后重新读取配置，并验证 Mapping stats 和最终公网 HTTPS 地址；失败时报告原始错误，不静默降级。

## 用户准备

用户只需先取得 Printf Client token，并在本地创建 skill 环境文件：

```bash
cp skills/printf-client-release/.env.example skills/printf-client-release/.env
```

把真实 `PRINTF_TOKEN` 和控制面地址写入 `.env`。`PRINTF_TARGET_HOST` 和 `PRINTF_TARGET_PORT` 必须由 Agent 根据目标项目核实并更新。不要把 token 粘贴到本文件、Agent 提示词、Git 提交或 Issue。

## 文件索引

| 文件 | 用途 |
| --- | --- |
| [`skills/printf-client-release/SKILL.md`](skills/printf-client-release/SKILL.md) | 完整 Compose、Client API、幂等 Mapping 和验证工作流。 |
| [`skills/printf-client-release/docker-compose.yml`](skills/printf-client-release/docker-compose.yml) | 通用 `lprintf/printf:v0.2` Bridge Client。 |
| [`skills/printf-client-release/docker-compose.alpine.yml`](skills/printf-client-release/docker-compose.alpine.yml) | Windows Docker Desktop 验证过的 Alpine Bridge Client。 |
| [`skills/printf-client-release/compose.service.yml`](skills/printf-client-release/compose.service.yml) | 应用 HTTP 入口接入 external network 的最简结构参考。 |
| [`skills/printf-client-release/.env.example`](skills/printf-client-release/.env.example) | Client 凭据、网络、alias 和容器端口变量模板。 |
| [`docs/api/panel-api-experimental.md`](docs/api/panel-api-experimental.md) | Client token 可用的实验性 Panel API 说明。 |

根目录的 `docker-compose.yml`、`docker-compose.alpine.yml` 和 `compose.service.yml` 与 skill 内副本保持一致，便于用户直接查看和手工运行。
