# Printf Client Releases

本仓库公开分发官方 Printf Client 二进制、安装文档和 Client 公共协议说明。

当前阶段不包含 Client 源码。控制面、Admin API、Node SSH 部署逻辑和生产环境配置也不在本仓库公开范围内。

## 下载

从 [Latest Release](https://github.com/lprintf/printf-client-releases/releases/latest) 下载对应平台的压缩包：

| 平台 | 架构 | 文件 |
| --- | --- | --- |
| Linux | x86_64 / amd64 | `printf-client-linux-amd64.tar.gz` |
| Linux | arm64 / aarch64 | `printf-client-linux-arm64.tar.gz` |
| macOS | Intel | `printf-client-macos-amd64.tar.gz` |
| macOS | Apple Silicon | `printf-client-macos-arm64.tar.gz` |
| Windows | x86_64 / amd64 | `printf-client-windows-amd64.zip` |

每个压缩包都附带 `.sha256` 文件。当前发布包尚未提供 macOS notarization、Windows Authenticode 或独立的发布签名；SHA-256 只能验证文件完整性，不能替代发布者签名。

## 最短运行路径

先在 Printf 控制面创建 Client 并取得 token。token 是高价值 bearer credential，不要提交到 Git、粘贴到 Issue 或发送给其他人。

Linux/macOS 安装对应的 WireGuard 工具后，以管理员权限运行：

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
| Native Client | Windows、Linux、macOS 独立主机上的本地服务 | 否 |
| Docker host-mode Client | Linux 宿主机上的本地服务 | 否 |
| Docker Bridge Client | 同机 Node 或跨 Compose service alias | 是 |

公开 Release 当前重点支持 Native Client。Native Client 的 direct Mapping 数据路径为：

```text
Internet -> Public Node -> WireGuard -> Client printf0 IP:target_port
```

目标服务必须监听 `0.0.0.0:target_port` 或 Client 的 WireGuard 地址。只监听 `127.0.0.1` 的服务无法接收来自 WireGuard 的流量。

Native Client 不解析 Docker service alias，也不能设置 `PRINTF_BRIDGE_MODE=true`。

## 文档

| 文档 | 用途 |
| --- | --- |
| [开始使用](docs/getting-started.md) | 从 token、下载到首次运行。 |
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

公开二进制由私有 `lprintf/printf-next` 仓库的 GitHub Actions 构建。每个 Release 会记录对应的源码 commit SHA 和构建 run。

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
