# 网络与 Mapping

## Docker Compose Bridge Mapping

有 Docker 时使用 Bridge Client，数据路径为：

```text
Internet user
  -> HTTPS Public Node
  -> WireGuard tunnel
  -> Bridge Client TCP relay
  -> Docker alias:target_port
  -> HTTP entry service
```

Client 和应用入口加入同一个 external network。控制面的 Target Service 例如：

```text
http://my-app:8080
```

其中 `my-app` 是应用入口的唯一 Docker alias，`8080` 是容器内部监听端口。alias 必须是单个小写 DNS label，只能包含 `a-z0-9-`，不能以 `-` 开头或结尾。

## Compose 网络边界

参考仓库根目录的 `compose.service.yml`：

- 应用保留私有默认网络。
- 只把公网 HTTP 入口加入 Client 使用的 `gateway` external network。
- 有认证网关时发布认证网关；否则发布前端 Nginx 或单体应用入口。
- 数据库、Redis 和纯内部 API 不加入 `gateway`。
- 不为 Printf 添加宿主机 `ports`；已有端口若还承担其他用途，可继续保留。

应用入口必须在容器内监听 `0.0.0.0:target_port`，不能只监听容器的 `127.0.0.1`。Docker DNS 会把 alias 解析为入口容器地址。

## Native direct Mapping

没有 Docker 时，Native Client 使用 direct Mapping：

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

Native 目标服务必须监听：

```text
0.0.0.0:8080
<client-wireguard-ip>:8080
```

只监听 `127.0.0.1:8080` 通常不可达。Native Client 不在 Docker DNS 网络中，不能解析 Compose alias，也不能启用 `PRINTF_BRIDGE_MODE=true`。

## 与 Node 同机

Native Client 不应与 Public Node 同机。即使 Node 使用 `wg0`、Client 使用 `printf0`，两者仍可能为同一个 WireGuard subnet 安装冲突路由。

同机 Node + Client 应使用隔离网络命名空间的 Docker Bridge Client，并验证 Node 公网 UDP Endpoint 的 hairpin 路径。

## 防火墙

Bridge 模式需要允许：

- Client 出站访问控制面 HTTPS。
- Client 出站访问 Node WireGuard UDP Endpoint。
- external network 内 Client 连接应用入口的容器端口。

不需要把 Client 或应用的目标端口直接暴露到公网。

## 多 Node

Client join 可以收到多个 Node。相同 `allowed_ips` subnet 只选择一个 peer，其余作为 standby；不同 subnet 可以同时安装。

如果两个 Node 宣告重叠 subnet，路由结果不可预测，应在控制面修复 Node 网络规划。
