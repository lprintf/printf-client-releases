# Printf Client

使用 Docker Compose 和 Printf Client，把现有 HTTP 服务发布为公网 HTTPS 地址。

## 直接交给 AI Agent（推荐）

把下面整段复制给能读取本地文件并执行命令的 AI Agent：

```text
请读取当前仓库的 `AGENT-PUBLISH.md`，使用其中的 `$printf-client-release` skill，把我当前工作区中的 Docker Compose HTTP 服务发布为 Printf 公网 HTTPS 地址。先检查运行环境和配置；缺少 `PRINTF_TOKEN` 时停止，并告诉我按照 `GET-TOKEN.md` 获取内测 Token，不要猜测 Token，也不要要求我在聊天中公开 Token。不要安装额外的宿主机网络工具，不要为 Printf 暴露应用宿主机端口。
```

无需提前准备 Token。Agent 检查后如果发现缺少 Token，会告诉你如何获取和配置；取得方式也可直接查看 [GET-TOKEN.md](GET-TOKEN.md)。

Agent 入口见 [AGENT-PUBLISH.md](AGENT-PUBLISH.md)，完整 Skill 位于 [`skills/printf-client-release/`](skills/printf-client-release/SKILL.md)。

## 手动使用 Docker Compose

先[获取内测 Token](GET-TOKEN.md)，再在 Git Bash 或其他兼容 shell 中执行：

```bash
cp .env.example .env
chmod 0600 .env
# 编辑 .env，填写 PRINTF_TOKEN

docker network inspect gateway >/dev/null 2>&1 || docker network create gateway
docker compose pull
docker compose up -d
docker compose ps
```

Windows Docker Desktop 使用 Alpine 变体：

```bash
docker compose -f docker-compose.alpine.yml pull
docker compose -f docker-compose.alpine.yml up -d
```

应用只需按 [compose.service.yml](compose.service.yml) 把 HTTP 入口加入 `gateway` 网络。Target Service 填写 Docker alias 和容器内部端口，例如 `http://my-app:8080`；不需要为 Printf 暴露宿主机端口。

## 更多文档

| 文档 | 用途 |
| --- | --- |
| [开始使用](docs/getting-started.md) | 完整的 Compose 和 Native Client 流程。 |
| [平台安装](docs/platforms.md) | 没有 Docker 时安装 Native Client。 |
| [配置参考](docs/configuration.md) | 环境变量和数据目录。 |
| [故障排查](docs/troubleshooting.md) | 定位 Client、Mapping 和服务访问问题。 |
| [安全说明](SECURITY.md) | Token、私钥和漏洞报告。 |

Docker 镜像：通用版本 `lprintf/printf:v0.2`；Windows Docker Desktop 验证版本 `lprintf/printf:alpine-v0.2`。

Printf Client 按 [Binary Distribution License](LICENSE.md) 分发，第三方依赖见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。
