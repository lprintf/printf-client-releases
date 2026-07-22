# 开始使用

本文从控制面生成 token 开始，优先通过 Docker Compose 启动 Client；没有 Docker 时，再使用 Native Client。

## 1. 前提

你需要：

- 一个由 Printf 控制面创建的 Client。
- 该 Client 的 token。
- 控制面的 HTTPS 地址。
- 一台不与公网 Printf Node 同机的 Windows、Linux 或 macOS 主机。
- 管理员权限。
- 目标 HTTP 服务及其 Compose 入口服务和容器内部端口。

主机已有 Docker 时优先使用 Compose。镜像已包含 WireGuard 和网络工具，宿主机不需要额外安装。无 Docker 时使用 Native Client，并安装操作系统对应的 WireGuard 工具。

## 2. Docker Compose

在仓库根目录创建环境文件：

```bash
cp .env.example .env
chmod 0600 .env
```

编辑 `.env`：

```dotenv
PRINTF_TOKEN=replace-with-client-token
PRINTF_SERVER=https://moon.example.com
PRINTF_BRIDGE_NETWORK=gateway
```

创建共享 external network，启动通用 Client 并检查：

```bash
docker network inspect gateway >/dev/null 2>&1 || docker network create gateway
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail 100 printf-client
```

Windows Docker Desktop 使用验证过的 Alpine 变体：

```bash
docker compose -f docker-compose.alpine.yml pull
docker compose -f docker-compose.alpine.yml up -d
docker compose -f docker-compose.alpine.yml ps
```

Linux 和 macOS Docker 默认使用通用 `docker-compose.yml`。两个镜像都已包含 Client 所需工具，不要在宿主机重复安装 `wireguard-tools`。

参考仓库根目录的 `compose.service.yml` 修改应用现有 Compose：

- 只把前端 Nginx、认证网关或单体应用等公网 HTTP 入口加入 `gateway`。
- 为入口声明唯一的小写 DNS alias，例如 `my-app`。
- 使用容器内部监听端口，不为 Printf 添加宿主机 `ports`。
- 数据库、Redis 和内部 API 留在应用默认网络。

应用有认证网关时发布认证网关，不直接发布被保护的后端。启动应用后，在控制面把 Target Service 设置为 alias 和容器内部端口，例如：

```text
http://my-app:8080
```

`.env` 和 `wg_data/` 已被 Git 忽略。不要提交 token、私钥或完整 WireGuard 配置。同一个 token 不应同时运行 Compose 与 Native Client。

以下步骤只适用于没有 Docker 或需要原生运行的主机。

## 3. Native 保存 token

不要把真实 token 写进本文示例、Git 仓库或 Issue。

Linux/macOS 当前 shell：

```bash
export PRINTF_TOKEN='replace-with-client-token'
export PRINTF_SERVER='https://moon.example.com'
```

Windows PowerShell：

```powershell
$env:PRINTF_TOKEN = "replace-with-client-token"
$env:PRINTF_SERVER = "https://moon.example.com"
```

同一个 token 不应同时运行两个 Client。控制面会拒绝在线 Client 使用同一 token 注册不同 WireGuard 公钥。

## 4. Native 选择下载包

确认系统架构：

Linux：

```bash
uname -m
```

macOS：

```bash
uname -m
```

对应关系：

| 输出 | 下载包 |
| --- | --- |
| `x86_64` | `*-amd64.*` |
| `aarch64` | Linux `*-arm64.*` |
| `arm64` | macOS `*-arm64.*` |

Windows 当前提供 amd64 版本。

从 [Releases](https://github.com/lprintf/printf-client-releases/releases) 下载压缩包及同名 `.sha256`。

## 5. Native 验证哈希

Linux：

```bash
sha256sum --check printf-client-linux-amd64.tar.gz.sha256
```

macOS：

```bash
shasum -a 256 -c printf-client-macos-arm64.tar.gz.sha256
```

Windows PowerShell：

```powershell
$expected = (Get-Content .\printf-client-windows-amd64.zip.sha256).Split()[0]
$actual = (Get-FileHash .\printf-client-windows-amd64.zip -Algorithm SHA256).Hash.ToLowerInvariant()
if ($actual -ne $expected) { throw "SHA-256 mismatch" }
```

哈希不一致时不要运行文件，重新从官方 Release 下载。

## 6. Native 安装 WireGuard

Linux Debian/Ubuntu：

```bash
sudo apt-get update
sudo apt-get install --yes wireguard-tools
```

macOS：

```bash
brew install wireguard-tools
```

Windows：从 WireGuard 官方网站安装 WireGuard for Windows。Client 会调用安装目录中的 `wireguard.exe` 管理 tunnel service。

## 7. Native 前台运行

Linux amd64：

```bash
tar -xzf printf-client-linux-amd64.tar.gz
chmod 0755 printf-client
sudo env \
  PRINTF_RUNTIME_MODE=native \
  PRINTF_TOKEN="$PRINTF_TOKEN" \
  PRINTF_SERVER="$PRINTF_SERVER" \
  ./printf-client
```

macOS Apple Silicon：

```bash
tar -xzf printf-client-macos-arm64.tar.gz
chmod 0755 printf-client
sudo env \
  PRINTF_RUNTIME_MODE=native \
  PRINTF_WG_QUICK=/opt/homebrew/bin/wg-quick \
  PRINTF_TOKEN="$PRINTF_TOKEN" \
  PRINTF_SERVER="$PRINTF_SERVER" \
  ./printf-client
```

Windows 管理员 PowerShell：

```powershell
Expand-Archive .\printf-client-windows-amd64.zip -DestinationPath .\printf-client
Set-Location .\printf-client
$env:PRINTF_RUNTIME_MODE = "native"
.\printf-client.exe
```

## 8. 预期日志

首次启动会生成 WireGuard key：

```text
Generating new WireGuard keys...
Registering with Printf Control Panel
Success! Subscribed to 1 nodes.
Restarting WireGuard interface printf0 (native)...
WireGuard tunnel established.
Client is running
```

如果响应没有 active Node，Client 会明确退出，不会伪装在线。

## 9. Native 后台运行

Linux 可以安装仓库提供的 systemd unit：

```bash
sudo install -m 0755 printf-client /usr/local/bin/printf-client
sudo install -m 0644 packaging/printf-client.service /etc/systemd/system/printf-client.service
```

创建 `/etc/printf-client.env`：

```dotenv
PRINTF_TOKEN=replace-with-client-token
PRINTF_SERVER=https://moon.example.com
```

限制权限并启动：

```bash
sudo chmod 0600 /etc/printf-client.env
sudo systemctl daemon-reload
sudo systemctl enable --now printf-client
sudo systemctl status printf-client --no-pager
```

macOS 和 Windows 当前先以前台或管理员自行管理的服务方式运行。WireGuard tunnel service 不能替代 Printf Client 进程；Client 进程必须持续运行以发送 heartbeat 和接收配置更新。

## 10. 创建 Native direct Mapping

在控制面把 Target Service 设置为默认本地端口，例如：

```text
http://[默认本地]:8080
```

目标应用必须监听：

```text
0.0.0.0:8080
```

或 Client 的 WireGuard IP。只监听：

```text
127.0.0.1:8080
```

无法接收 Node 通过 WireGuard 发来的流量。

## 11. 验证

Docker Compose：

```bash
docker compose ps
docker compose logs --tail 100 printf-client
```

Linux Native：

```bash
sudo wg show
ip address show printf0
sudo journalctl -u printf-client -n 100 --no-pager
```

macOS：

```bash
sudo wg show
```

Windows：在 WireGuard UI 或管理员 PowerShell 中检查 `printf0` tunnel service。

最后请求 Mapping 的公网 HTTPS 地址。出现问题时按 [故障排查](troubleshooting.md) 从 Client、WireGuard、Node 到目标服务依次检查。
