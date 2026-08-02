# 使用 AI Agent 发布 Printf

请完整读取并严格执行官方 [`skills/printf-client-release/SKILL.md`](https://github.com/lprintf/printf-client-releases/blob/main/skills/printf-client-release/SKILL.md)，把用户当前工作区中的 HTTP 服务发布为 Printf 公网 HTTPS 地址。

## 获取 Skill

用户不需要 clone `printf-client-releases` 仓库。

- 本地已有完整 `skills/printf-client-release/` 时直接使用。
- 本地没有时，在目标项目之外选择一个不会被 Git 提交、重启后仍保留的私有目录。
- 从 `https://raw.githubusercontent.com/lprintf/printf-client-releases/main/skills/printf-client-release/` 下载下列最小文件集，并保持相对目录结构：
  `SKILL.md`、`.env.example`、`.gitignore`、`docker-compose.yml`、`docker-compose.alpine.yml`、`compose.service.yml`、`references/get-token.md`、`references/native.md` 和 `references/report-issue.md`。
- 每个文件都必须下载成功且非空。失败时报告原始错误并停止，不要凭记忆重建内容，也不要把远程内容通过管道直接交给 shell 执行。
- Client 的本地配置和传输数据会保存在该目录，因此发布完成后不要把它当作临时目录删除。

取得文件后，把该目录作为 Skill 工作目录，先完整读取 `SKILL.md`，再开始执行。

## 执行顺序

1. 先检查目标项目、Docker、Docker Compose、HTTPS 客户端、结构化 JSON 工具和 `PRINTF_TOKEN`，不要先修改项目或启动服务。
2. 如果进程环境和 Skill 目录的 `.env` 都没有有效 `PRINTF_TOKEN`，立即停止，告诉用户按照 [GET-TOKEN.md](GET-TOKEN.md) 获取内测 Token。不要猜测 Token，也不要要求用户在聊天中发送 Token。
3. Docker 正常可用时使用 Compose Bridge。Docker 命令不存在且应用能脱离 Docker 运行时，读取 `references/native.md` 并使用 Native Client；纯 Compose 应用没有 Docker 时停止。
4. 用户把 Token 写入本地环境或 Skill 目录的 `.env` 后，再继续 Client 启动、Mapping 和公网 HTTPS 验证。
5. 确认 Native 发布包自身有缺陷时，读取 `references/report-issue.md`；先去重、脱敏并展示草稿，取得用户确认后才提交公开 Issue。安全问题只使用私密漏洞报告。

## 发布边界

- 有 Docker 时使用 Compose Bridge，不在宿主机安装额外的 Client 网络辅助工具。
- Native 只发布宿主机已经运行或能原生启动的 HTTP 服务，不能替代 Docker 启动 Compose 应用。
- 从目标项目现有 Compose、Dockerfile、网关和健康检查中确认真正的 HTTP 入口及容器内部端口，不把示例 alias 或端口当作真实配置。
- 只把认证网关、前端 Nginx 或单体应用入口加入共享 external network；数据库、Redis 和内部 API 保持私有。
- Compose 使用 Docker alias 和容器内部端口；Native 使用默认本地目标和宿主机服务端口。
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
| [`skills/printf-client-release/references/native.md`](skills/printf-client-release/references/native.md) | 无 Docker 时的 Native 降级流程。 |
| [`skills/printf-client-release/references/report-issue.md`](skills/printf-client-release/references/report-issue.md) | Native 发布包缺陷的脱敏报告流程。 |
| [`GET-TOKEN.md`](GET-TOKEN.md) | 内测 Token 获取方式。 |
