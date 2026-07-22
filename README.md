# Printf Client Releases

本仓库公开分发官方 Printf Client 二进制、Docker 镜像、安装文档和 AI Agent 发布工具。当前阶段不包含 Client 源码、服务端实现或生产环境配置。

## 使用 AI Agent 发布（推荐）

> 已取得 Printf Client token 且本机已有 Docker 时，推荐直接让能读取本地文件并执行命令的 AI Agent 完成 Compose 接入、Client 启动、Mapping 和公网 HTTPS 验证。Docker 镜像已包含运行所需组件，宿主机不需要额外安装网络辅助工具。

先把 token 写入 skill 目录中已被 Git 忽略的本地环境文件：

```bash
cp skills/printf-client-release/.env.example skills/printf-client-release/.env
```

不要把真实 token 放进提示词、Git 提交或 Issue。然后把下面这句话直接交给 Agent：

> 请读取当前仓库的 `AGENT-PUBLISH.md`，并严格按照它索引的 `$printf-client-release` skill，把我当前工作区中的 Docker Compose HTTP 服务发布为 Printf 公网 HTTPS 地址。不要输出 token，不要安装额外的宿主机网络工具，不要为 Printf 暴露应用宿主机端口。

完整入口见 [AGENT-PUBLISH.md](AGENT-PUBLISH.md)。可独立安装或提供给 Agent 的 skill 位于 [`skills/printf-client-release/`](skills/printf-client-release/SKILL.md)。

## Docker Compose 快速开始

先在 Printf 控制面创建 Client 并取得 token。在 Git Bash 或其他兼容 shell 中：

```bash
cp .env.example .env
chmod 0600 .env
```

编辑 `.env`：

```dotenv
PRINTF_TOKEN=replace-with-client-token
PRINTF_SERVER=https://moon.lprintf.com
PRINTF_BRIDGE_NETWORK=gateway
```

创建共享网络并启动 Client：

```bash
docker network inspect gateway >/dev/null 2>&1 || docker network create gateway
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail 100 printf-client
```

应用 Compose 参考 [compose.service.yml](compose.service.yml)：

- 只把认证网关、前端 Nginx 或单体应用等 HTTP 入口加入 `gateway`。
- 为入口声明唯一的小写 Docker alias。
- 数据库、Redis 和内部 API 留在应用默认网络。
- 使用容器内部端口，不为 Printf 添加宿主机 `ports`。

在控制面把 Target Service 设置为 alias 和容器内部端口，例如：

```text
http://my-app:8080
```

Windows Docker Desktop 使用验证过的 Alpine 变体：

```bash
docker compose -f docker-compose.alpine.yml pull
docker compose -f docker-compose.alpine.yml up -d
docker compose -f docker-compose.alpine.yml ps
```

Linux 和 macOS Docker 默认使用 `docker-compose.yml`。最后请求控制面返回的公网 HTTPS 地址，确认应用响应符合预期。

## Docker 镜像

Native GitHub Release 与 Docker 镜像使用独立版本线：

| 镜像 | 用途 |
| --- | --- |
| `lprintf/printf:v0.2` | 默认通用变体。 |
| `lprintf/printf:alpine-v0.2` | Windows Docker Desktop 验证变体。 |

单独拉取镜像：

```bash
docker pull lprintf/printf:v0.2
docker pull lprintf/printf:alpine-v0.2
```

两个镜像都是 Linux 容器镜像。`alpine` 不表示 Windows 原生二进制；无 Docker 时使用对应平台的 Native Release。

Compose 不挂载 Docker socket，token 和加密传输配置只持久化到本目录的 `.env` 与 `wg_data/`。Client 需要 `privileged` 和 `NET_ADMIN`，但不使用 host network；不要把它当作最小权限安全沙箱。

## Native 二进制

只有没有 Docker 或明确需要原生运行时，才使用 Native Client：

| 平台 | 架构 | 文件 |
| --- | --- | --- |
| Linux | x86_64 / amd64 | `printf-client-linux-amd64.tar.gz` |
| Linux | arm64 / aarch64 | `printf-client-linux-arm64.tar.gz` |
| macOS | Intel | `printf-client-macos-amd64.tar.gz` |
| macOS | Apple Silicon | `printf-client-macos-arm64.tar.gz` |
| Windows | x86_64 / amd64 | `printf-client-windows-amd64.zip` |

从 [Latest Release](https://github.com/lprintf/printf-client-releases/releases/latest) 下载压缩包及同名 `.sha256`。当前 Native 平台依赖和管理员权限要求见 [平台安装](docs/platforms.md)，完整步骤见 [开始使用](docs/getting-started.md)。

当前发布包尚未提供 macOS notarization、Windows Authenticode 或独立发布签名。SHA-256 只能验证文件完整性，不能替代发布者签名。

## 使用模型

| 模式 | 用途 | Target Service |
| --- | --- | --- |
| Docker Compose Bridge Client | 有 Docker 的 Windows、Linux 或 macOS 主机 | Docker alias 和容器内部端口。 |
| Native Client | 没有 Docker 或需要原生运行 | 默认本地服务端口。 |

Docker Compose 路径：

```text
公网 HTTPS -> Printf 加密传输通道 -> Client -> Docker alias:target_port
```

应用入口必须在容器内监听 `0.0.0.0:target_port`。不需要向宿主机或公网发布该端口。

## 文档

| 文档 | 用途 |
| --- | --- |
| [AI Agent 发布入口](AGENT-PUBLISH.md) | 可直接交给 Agent 的提示词和 skill 文件索引。 |
| [开始使用](docs/getting-started.md) | 从 token、Compose 或 Native 到首次运行。 |
| [配置参考](docs/configuration.md) | 用户需要设置的环境变量和数据目录。 |
| [平台安装](docs/platforms.md) | Linux、macOS、Windows Native 安装方法。 |
| [网络与 Mapping](docs/networking.md) | Compose alias、监听地址和必要网络要求。 |
| [故障排查](docs/troubleshooting.md) | Client、权限、Mapping 和服务不可达。 |
| [安全说明](SECURITY.md) | token、私钥和漏洞报告。 |

## Token 安全

Client token 可以运行 Client 并管理属于该 Client 的 Mapping，因此应把它当作密码处理：

- 一个 token 只运行一个 Client 实例。
- 不要同时运行使用同一 token 的 Docker Client 和 Native Client。
- 不要把 token 放进 URL、提示词、Issue、截图或 Git 历史。
- 环境文件权限应限制为仅管理员可读。
- token 泄露后立即在控制面刷新。

## 版本与发布来源

公开二进制由私有 `lprintf/printf-next` 仓库的 GitHub Actions 构建。每个 GitHub Release 会记录对应的源码 commit SHA 和构建 run。Docker 镜像使用独立版本标签，不要用 Native Release 版本推断 Docker 标签。

Printf Client 官方二进制按 [Binary Distribution License](LICENSE.md) 分发。第三方依赖仍受各自许可证约束，见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。

## 问题反馈

提交 Issue 前删除 token、认证 header、private key、完整传输配置、内网地址和不希望公开的生产信息。安全问题不要创建公开 Issue，请按 [SECURITY.md](SECURITY.md) 使用 GitHub 私密漏洞报告。
