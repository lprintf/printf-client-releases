# Security Policy

Printf Client 以 root 或 Administrator 权限管理 WireGuard，因此 token、私钥和发布包完整性都属于安全边界。

## 支持版本

当前仅支持 Latest Release。发现安全问题后可能要求升级到新版本，不保证旧版本继续收到修复。

## 私密报告漏洞

不要为安全问题创建公开 Issue。请使用本仓库的 GitHub Private Vulnerability Reporting：

```text
Security -> Report a vulnerability
```

报告应包含：

- 受影响版本和平台。
- 可复现步骤。
- 预期与实际行为。
- 影响范围。
- 已脱敏的日志。

不要提交：

- Client token。
- WireGuard private key。
- 完整 `printf0.conf` 或 `wg0.conf`。
- SSH、Cloudflare、GitHub 或其他平台凭据。
- 未经允许的生产用户数据。

## Token

`PRINTF_TOKEN` 是 bearer credential。当前同一个 token 同时具备 Client runtime 和 Client Panel 权限。

- 一个 token 只能运行一个 Client 实例。
- token 不应出现在 URL、Issue、截图或 Git 历史中。
- systemd 环境文件应为 `0600`。
- Windows 上应仅允许 Administrators 和 SYSTEM 读取配置目录。
- 怀疑泄露时立即在控制面刷新 token，并停止旧 Client。

## WireGuard 私钥

默认私钥位置：

| 平台 | 路径 |
| --- | --- |
| Linux | `/var/lib/printf-client/private.key` |
| macOS | `/Library/Application Support/Printf Client/private.key` |
| Windows | `%ProgramData%\Printf Client\private.key` |

不要复制私钥到日志、备份工单或聊天工具。迁移主机时，优先生成新密钥并重新 join。

## 发布包

下载后至少验证 `.sha256`。当前版本尚未完成独立发布签名、macOS notarization 和 Windows Authenticode，因此操作系统可能显示未知发布者警告。

不要通过关闭系统安全机制来长期绕过警告。正式生产部署应记录下载来源、版本和哈希。

## 安全边界

公开 Client 协议不是秘密。服务端安全依赖：

- token 验证和轮换；
- Client 与 Mapping 的资源归属校验；
- WireGuard peer 和 assigned IP 约束；
- capability 校验；
- 服务端审计、限流和输入校验。

协议公开不授权访问任何 Client、Node 或生产环境。
