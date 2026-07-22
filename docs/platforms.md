# 平台安装

## Linux

### 依赖

Debian/Ubuntu：

```bash
sudo apt-get update
sudo apt-get install --yes wireguard-tools
```

Fedora：

```bash
sudo dnf install wireguard-tools
```

Arch Linux：

```bash
sudo pacman -S wireguard-tools
```

Linux Release 使用 musl static PIE，避免依赖目标主机的 glibc 版本。

### 安装

```bash
tar -xzf printf-client-linux-amd64.tar.gz
sudo install -m 0755 printf-client /usr/local/bin/printf-client
```

使用 [systemd unit](../packaging/printf-client.service) 运行时，进程默认以 root 身份执行并管理 `/var/lib/printf-client` 与 `printf0`。

### 检查

```bash
sudo systemctl status printf-client --no-pager
sudo journalctl -u printf-client -n 100 --no-pager
sudo wg show
ip route show
```

## macOS

### 依赖

```bash
brew install wireguard-tools
```

Apple Silicon Homebrew 常见路径：

```text
/opt/homebrew/bin/wg-quick
```

Intel Homebrew 常见路径：

```text
/usr/local/bin/wg-quick
```

### 运行

Apple Silicon：

```bash
sudo env \
  PRINTF_RUNTIME_MODE=native \
  PRINTF_WG_QUICK=/opt/homebrew/bin/wg-quick \
  PRINTF_TOKEN="$PRINTF_TOKEN" \
  PRINTF_SERVER="$PRINTF_SERVER" \
  ./printf-client
```

Intel：

```bash
sudo env \
  PRINTF_RUNTIME_MODE=native \
  PRINTF_WG_QUICK=/usr/local/bin/wg-quick \
  PRINTF_TOKEN="$PRINTF_TOKEN" \
  PRINTF_SERVER="$PRINTF_SERVER" \
  ./printf-client
```

当前 macOS 二进制未 notarize。不要把长期关闭 Gatekeeper 作为部署方案；正式生产使用前应评估本机安全策略。

## Windows

### 依赖

安装 WireGuard for Windows。默认 executable：

```text
C:\Program Files\WireGuard\wireguard.exe
```

### 运行

以管理员身份打开 PowerShell：

```powershell
$env:PRINTF_RUNTIME_MODE = "native"
$env:PRINTF_TOKEN = "replace-with-client-token"
$env:PRINTF_SERVER = "https://moon.example.com"
.\printf-client.exe
```

自定义 WireGuard 路径：

```powershell
$env:PRINTF_WIREGUARD_EXE = "D:\Tools\WireGuard\wireguard.exe"
```

Client 会管理当前版本所需的本地 tunnel service；安装或启动失败时会显式退出。

当前 Windows 二进制未做 Authenticode 签名，SmartScreen 可能显示未知发布者。只从官方 Release 下载并先验证 SHA-256。

## 权限原因

Client 需要管理员权限是因为它必须：

- 管理当前版本的加密传输和本地路由；
- 保存受保护的 Client 私钥与运行配置。

Client 不需要 Docker socket，也不通过服务端执行任意本地 shell 命令。
