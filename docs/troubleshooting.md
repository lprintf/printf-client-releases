# 故障排查

按以下顺序检查，不要一开始就删除私钥或重建 Client。

## 1. Client 是否启动

Linux：

```bash
sudo systemctl status printf-client --no-pager
sudo journalctl -u printf-client -n 100 --no-pager
```

前台运行时检查是否出现：

```text
WireGuard tunnel established.
Client is running
```

## 2. `PRINTF_TOKEN not set`

原因：进程没有收到环境变量。

systemd 环境文件应是：

```dotenv
PRINTF_TOKEN=replace-with-client-token
PRINTF_SERVER=https://moon.example.com
```

不要写：

```bash
export PRINTF_TOKEN=...
```

修改后：

```bash
sudo systemctl restart printf-client
```

## 3. `Invalid token`

可能原因：

- token 复制错误。
- token 已在控制面刷新。
- Client 指向了错误的控制面。
- 环境文件包含多余引号或不可见字符。

不要把 token 发到 Issue。可以在控制面重新生成并替换本机配置。

## 4. `Token already in use by an active client`

同一个 token 正被另一组 WireGuard 公钥使用。

检查：

- 是否同时运行 Docker 和 Native Client。
- 旧主机是否仍在线。
- systemd 是否启动了多个 unit。
- Windows 是否还有另一份 Client 进程。

停止旧实例，等待其离线或刷新 token 后再启动。

## 5. `Control plane returned no active nodes`

token 有效，但 Client 当前没有可用 Node。检查控制面：

- Client 是否绑定域名。
- 域名是否绑定 active Node。
- Node 是否标记 offline。
- Node WireGuard 是否已部署。

## 6. `wg-quick` 找不到

Linux：

```bash
command -v wg-quick
```

macOS Apple Silicon：

```bash
ls -l /opt/homebrew/bin/wg-quick
```

显式设置 `PRINTF_WG_QUICK`。

## 7. Windows WireGuard 安装失败

确认：

- 已安装 WireGuard for Windows。
- PowerShell 以管理员身份运行。
- `wireguard.exe` 路径正确。
- `printf0` tunnel service 没有被其他管理程序锁定。

必要时设置：

```powershell
$env:PRINTF_WIREGUARD_EXE = "C:\Program Files\WireGuard\wireguard.exe"
```

## 8. Client 在线但站点访问失败

按路径检查：

1. Client 最近是否 heartbeat。
2. Node 与 Client 是否有 WireGuard handshake。
3. Node 是否能访问 Client WireGuard IP。
4. Node 是否能连接 `client_wg_ip:target_port`。
5. 服务是否监听 `0.0.0.0:target_port`。
6. Node Nginx 是否已同步对应 Mapping。
7. 公网 DNS 和 HTTPS 是否指向正确 Node。

Linux 查看监听：

```bash
sudo ss -lntp
```

如果只看到：

```text
127.0.0.1:8080
```

应修改应用监听地址。

## 9. 修改 Mapping 后未生效

Client 每 30 秒 heartbeat。控制面通过 `config_version` 和 `route_version` 通知变更：

- WireGuard/Node 配置变化：Client 重新 join。
- Bridge route 变化：Bridge Client重新拉取 `/routes`。
- direct Mapping 的 Nginx upstream 主要由控制面同步 Node。

检查控制面 Node sync 是否成功，不要只重启 Client。

## 10. 收集脱敏信息

Issue 可包含：

- Client 版本。
- 操作系统和架构。
- runtime mode。
- 错误文本。
- 是否能访问控制面 HTTPS。
- 是否有 WireGuard handshake。

必须删除：

- token。
- private key。
- 完整 WireGuard config。
- 生产域名和不希望公开的 IP。
