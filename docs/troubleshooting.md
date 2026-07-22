# 故障排查

按以下顺序检查，不要一开始就删除私钥、数据目录或重建 Client。

## 1. Client 是否启动

Docker Compose：

```bash
docker compose ps
docker compose logs --tail 100 printf-client
```

Native Linux：

```bash
sudo systemctl status printf-client --no-pager
sudo journalctl -u printf-client -n 100 --no-pager
```

日志出现 `Client is running` 表示 Client 已进入运行状态。容器反复重启时保留最早的明确错误，不要只看最后一行。

## 2. `PRINTF_TOKEN not set`

进程没有收到 token。Compose 用户检查当前目录的 `.env`；systemd 环境文件应为：

```dotenv
PRINTF_TOKEN=replace-with-client-token
PRINTF_SERVER=https://moon.example.com
```

systemd 环境文件不要写 `export`。修改后执行：

```bash
sudo systemctl restart printf-client
```

## 3. `Invalid token`

检查：

- token 是否复制完整。
- token 是否已在控制面刷新。
- `PRINTF_SERVER` 是否指向正确控制面。
- 环境文件是否包含多余引号或不可见字符。

不要把 token 发到 Issue 或聊天记录。无法确认时，在控制面刷新 token 并替换本地配置。

## 4. `Token already in use by an active client`

同一个 token 已有其他 Client 在线。检查是否同时运行了 Docker、Native、旧主机或重复容器。

停止旧实例并等待控制面显示离线后再启动。不要删除 `wg_data/` 或 Native 数据目录重试。

## 5. 没有可用服务节点

token 有效，但当前 Client 没有可用的公网服务路径。检查：

- Client 是否已经绑定域名。
- 域名服务状态是否正常。
- 控制面是否有明确的维护或配置错误。

本地重复重启通常不能解决服务端状态问题。保留错误文本并联系管理员。

## 6. Client 在线但站点无法访问

Docker Compose 按顺序检查：

1. Mapping 的 Target Service 是否使用正确 alias 和容器内部端口。
2. Client 和应用入口是否加入同一个 external network。
3. 应用入口是否在容器内监听 `0.0.0.0:target_port`。
4. 应用容器和健康检查是否正常。
5. Mapping 状态是否报告目标离线。
6. 公网 HTTPS 地址是否使用控制面返回的最终域名。

查看共享网络：

```bash
docker network inspect gateway
```

Native 模式还需确认目标服务没有只监听 `127.0.0.1`。Linux 查看监听：

```bash
sudo ss -lntp
```

## 7. 修改 Mapping 后未生效

1. 重新读取控制面的 Mapping，确认 alias、端口和启用状态已保存。
2. 等待最多 60 秒，再检查 Mapping 状态。
3. 查看 Client 日志是否有新的明确错误。
4. 直接请求最终公网 HTTPS 地址，不自行拼接域名。

仍未生效时保留 Mapping 状态和脱敏日志，不要通过反复开关或删除 Mapping 掩盖问题。

## 8. Native 平台依赖错误

当前 Native Linux/macOS 版本找不到 `wg-quick` 时，先按 [平台安装](platforms.md) 安装依赖，再确认路径：

```bash
command -v wg-quick
```

Windows 当前 Native 版本需要 WireGuard for Windows，并要求管理员权限。自定义安装路径时设置：

```powershell
$env:PRINTF_WIREGUARD_EXE = "D:\Tools\WireGuard\wireguard.exe"
```

Docker 用户不需要在宿主机安装这些 Native 依赖。

## 9. 收集脱敏信息

Issue 可以包含：

- Client 版本和使用的 Compose 文件。
- 操作系统和架构。
- Docker 或 Native 运行方式。
- 完整错误文本。
- Client、Mapping 和目标服务是否在线。

必须删除：

- token 和认证 header。
- private key 和完整传输配置。
- 生产域名、内网地址和不希望公开的日志内容。
