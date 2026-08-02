# Native Client 降级流程

只在主 Skill 已判定 Docker 命令不存在，并且目标 HTTP 服务能脱离 Docker 运行时执行本流程。Native Client 不能启动仅由 Compose 定义的应用。

## 1. 确认目标服务

1. 找到应用的原生启动命令、HTTP 入口和实际监听端口，不从示例配置猜测。
2. 启动应用并确认它持续运行。
3. 确认服务监听 `0.0.0.0:<port>` 或平台允许的 Client 地址。只监听 `127.0.0.1` 通常无法接收 Printf 通道流量。
4. 使用本机 HTTP 请求验证已知路径和预期状态。应用未就绪时停止，不下载或启动 Client 掩盖应用错误。

将核实后的端口保存为整数 `PRINTF_TARGET_PORT`，范围为 `1..65535`。Native 不使用 `PRINTF_TARGET_HOST`、`PRINTF_BRIDGE_NETWORK` 或 Docker alias。

## 2. 选择官方发布包

只使用 `https://github.com/lprintf/printf-client-releases/releases/latest` 的资产：

| 平台 | 架构 | 文件 |
| --- | --- | --- |
| Linux | amd64 / x86_64 | `printf-client-linux-amd64.tar.gz` |
| Linux | arm64 / aarch64 | `printf-client-linux-arm64.tar.gz` |
| macOS | Intel / x86_64 | `printf-client-macos-amd64.tar.gz` |
| macOS | Apple Silicon / arm64 | `printf-client-macos-arm64.tar.gz` |
| Windows | amd64 / x86_64 | `printf-client-windows-amd64.zip` |

没有精确匹配的平台资产时停止，不运行其他系统或架构的文件。通过 GitHub Releases API 读取 Latest Release，按资产完整文件名选择 `browser_download_url`，同时选择同名 `.sha256`。下载到目标项目之外不会被 Git 提交的持久私有目录；禁止通过管道直接执行远程内容。

## 3. 验证发布包

下载资产和同名 `.sha256` 后，在同一目录校验：

- Linux：`sha256sum --check <archive>.sha256`
- macOS：`shasum -a 256 -c <archive>.sha256`
- Windows：用 `Get-FileHash -Algorithm SHA256` 与校验文件的第一列比较，忽略十六进制大小写。

校验失败时禁止解压和执行。删除本次失败的下载文件，从官方 Latest Release 重新下载一次并再次校验；第二次仍失败时保留文件名、版本、预期哈希和实际哈希，读取 `references/report-issue.md`。不得保留或运行哈希不一致的二进制。

## 4. 检查平台要求

完整读取官方[平台安装](https://github.com/lprintf/printf-client-releases/blob/main/docs/platforms.md)和[配置参考](https://github.com/lprintf/printf-client-releases/blob/main/docs/configuration.md)。Native 当前需要平台网络支持和管理员权限。

- 安装系统依赖、注册系统服务或提升权限前，说明具体动作并取得用户确认。
- 不长期关闭 Gatekeeper、SmartScreen、防火墙或其他系统安全机制。
- macOS 未 notarize、Windows 未做 Authenticode 是当前已知发布限制，不单独作为缺陷提交。
- 凭据和运行数据必须保存在仅管理员可读的位置，不放进目标项目、Issue 或聊天内容。

## 5. 检查 Token 归属

使用主 Skill 已加载的 `PRINTF_TOKEN` 和 `PRINTF_SERVER` 调用 `GET /api/client/config`：

- `is_online=true` 且能确认本机对应 Native 进程或服务正在运行：复用它。
- `is_online=true` 但本机没有对应 Native 实例：停止，说明 Token 正由其他实例使用。
- `is_online=false`：继续启动 Native Client。
- `403`：停止并要求核对 Token 和控制面地址。

不要同时使用同一 Token 运行 Docker 和 Native Client，也不要删除本地数据目录规避冲突。

## 6. 启动 Native Client

解压到持久私有目录，确认可执行文件名为 Linux/macOS 的 `printf-client` 或 Windows 的 `printf-client.exe`。启动时必须满足：

```text
PRINTF_RUNTIME_MODE=native
PRINTF_TOKEN=<从安全环境读取>
PRINTF_SERVER=https://moon.lprintf.com
PRINTF_BRIDGE_MODE 未设置或为 false
```

按照平台要求使用管理员权限运行。Linux 优先使用仓库提供的 [systemd unit](https://raw.githubusercontent.com/lprintf/printf-client-releases/main/packaging/printf-client.service)；macOS 和 Windows 当前使用管理员管理的持续进程或服务。不要以短生命周期的临时 Agent 子进程冒充持久部署。无法建立持久运行方式时停止并说明限制。

最多等待 60 秒并重新读取 `/api/client/config`。只有 `is_online=true` 时才返回主 Skill 的“选择域名”；Native 不要求 `bridge_tcp_relay_v1` capability。启动失败时保留原始 stderr 或系统日志，先按平台要求、权限、Token 和网络问题诊断。

## 7. 使用 Direct Mapping

执行主 Skill 的“幂等写入 Mapping”时，Native 必须显式发送 `target_host: null`。

更新已有 Mapping：

```bash
payload="$(jq -cn \
  --argjson target_port "${PRINTF_TARGET_PORT}" \
  '{target_host: null, target_port: $target_port, is_active: true}')"
```

创建非 root 项目 Mapping：

```bash
payload="$(jq -cn \
  --arg subdomain "${PRINTF_SUBDOMAIN}" \
  --argjson target_port "${PRINTF_TARGET_PORT}" \
  --argjson domain_id "${PRINTF_DOMAIN_ID}" \
  '{subdomain: $subdomain, target_host: null, target_port: $target_port, domain_id: $domain_id, is_active: true}')"
```

沿用主 Skill 的 `PUT`、`POST`、重新读取配置和公网验证步骤。最终配置的 `target_host` 必须为 `null`，`target_port` 必须等于宿主机服务端口。验证失败时先区分应用监听、Native Client、平台网络支持、Node、DNS/TLS 和应用 HTTP 错误。

只有在完成这些诊断后仍能复现发布包自身问题，才读取 `references/report-issue.md`。
