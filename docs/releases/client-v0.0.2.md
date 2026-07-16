# Printf Client v0.0.2

首个公开二进制分发版本。

## 来源

```text
Private source repository: lprintf/printf-next
Source commit: 76a2a3615bc6b4046f3fc69e47c9446e1e5985b5
Private build run: https://github.com/lprintf/printf-next/actions/runs/29491927600
```

私有源码仓库和 build run 需要相应 GitHub 权限；公开用户以本仓库 Release 资产和 SHA-256 为准。

## 平台

- Linux amd64，musl static PIE。
- Linux arm64，musl static PIE。
- macOS Intel。
- macOS Apple Silicon。
- Windows amd64。

## 功能

- Native Client 模式。
- Docker Client 兼容模式。
- WireGuard key 本地生成与持久化。
- join 和 heartbeat。
- 多 Node peer 配置。
- Docker Bridge TCP relay capability。
- Windows WireGuard tunnel service。
- Linux/macOS `wg-quick`。

## 已知限制

- macOS 尚未 notarize。
- Windows 尚未 Authenticode 签名。
- Release 尚未提供独立签名或 SBOM。
- Native Client 不支持 Docker service alias。
- Native Client 不应与 Public Node 同机。
- Client Protocol 仍为 v0。

## 校验

每个压缩包附带同名 `.sha256` 文件。下载两者后按平台执行：

```bash
sha256sum --check <file>.sha256
```

或：

```bash
shasum -a 256 -c <file>.sha256
```
