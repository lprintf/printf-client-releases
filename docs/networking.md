# 网络与 Mapping

## Docker Compose（推荐）

有 Docker 时，公网请求通过 Printf 加密传输通道到达 Client，再由 Client 访问应用入口：

```text
公网 HTTPS
  -> Printf 加密传输通道
  -> Printf Client
  -> Docker alias:target_port
  -> HTTP 入口服务
```

用户不需要配置公网端口、路由或隧道参数。

## 应用 Compose

参考仓库根目录的 `compose.service.yml`：

- 保留应用原有的私有默认网络。
- 只把认证网关、前端 Nginx 或单体应用等 HTTP 入口加入 `gateway` external network。
- 为入口声明唯一的小写 Docker alias，例如 `my-app`。
- 数据库、Redis 和内部 API 不加入 `gateway`。
- 不为 Printf 添加宿主机 `ports`；已有端口若还承担本机访问用途，可以继续保留。

应用入口必须在容器内监听 `0.0.0.0:target_port`，不能只监听容器内的 `127.0.0.1`。

## Target Service

在控制面使用 Docker alias 和容器内部端口，例如：

```text
http://my-app:8080
```

其中：

- `my-app` 必须与 Compose 中的 alias 完全一致。
- `8080` 必须是入口容器实际监听的端口，不是宿主机映射端口。
- alias 只能包含小写 `a-z0-9-`，不能以 `-` 开头或结尾。

## Native 模式

没有 Docker 时，Native Client 使用默认本地端口 Mapping：

```text
公网 HTTPS
  -> Printf 加密传输通道
  -> Native Client
  -> 本地服务端口
```

控制面中的 Target Service 例如：

```text
http://[默认本地]:8080
```

目标服务必须监听 `0.0.0.0:8080` 或平台允许的 Client 地址。只监听 `127.0.0.1:8080` 通常不可达。当前版本的 Native 平台依赖见 [平台安装](platforms.md)。

## 网络要求

Client 主机需要允许：

- 出站访问 Printf 控制面 HTTPS。
- 出站访问 Printf 服务节点使用的加密 UDP 传输。
- Docker external network 内 Client 访问应用入口的容器端口。

不需要把 Client 端口或应用目标端口直接暴露到公网。

## 验证

发布完成后依次确认：

1. Client 在控制面显示在线。
2. Mapping 中的 alias 和端口与 Compose 一致。
3. Mapping 状态检查没有目标离线错误。
4. 最终公网 HTTPS 地址返回应用预期状态。

失败时按 [故障排查](troubleshooting.md) 检查，不要通过开放额外公网端口绕过问题。
