# 使用 AI Agent 发布 Printf

请完整读取并严格执行 [`skills/printf-client-release/SKILL.md`](skills/printf-client-release/SKILL.md)，把用户当前工作区中的 Docker Compose HTTP 服务发布为 Printf 公网 HTTPS 地址。

## 执行顺序

1. 先检查 Docker、Docker Compose、`curl`、`jq`、目标项目和 `PRINTF_TOKEN`，不要先修改项目或启动服务。
2. 如果进程环境和 Skill 目录的 `.env` 都没有有效 `PRINTF_TOKEN`，立即停止，告诉用户按照 [GET-TOKEN.md](GET-TOKEN.md) 获取内测 Token。不要猜测 Token，也不要要求用户在聊天中发送 Token。
3. 用户把 Token 写入本地环境或 Skill 目录的 `.env` 后，再继续 Compose 接入、Client 启动、Mapping 和公网 HTTPS 验证。

## 发布边界

- 有 Docker 时使用 Compose Bridge，不在宿主机安装额外的网络辅助工具。
- 从目标项目现有 Compose、Dockerfile、网关和健康检查中确认真正的 HTTP 入口及容器内部端口，不把示例 alias 或端口当作真实配置。
- 只把认证网关、前端 Nginx 或单体应用入口加入共享 external network；数据库、Redis 和内部 API 保持私有。
- 使用 Docker alias 和容器内部端口创建 Mapping，不为 Printf 添加宿主机 `ports`。
- 默认使用 `lprintf/printf:v0.2`。只有用户明确使用 Windows Docker Desktop 时才选择 `lprintf/printf:alpine-v0.2`。
- 不回显、不提交 Token。发现 Token 已被其他在线 Client 使用时停止，不启动第二个实例。
- 完成 Mapping 后重新读取配置，并验证 Mapping stats 和最终公网 HTTPS 地址；失败时报告原始错误，不静默降级。

## 文件索引

| 文件 | 用途 |
| --- | --- |
| [`skills/printf-client-release/SKILL.md`](skills/printf-client-release/SKILL.md) | 完整发布工作流。 |
| [`skills/printf-client-release/docker-compose.yml`](skills/printf-client-release/docker-compose.yml) | 通用 Compose Bridge Client。 |
| [`skills/printf-client-release/docker-compose.alpine.yml`](skills/printf-client-release/docker-compose.alpine.yml) | Windows Docker Desktop 验证版本。 |
| [`skills/printf-client-release/compose.service.yml`](skills/printf-client-release/compose.service.yml) | 应用 HTTP 入口接入参考。 |
| [`skills/printf-client-release/.env.example`](skills/printf-client-release/.env.example) | 本地配置模板。 |
| [`GET-TOKEN.md`](GET-TOKEN.md) | 内测 Token 获取方式。 |
