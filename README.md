# Printf Client Releases

本仓库公开分发官方 Printf Client 二进制、Docker 镜像版本、安装文档和 Client 公共协议说明。

当前阶段不包含 Client 源码。控制面、Admin API、Node SSH 部署逻辑和生产环境配置也不在本仓库公开范围内。

## Native 二进制下载

从 [Latest Release](https://github.com/lprintf/printf-client-releases/releases/latest) 下载对应平台的压缩包：

| 平台 | 架构 | 文件 |
| --- | --- | --- |
| Linux | x86_64 / amd64 | `printf-client-linux-amd64.tar.gz` |
| Linux | arm64 / aarch64 | `printf-client-linux-arm64.tar.gz` |
| macOS | Intel | `printf-client-macos-amd64.tar.gz` |
| macOS | Apple Silicon | `printf-client-macos-arm64.tar.gz` |
| Windows | x86_64 / amd64 | `printf-client-windows-amd64.zip` |

每个压缩包都附带 `.sha256` 文件。当前发布包尚未提供 macOS notarization、Windows Authenticode 或独立的发布签名；SHA-256 只能验证文件完整性，不能替代发布者签名。

## Docker 镜像

Native GitHub Release 与 Docker 镜像使用独立版本线。当前 Docker 镜像为：

| 镜像 | 运行时基础 | 用途 |
| --- | --- | --- |
| `lprintf/printf:v0.2` | Ubuntu 22.04 | 默认通用变体。 |
| `lprintf/printf:alpine-v0.2` | Alpine Linux | Windows Docker Desktop 验证变体。 |

主机已有 Docker 时，优先使用本仓库的 Compose。镜像已经包含 `wireguard-tools`、`iptables`、`iproute2` 和 CA 证书，宿主机不需要额外安装 `wg`、`wg-quick` 或其他 Client 辅助工具。

单独拉取镜像：

```bash
docker pull lprintf/printf:v0.2
docker pull lprintf/printf:alpine-v0.2
```

两个镜像都是 Linux 容器镜像。`alpine` 表示 Windows Docker Desktop 已验证的容器变体，不表示 Windows 原生二进制。Linux 和 macOS Docker 默认使用通用 `v0.2`；无 Docker 时再使用对应平台的 Native Release。

Compose 不挂载 Docker socket，token、WireGuard 私钥和配置只持久化到本目录的 `.env` 与 `wg_data/`。Client 需要 `privileged` 和 `NET_ADMIN`，但不使用 host network；不要把它当作最小权限安全沙箱。

## 最短运行路径

先在 Printf 控制面创建 Client 并取得 token。token 是高价值 bearer credential，不要提交到 Git、粘贴到 Issue 或发送给其他人。

已有 Docker 时，在 Git Bash 或其他兼容 shell 中：

```bash
cp .env.example .env
chmod 0600 .env
```

编辑 `.env` 写入真实 token 和控制面地址，然后启动：

```bash
docker network inspect gateway >/dev/null 2>&1 || docker network create gateway
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail 100 printf-client
```

默认 `docker-compose.yml` 使用通用 `lprintf/printf:v0.2`。Windows Docker Desktop 使用验证过的 Alpine 变体：

```bash
docker compose -f docker-compose.alpine.yml pull
docker compose -f docker-compose.alpine.yml up -d
docker compose -f docker-compose.alpine.yml ps
```

应用 Compose 参考 [compose.service.yml](compose.service.yml)：只把公网 HTTP 入口加入同一个 `gateway` external network，并声明唯一 alias。内部 API、数据库和 Redis 留在默认网络；不需要为 Printf 发布应用端口到宿主机。在控制面把 Target Service 设置为该 alias 和容器内部端口，例如 `http://my-app:8080`。

没有 Docker，或需要 Windows/macOS 原生运行时，再下载 Native Client。Linux/macOS 安装对应的 WireGuard 工具后，以管理员权限运行：

```bash
sudo env \
  PRINTF_RUNTIME_MODE=native \
  PRINTF_TOKEN="$PRINTF_TOKEN" \
  PRINTF_SERVER="https://moon.example.com" \
  ./printf-client
```

Windows 需要先安装 WireGuard for Windows，然后在管理员 PowerShell 中运行：

```powershell
$env:PRINTF_RUNTIME_MODE = "native"
$env:PRINTF_TOKEN = "replace-with-client-token"
$env:PRINTF_SERVER = "https://moon.example.com"
.\printf-client.exe
```

看到以下日志表示 Client 已完成注册和 WireGuard 配置：

```text
WireGuard tunnel established.
Client is running
```

完整步骤见 [开始使用](docs/getting-started.md)。

## 运行模式

| 模式 | 用途 | 支持 Docker alias |
| --- | --- | --- |
| Docker Compose Bridge Client | 有 Docker 的 Windows、Linux 或 macOS 主机 | 是 |
| Native Client | 没有 Docker，或需要 Windows、macOS 原生运行 | 否 |

本仓库根目录 Compose 提供 Bridge Client，数据路径为：

```text
Internet -> Public Node -> WireGuard -> Bridge Client relay -> Docker alias:target_port
```

应用入口必须在容器内监听 `0.0.0.0:target_port`。Client 与应用入口共享 external network，但应用不需要向宿主机或公网发布该端口。

公开 GitHub Release 继续提供 Native Client 二进制。Native Client 使用 direct Mapping，不解析 Docker service alias，也不能设置 `PRINTF_BRIDGE_MODE=true`。

## 文档

| 文档 | 用途 |
| --- | --- |
| [开始使用](docs/getting-started.md) | 从 token、Compose 或 Native 到首次运行。 |
| [配置参考](docs/configuration.md) | 环境变量、数据目录和接口名。 |
| [平台安装](docs/platforms.md) | Linux、macOS、Windows 安装方法。 |
| [网络与 Mapping](docs/networking.md) | direct Mapping、监听地址和同机限制。 |
| [故障排查](docs/troubleshooting.md) | join、WireGuard、权限和服务不可达。 |
| [Agent Protocol v0](docs/api/agent-protocol-v0.md) | 官方 Client 使用的 join、heartbeat 和 routes 协议。 |
| [Panel API（实验性）](docs/api/panel-api-experimental.md) | token 可访问的 Mapping 管理接口。 |
| [安全说明](SECURITY.md) | token、私钥和漏洞报告。 |

## Client 协议

官方 Client 当前依赖三个运行时接口：

```text
POST /api/client/join
POST /api/client/heartbeat
GET  /api/client/routes    # 仅 Bridge Client
```

普通用户不需要手动调用这些接口。公开协议主要用于：

- 理解 Client 的网络行为。
- 故障定位。
- 开发兼容的第三方 Client。
- 审计 token、WireGuard 和 route 同步边界。

协议当前标记为 `v0`。HTTP 路径尚未包含 `/v1`，在正式 v1 之前可能发生不兼容变更。

## Token 安全

当前 Client token 同时允许：

- Client join。
- heartbeat。
- Bridge route 同步。
- 读取 Client 配置和 Mapping。
- 创建、更新、启停和删除属于该 Client 的 Mapping。

因此应把 token 当作密码处理：

- 一个 token 只运行一个 Client 实例。
- 不要同时运行 Docker Client 和 Native Client。
- 不要把 token 放进命令行 URL。
- 环境文件权限应限制为仅管理员可读。
- token 泄露后立即在控制面刷新。

## 版本与发布来源

公开二进制由私有 `lprintf/printf-next` 仓库的 GitHub Actions 构建。每个 GitHub Release 会记录对应的源码 commit SHA 和构建 run。Docker 镜像使用上文列出的独立版本标签，不要用 Native Release 版本推断 Docker 标签。

公开仓库不接受二进制形式的第三方“重新打包版本”。请只从本仓库 GitHub Releases 下载官方文件。

## 许可证

Printf Client 官方二进制按 [Binary Distribution License](LICENSE.md) 分发。第三方 Rust 依赖仍受各自许可证约束，见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。

## 问题反馈

提交 Issue 前请删除：

- `PRINTF_TOKEN`。
- `private.key`。
- 完整 WireGuard 配置。
- `X-Printf-Token` header。
- 内网地址、生产域名或其他敏感部署信息。

安全问题不要提交公开 Issue，请按 [SECURITY.md](SECURITY.md) 使用 GitHub 私密漏洞报告。
