# 配置参考

Printf Client 通过环境变量配置。未知或非法值会显式报错，不会静默切换运行模式。

## 必要变量

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `PRINTF_TOKEN` | 无 | 控制面生成的 Client token。必须设置。 |
| `PRINTF_SERVER` | `https://moon.lprintf.com` | 控制面 HTTPS 地址。建议生产环境显式设置。 |
| `PRINTF_RUNTIME_MODE` | `docker` | `docker` 或 `native`。下载的宿主机二进制必须设置为 `native`。 |

Client 会删除 `PRINTF_SERVER` 末尾多余的 `/`。

## Compose 变量

| 变量 | 示例 | 说明 |
| --- | --- | --- |
| `PRINTF_BRIDGE_NETWORK` | `gateway` | Client 与应用 HTTP 入口共享的预先创建 external network。 |

容器默认使用 `PRINTF_RUNTIME_MODE=docker`，根目录 Compose 显式设置 `PRINTF_BRIDGE_MODE=true`，不需要在 `.env` 重复设置。应用 Mapping 使用入口的 Docker alias 和容器内部端口，不使用宿主机端口。

## WireGuard 变量

| 变量 | Native 默认值 | 说明 |
| --- | --- | --- |
| `PRINTF_DATA_DIR` | 按平台选择 | 私钥、公钥和 WireGuard config 目录。 |
| `PRINTF_INTERFACE_NAME` | `printf0` | WireGuard 配置/接口名，1–15 个 ASCII 字符。 |
| `PRINTF_WG_QUICK` | `wg-quick` | Linux/macOS 的可执行文件。 |
| `PRINTF_WIREGUARD_EXE` | `%ProgramFiles%\WireGuard\wireguard.exe` | Windows WireGuard executable。 |

接口名允许：

```text
a-z A-Z 0-9 . - _
```

## 默认数据目录

| 平台 | Native 数据目录 |
| --- | --- |
| Linux | `/var/lib/printf-client` |
| macOS | `/Library/Application Support/Printf Client` |
| Windows | `%ProgramData%\Printf Client` |

目录内容：

```text
private.key
public.key
printf0.conf
```

缺少私钥但存在公钥，或公私钥不匹配时，Client 会停止并报错，不会覆盖现有密钥。

## Bridge relay 变量

以下变量只适用于 Docker Bridge Client：

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `PRINTF_BRIDGE_MODE` | `false` | 是否注册 `bridge_tcp_relay_v1`。根目录 Compose 显式启用；Native 模式禁止启用。 |
| `PRINTF_DNS_CACHE_TTL_SECONDS` | `10` | Docker alias DNS 缓存时间。 |
| `PRINTF_RELAY_CONNECT_TIMEOUT_SECONDS` | `5` | relay 连接目标超时。 |
| `PRINTF_RELAY_IDLE_TIMEOUT_SECONDS` | `300` | relay 空闲连接超时。 |
| `PRINTF_RELAY_MAX_CONNECTIONS` | `1024` | 全局并发连接上限。 |
| `PRINTF_RELAY_MAX_CONNECTIONS_PER_ROUTE` | `256` | 每条 route 的并发上限。 |

所有时间和连接限制必须大于零。

## Linux systemd 环境文件

推荐 `/etc/printf-client.env`：

```dotenv
PRINTF_TOKEN=replace-with-client-token
PRINTF_SERVER=https://moon.example.com
```

systemd unit 已设置：

```dotenv
PRINTF_RUNTIME_MODE=native
```

权限：

```bash
sudo chown root:root /etc/printf-client.env
sudo chmod 0600 /etc/printf-client.env
```

不要在环境文件中添加 shell `export`，systemd `EnvironmentFile` 不是 shell 脚本。

## 多实例

不建议同机运行多个使用同一 WireGuard 子网的 Client。确需多个实例时，至少需要不同的：

- token；
- `PRINTF_DATA_DIR`；
- `PRINTF_INTERFACE_NAME`；
- systemd unit；
- WireGuard assigned IP 和路由。

同一个 token 不允许两个不同 WireGuard 公钥的在线实例同时 join。

## 切换 Docker 与 Native

切换前：

1. 停止旧 Client。
2. 确认旧 heartbeat 已离线。
3. 决定是否迁移旧 WireGuard key。
4. 启动新 Client。
5. 检查 Node handshake 和 Mapping。

不要让两种模式并行争用同一个 token 和 assigned IP。
