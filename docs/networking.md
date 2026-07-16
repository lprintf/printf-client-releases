# 网络与 Mapping

## Native direct Mapping

Native Client 数据路径：

```text
Internet user
  -> HTTPS Public Node
  -> WireGuard tunnel
  -> Client printf0 IP:target_port
  -> local service
```

控制面中的默认本地 Target Service：

```text
http://[默认本地]:8080
```

对应 Node upstream：

```text
http://<client-wireguard-ip>:8080
```

## 服务监听地址

可达：

```text
0.0.0.0:8080
<client-wireguard-ip>:8080
```

通常不可达：

```text
127.0.0.1:8080
```

`127.0.0.1` 只接受本机 loopback 流量，Node 发往 WireGuard IP 的连接不会自动转发到 loopback。

## 本机 Docker 服务

Native Client 要访问 Docker 应用，需要把应用端口发布到宿主机可从 WireGuard 地址访问的位置，例如：

```yaml
services:
  app:
    image: example/app
    ports:
      - "8080:8080"
```

应用容器内也必须监听 `0.0.0.0:8080`。

不要把 `127.0.0.1:8080:8080` 当作 WireGuard 可达入口；它只绑定宿主机 loopback。

## Docker alias

以下 Mapping：

```text
https://a-uname.example.com -> http://a:8080
```

要求 Docker Bridge Client 加入与 `a` 服务相同的 external network。Native Client 不在 Docker DNS 网络中，不能解析 Compose alias，也不能开启 `PRINTF_BRIDGE_MODE=true`。

## 与 Node 同机

Native Client 不应与 Public Node 同机。即使 Node 使用 `wg0`、Client 使用 `printf0`，两者仍可能为同一个 WireGuard subnet 安装冲突路由。

同机 Node + Client 应使用隔离网络命名空间的 Docker Bridge Client，并验证 Node 公网 UDP Endpoint 的 hairpin 路径。

## 防火墙

需要允许：

- Client 出站访问控制面 HTTPS。
- Client 出站访问 Node WireGuard UDP Endpoint。
- Client 本机目标服务接受来自 WireGuard interface 的 TCP 流量。

不需要把 Client 的目标端口直接暴露到公网。

## 多 Node

Client join 可以收到多个 Node。相同 `allowed_ips` subnet 只选择一个 peer，其余作为 standby；不同 subnet 可以同时安装。

如果两个 Node 宣告重叠 subnet，路由结果不可预测，应在控制面修复 Node 网络规划。
