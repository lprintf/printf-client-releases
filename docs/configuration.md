# 配置参考

Printf Client 通过环境变量配置。未知或非法值会显式报错，不会静默切换运行模式。

## 必要变量

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `PRINTF_TOKEN` | 无 | 控制面生成的 Client token，必须设置。 |
| `PRINTF_SERVER` | `https://moon.lprintf.com` | 控制面 HTTPS 地址。 |
| `PRINTF_RUNTIME_MODE` | `docker` | Docker 使用默认值；下载的宿主机二进制必须设置为 `native`。 |

Client 会删除 `PRINTF_SERVER` 末尾多余的 `/`。

## Docker Compose

根目录 `.env`：

```dotenv
PRINTF_TOKEN=replace-with-client-token
PRINTF_SERVER=https://moon.example.com
PRINTF_BRIDGE_NETWORK=gateway
```

| 变量 | 默认示例 | 说明 |
| --- | --- | --- |
| `PRINTF_BRIDGE_NETWORK` | `gateway` | Client 与应用 HTTP 入口共享的 external network。 |

应用 Mapping 使用入口的 Docker alias 和容器内部端口，不使用宿主机端口。一般不需要调整其他网络参数。

## Native 平台选项

当前 Native 版本在需要时支持以下覆盖：

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `PRINTF_DATA_DIR` | 按平台选择 | 本地凭据和传输配置目录。 |
| `PRINTF_INTERFACE_NAME` | `printf0` | 当前版本的本地接口名。 |
| `PRINTF_WG_QUICK` | `wg-quick` | Linux/macOS 当前版本使用的命令路径。 |
| `PRINTF_WIREGUARD_EXE` | `%ProgramFiles%\WireGuard\wireguard.exe` | Windows 当前版本使用的程序路径。 |

只有默认路径不可用时才设置后两项。具体安装命令见 [平台安装](platforms.md)。

## 默认数据目录

| 平台 | Native 数据目录 |
| --- | --- |
| Linux | `/var/lib/printf-client` |
| macOS | `/Library/Application Support/Printf Client` |
| Windows | `%ProgramData%\Printf Client` |

目录中包含 Client 私钥和运行配置。不要手工修改、提交或发送这些文件；权限应限制为管理员可读。

## Linux systemd

推荐 `/etc/printf-client.env`：

```dotenv
PRINTF_TOKEN=replace-with-client-token
PRINTF_SERVER=https://moon.example.com
```

权限：

```bash
sudo chown root:root /etc/printf-client.env
sudo chmod 0600 /etc/printf-client.env
```

systemd unit 已设置 Native 运行模式。环境文件不是 shell 脚本，不要添加 `export`。

## 实例限制

- 一个 token 只运行一个 Client。
- 不要同时用同一 token 运行 Docker Client 和 Native Client。
- 切换运行方式前先停止旧 Client，并确认控制面已经显示离线。
- 不要通过删除本地数据目录来规避 token 冲突；需要迁移或重置时先刷新 token。
