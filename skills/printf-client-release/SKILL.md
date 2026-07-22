---
name: printf-client-release
description: 通过 Docker Compose、Printf Client 和 Client API，把现有 Compose HTTP 服务发布为公网 HTTPS 地址。用于用户要求部署、公开、映射、上线或检查 Web 服务，并已提供 Printf token、控制面地址或期望域名的场景。
---

# 使用 Printf 发布 Compose 服务

## 边界

- 有 Docker 时使用 Compose Bridge 模式。镜像已包含 WireGuard 和网络工具，不在宿主机额外安装 `wg`、`wg-quick` 或其他 Client 辅助工具。
- 先读取本目录的 `compose.service.yml` 和 `docker-compose.yml`。前者是应用侧入口服务参考，后者是通用 Client；Windows Docker Desktop 使用 `docker-compose.alpine.yml`。
- 只把应用的公网 HTTP 入口服务加入 Client 所在的 external network。数据库、Redis、内部 API 等服务留在项目默认网络。
- 使用唯一且稳定的 Docker alias 作为 API 的 `target_host`，使用容器内部监听端口作为 `target_port`。不为 Printf 向宿主机发布应用端口。
- 应用已有认证网关时发布认证网关；否则发布前端 Nginx 或单体应用入口。不绕过现有认证层。
- 一个 token 只运行一个 Client。控制面显示在线但当前 Compose 没有对应 Client 时停止，不启动第二个实例。
- 不调用 Admin API，不刷新 token，不删除映射。删除或替换用户资源前另行确认。

## 输入

先从项目和用户提供的信息取得以下值，不猜测端口或入口服务：

| 值 | 规则 |
| --- | --- |
| `PRINTF_TOKEN` | 必填 secret，禁止提交、回显或写入命令输出。 |
| `PRINTF_SERVER` | 控制面 origin，默认 `https://moon.lprintf.com`，末尾不要带 `/`。 |
| `PRINTF_BRIDGE_NETWORK` | Client 和应用入口共享的 external network，默认 `gateway`。 |
| `PRINTF_TARGET_HOST` | 应用入口的 Docker alias，只能是小写 `a-z0-9-`，不能以 `-` 开头或结尾，最长 63 字符，且不能是 `localhost` 或 `host-docker-internal`。 |
| `PRINTF_TARGET_PORT` | 应用入口容器内的 HTTP 监听端口，范围 `1..65535`。 |
| 发布方式 | 默认站点，或带非空短名称的项目站点。 |
| 基础域名 | 从 `/api/client/domains` 返回值选择。只有一个时可直接使用；多个且用户未指定时必须询问。 |

项目短名称遵守与 alias 相同的 DNS label 规则。不要自行拼接最终域名，以 API 返回的 `full_subdomain` 为准。

## 工作流

### 1. 调整应用 Compose

读取用户项目的 Compose、Dockerfile、反向代理和健康检查配置，确定唯一的公网 HTTP 入口及其容器内部端口。参考 `compose.service.yml` 修改现有 Compose：

1. 保留项目的私有默认网络。
2. 只给入口服务增加 external network 和唯一 alias。
3. 不给数据库、Redis 或纯内部服务增加 external network。
4. 不添加仅供 Printf 使用的 `ports`。已有 `ports` 若还承担本机访问用途，不擅自删除。
5. 保留或补充项目原生可执行的 healthcheck；不要假设镜像一定包含 `curl`、`wget` 或 Python。
6. 有认证网关时，通过 `depends_on` 健康条件等待后端，并把 external network 放在认证网关上。

`compose.service.yml` 是结构参考，不覆盖用户现有服务定义。项目没有 Compose 时才以它为起点，并替换示例中的 service、build、端口和 alias。

### 2. 准备 Compose 和 Client 凭据

确认 `docker`、`docker compose`、`curl` 和 `jq` 可用。在本 skill 目录复制 `.env.example` 为 `.env`，写入真实值；`.env` 已被忽略，提交前仍要确认没有进入 Git。

默认设置 `PRINTF_CLIENT_COMPOSE=./docker-compose.yml`，使用 `lprintf/printf:v0.2`。只有用户明确在 Windows Docker Desktop 环境使用验证过的 Alpine 变体时，才改为 `PRINTF_CLIENT_COMPOSE=./docker-compose.alpine.yml`，使用 `lprintf/printf:alpine-v0.2`。不要根据 shell 名称猜测平台。

关闭 shell trace 并加载环境：

```bash
set +x
set -a
. ./.env
set +a
: "${PRINTF_TOKEN:?PRINTF_TOKEN is required}"
: "${PRINTF_TARGET_HOST:?PRINTF_TARGET_HOST is required}"
: "${PRINTF_TARGET_PORT:?PRINTF_TARGET_PORT is required}"
PRINTF_SERVER="${PRINTF_SERVER:-https://moon.lprintf.com}"
PRINTF_SERVER="${PRINTF_SERVER%/}"
PRINTF_BRIDGE_NETWORK="${PRINTF_BRIDGE_NETWORK:-gateway}"
PRINTF_CLIENT_COMPOSE="${PRINTF_CLIENT_COMPOSE:-./docker-compose.yml}"
```

确认 `PRINTF_TARGET_HOST` 与应用 Compose 的 alias 完全一致。不要运行会输出完整环境的 `docker compose config`；Client Compose 只用 `docker compose ... config --quiet` 校验。

### 3. 启动应用

返回应用 Compose 所在目录，创建共享网络并启动应用。网络已存在时复用：

```bash
if ! docker network inspect "${PRINTF_BRIDGE_NETWORK}" >/dev/null 2>&1; then
  docker network create "${PRINTF_BRIDGE_NETWORK}"
fi

docker compose config --quiet
docker compose up -d --build
docker compose ps
```

连接失败、容器不健康或入口进程未监听目标端口时停止并修复应用，不继续创建公网映射。完成后返回本 skill 目录执行 Client 命令。

### 4. 验证 token 和 Client 归属

```bash
config_json="$(curl --fail-with-body --silent --show-error \
  -H "X-Printf-Token: ${PRINTF_TOKEN}" \
  "${PRINTF_SERVER}/api/client/config")"
```

检查 `is_online`、`mappings` 和 `capabilities`：

- `is_online=true` 且当前 Client Compose 的容器正在运行：复用它。
- `is_online=true` 且当前 Compose 没有 Client 容器：停止并说明 token 正由其他实例使用。
- `is_online=false`：继续启动本目录 Client。
- API 返回 `403`：停止并要求核对 token 和控制面地址。

不重试或重建 `wg_data/` 来掩盖 token 冲突。

### 5. 启动 Bridge Client

```bash
docker compose -f "${PRINTF_CLIENT_COMPOSE}" config --quiet
docker compose -f "${PRINTF_CLIENT_COMPOSE}" pull
docker compose -f "${PRINTF_CLIENT_COMPOSE}" up -d
docker compose -f "${PRINTF_CLIENT_COMPOSE}" ps
```

最多等待 60 秒，再次读取 `/api/client/config`。只有 `is_online=true` 且 `capabilities` 包含 `bridge_tcp_relay_v1` 时继续。超时后执行：

```bash
docker compose -f "${PRINTF_CLIENT_COMPOSE}" logs --tail 100 printf-client
```

报告原始错误并停止，不继续创建映射。

### 6. 选择域名

```bash
domains_json="$(curl --fail-with-body --silent --show-error \
  -H "X-Printf-Token: ${PRINTF_TOKEN}" \
  "${PRINTF_SERVER}/api/client/domains")"
```

按 `domain_name` 精确匹配并使用对应整数 `id`。返回空数组时停止，提示用户先在控制面为 Client 绑定域名。不要依赖 `Host` header 推断域名。

### 7. 幂等写入 Bridge 映射

重新读取 `/api/client/config`，按以下键查找：

- 默认站点：`is_root=true` 且 `domain_id` 等于所选域名。root 映射必须已经存在，只允许更新。
- 项目站点：`is_root=false`、`subdomain` 等于短名称且 `domain_id` 等于所选域名。

匹配超过一条时停止并报告重复数据。不要调用非幂等的 `/toggle`。

更新已有映射：

```bash
payload="$(jq -cn \
  --arg target_host "${PRINTF_TARGET_HOST}" \
  --argjson target_port "${PRINTF_TARGET_PORT}" \
  '{target_host: $target_host, target_port: $target_port, is_active: true}')"

curl --fail-with-body --silent --show-error \
  -X PUT \
  -H "X-Printf-Token: ${PRINTF_TOKEN}" \
  -H "Content-Type: application/json" \
  --data "${payload}" \
  "${PRINTF_SERVER}/api/client/mappings/${PRINTF_MAPPING_ID}"
```

项目站点没有匹配项时创建映射；默认站点禁止创建空 `subdomain`：

```bash
payload="$(jq -cn \
  --arg subdomain "${PRINTF_SUBDOMAIN}" \
  --arg target_host "${PRINTF_TARGET_HOST}" \
  --argjson target_port "${PRINTF_TARGET_PORT}" \
  --argjson domain_id "${PRINTF_DOMAIN_ID}" \
  '{subdomain: $subdomain, target_host: $target_host, target_port: $target_port, domain_id: $domain_id, is_active: true}')"

curl --fail-with-body --silent --show-error \
  -X POST \
  -H "X-Printf-Token: ${PRINTF_TOKEN}" \
  -H "Content-Type: application/json" \
  --data "${payload}" \
  "${PRINTF_SERVER}/api/client/mappings"
```

`POST` 和 `PUT` 只返回操作状态。成功后必须再次读取 `/api/client/config`，用相同键取得 `id`、`target_host`、`target_port`、`is_active` 和 `full_subdomain`。字段与请求不一致或 `full_subdomain` 为空时视为失败。

### 8. 验证公网链路

1. 调用 `GET /api/client/mappings/{id}/stats`，检查 `client_online`、`best.wg_healthy`、`best.target_offline`、`best.http_status` 和 `best.relay_status`。
2. 请求 `https://{full_subdomain}` 的已知健康检查路径，限制连接和总超时，并保留实际 HTTP 状态。
3. 只有映射已激活、目标 alias 与端口一致且公网请求符合应用预期时才报告成功。
4. 失败时根据原始响应区分 Docker DNS/alias、Client、WireGuard、Node upstream、DNS/TLS 和应用 HTTP 错误，不做猜测性修复。

最终只返回公网 HTTPS URL、对应入口服务/alias/容器端口和验证结果。不要返回 token、完整 `.env`、私钥或 WireGuard 配置。

## API 约定

所有调用都使用 `X-Printf-Token: ${PRINTF_TOKEN}`：

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `GET` | `/api/client/config` | 验证 token，读取在线状态、能力、映射和最终域名。 |
| `GET` | `/api/client/domains` | 读取 Client 已绑定的基础域名。 |
| `POST` | `/api/client/mappings` | 创建非 root 映射。 |
| `PUT` | `/api/client/mappings/{id}` | 幂等更新 alias、端口和启用状态。 |
| `GET` | `/api/client/mappings/{id}/stats` | 检查公网 Node 到应用入口的数据路径。 |

始终用 `curl --fail-with-body` 暴露非 2xx 响应。遇到 `403`、`409` 或 `422` 时保留响应体并停止当前步骤，不静默降级。
