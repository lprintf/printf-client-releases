# Printf Client

使用 Docker Compose 和 Printf Client，把现有 HTTP 服务发布为公网 HTTPS 地址。

## 直接交给 AI Agent（推荐）

无需 clone 本仓库。直接把下面整段复制给能访问 GitHub、读取本地项目并执行命令的 AI Agent：

```text
请读取 https://github.com/lprintf/printf-client-releases/blob/main/AGENT-PUBLISH.md，使用其中的 `$printf-client-release` skill，把我当前工作区中的 Docker Compose HTTP 服务发布为 Printf 公网 HTTPS 地址。如果本地没有该 Skill，按照文档从官方 Raw URL 获取最小文件集，不要 clone 整个 release 仓库。先检查运行环境和配置；缺少 `PRINTF_TOKEN` 时停止，并告诉我按照 https://github.com/lprintf/printf-client-releases/blob/main/GET-TOKEN.md 获取内测 Token，不要猜测 Token，也不要要求我在聊天中公开 Token。不要安装额外的宿主机网络工具，不要为 Printf 暴露应用宿主机端口。
```

无需提前准备 Token。Agent 会自行取得所需公开文件；如果发现缺少 Token，会告诉你如何获取和配置。取得方式也可直接查看 [GET-TOKEN.md](GET-TOKEN.md)。

Agent 入口见 [AGENT-PUBLISH.md](AGENT-PUBLISH.md)，完整 Skill 位于 [`skills/printf-client-release/`](skills/printf-client-release/SKILL.md)。

## 手动使用 Docker Compose

先[获取内测 Token](GET-TOKEN.md)。不 clone 仓库时，把 [`.env.example`](.env.example) 下载为 `.env`，再把 [docker-compose.yml](docker-compose.yml) 下载到同一持久化目录。然后在该目录使用 Git Bash 或其他兼容 shell 执行：

```bash
chmod 0600 .env
# 编辑 .env，填写 PRINTF_TOKEN

docker network inspect gateway >/dev/null 2>&1 || docker network create gateway
docker compose pull
docker compose up -d
docker compose ps
```

Windows Docker Desktop 还需下载 [docker-compose.alpine.yml](docker-compose.alpine.yml)，并使用 Alpine 变体：

```bash
docker compose -f docker-compose.alpine.yml pull
docker compose -f docker-compose.alpine.yml up -d
```

应用只需按 [compose.service.yml](compose.service.yml) 把 HTTP 入口加入 `gateway` 网络。Target Service 填写 Docker alias 和容器内部端口，例如 `http://my-app:8080`；不需要为 Printf 暴露宿主机端口。

## Native 二进制

没有 Docker 或明确需要原生运行时，可下载当前公开版本 [`client-v0.0.2`](https://github.com/lprintf/printf-client-releases/releases/latest)：

| 平台 | 下载包 |
| --- | --- |
| Linux amd64 | `printf-client-linux-amd64.tar.gz` |
| Linux arm64 | `printf-client-linux-arm64.tar.gz` |
| macOS Intel | `printf-client-macos-amd64.tar.gz` |
| macOS Apple Silicon | `printf-client-macos-arm64.tar.gz` |
| Windows amd64 | `printf-client-windows-amd64.zip` |

每个包都提供同名 `.sha256` 校验文件。Native Release 与 Docker 镜像使用独立版本线；当前 Native 版本是 `v0.0.2`，Docker 镜像版本是 `v0.2`。安装要求见 [平台安装](docs/platforms.md)。

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
