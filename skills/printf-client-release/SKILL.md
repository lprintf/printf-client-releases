---
name: printf-client-release
description: 通过 Docker Compose Bridge 或 Native Client 和 Client API，把现有 HTTP 服务发布为公网 HTTPS 地址。用于用户要求部署、公开、映射、上线或检查 Web 服务，Docker 可能不可用、尚未配置 Printf token，或 Native 二进制疑似故障需要生成脱敏 Issue 的场景。
---

# 使用 Printf 发布 HTTP 服务

## 边界

- Docker 和 Compose 正常可用时必须使用 Compose Bridge。镜像已包含运行组件，不在宿主机额外安装 Client 网络辅助工具。
- Docker 命令不存在时，只有目标 HTTP 服务已经在宿主机运行或能脱离 Docker 原生启动，才使用 Native Client，并完整读取 `references/native.md`。Native Client 不能替代 Docker 启动纯 Compose 应用。
- Compose Bridge 先读取本目录的 `compose.service.yml` 和 `docker-compose.yml`。Windows Docker Desktop 使用 `docker-compose.alpine.yml`。
- Compose Bridge 只把公网 HTTP 入口加入 external network，并使用唯一 Docker alias 和容器内部端口。Native Mapping 必须使用默认本地目标和宿主机服务端口。
- 应用已有认证网关时发布认证网关，不绕过现有认证层。数据库、Redis 和内部 API 保持私有。
- 一个 token 只运行一个 Client。控制面显示在线但本机没有对应 Client 时停止，不启动第二个实例。
- 不调用 Admin API，不刷新 token，不删除映射。删除或替换用户资源前另行确认。
- 怀疑 Native 发布包有缺陷时完整读取 `references/report-issue.md`。普通发布授权不等于公开发 Issue 的授权；提交前必须完成脱敏并取得用户确认，安全问题只使用私密漏洞报告。

## 输入

先从项目和用户提供的信息取得以下值，不猜测端口或入口服务：

| 值 | 规则 |
| --- | --- |
| `PRINTF_TOKEN` | 必填 secret，禁止提交、回显或写入命令输出。 |
| `PRINTF_SERVER` | 控制面 origin，默认 `https://moon.lprintf.com`，末尾不要带 `/`。 |
| 运行方式 | Docker 可用时为 Compose Bridge；Docker 不存在且满足 Native 条件时为 Native。 |
| `PRINTF_BRIDGE_NETWORK` | 仅 Compose 使用，Client 和应用入口共享的 external network，默认 `gateway`。 |
| `PRINTF_TARGET_HOST` | 仅 Compose 使用的 Docker alias；Native 必须省略或设为 `null`。 |
| `PRINTF_TARGET_PORT` | Compose 为入口容器内部端口；Native 为宿主机服务端口，范围 `1..65535`。 |
| 发布方式 | 默认站点，或带非空短名称的项目站点。 |
| 基础域名 | 从 `/api/client/domains` 返回值选择。只有一个时可直接使用；多个且用户未指定时必须询问。 |

项目短名称遵守与 alias 相同的 DNS label 规则。不要自行拼接最终域名，以 API 返回的 `full_subdomain` 为准。

## 工作流

### 1. 检查环境和 Token

在修改项目或启动服务前，先读取目标项目的启动方式，并检查 HTTPS 客户端和结构化 JSON 工具。下面示例使用 `curl` 与 `jq`；Windows 没有 `jq` 时可使用 PowerShell `ConvertFrom-Json`、`ConvertTo-Json`，不要为此做字符串拼接或启发式解析。再按以下规则选择且只选择一种运行方式：

- `docker`、`docker compose` 和 Docker daemon 都正常：选择 Compose Bridge。
- Docker 命令不存在：确认应用是否已经原生运行或有可执行的原生启动方式。满足时选择 Native；只有 Compose 定义时停止并说明仍需 Docker。
- Docker 已安装但 daemon 或 Compose 异常：报告原始错误并停止。除非用户明确选择 Native 且应用满足 Native 条件，不静默绕过损坏的 Docker 环境。

不要为了执行 Printf Client 自动安装 Docker。记录选择结果，后续禁止混用 Compose alias 与 Native 默认本地目标。

关闭 shell trace，检查进程环境和本 skill 目录的 `.env`。进程环境中的 `PRINTF_TOKEN` 优先于 `.env`：

```bash
set +x
case "${PRINTF_TOKEN-}" in
  ""|replace-with-client-token|replace_with_your_token)
    printf_token_from_process=""
    ;;
  *)
    printf_token_from_process="${PRINTF_TOKEN}"
    ;;
esac
if [ -f ./.env ]; then
  set -a
  . ./.env
  set +a
fi
if [ -n "${printf_token_from_process}" ]; then
  PRINTF_TOKEN="${printf_token_from_process}"
fi
unset printf_token_from_process
```

空值、`replace-with-client-token` 和 `replace_with_your_token` 都表示未配置。缺少有效 `PRINTF_TOKEN` 时立即停止，读取 `references/get-token.md`，把其中的获取方式告诉用户，并提示用户取得后写入本地进程环境或本 skill 目录的 `.env`。不要猜测 Token，不要要求用户在聊天中发送 Token，也不要继续修改项目、启动 Client 或调用 Mapping API。

Token 有效时设置控制面默认值；只有 Compose Bridge 才设置 Compose 默认值：

```bash
PRINTF_SERVER="${PRINTF_SERVER:-https://moon.lprintf.com}"
PRINTF_SERVER="${PRINTF_SERVER%/}"
```

Compose Bridge 继续执行下面的步骤 2 至 6。选择 Native 时，立即完整读取并执行 `references/native.md`，直到 Native Client 在线，再从下面的“选择域名”继续；不要执行 Compose 专用步骤。

### 2. 调整应用 Compose

读取用户项目的 Compose、Dockerfile、反向代理和健康检查配置，确定唯一的公网 HTTP 入口及其容器内部端口。参考 `compose.service.yml` 修改现有 Compose：

1. 保留项目的私有默认网络。
2. 只给入口服务增加 external network 和唯一 alias。
3. 不给数据库、Redis 或纯内部服务增加 external network。
4. 不添加仅供 Printf 使用的 `ports`。已有 `ports` 若还承担本机访问用途，不擅自删除。
5. 保留或补充项目原生可执行的 healthcheck；不要假设镜像一定包含 `curl`、`wget` 或 Python。
6. 有认证网关时，通过 `depends_on` 健康条件等待后端，并把 external network 放在认证网关上。

`compose.service.yml` 是结构参考，不覆盖用户现有服务定义。项目没有 Compose 时才以它为起点，并替换示例中的 service、build、端口和 alias。

### 3. 准备 Compose 和 Client 配置

需要持久化配置且本 skill 目录没有 `.env` 时，才复制 `.env.example`；不要覆盖已有 `.env`。写入真实 Token 和已从目标项目核实的 alias、容器内部端口；`.env` 已被忽略，提交前仍要确认没有进入 Git。Token 也可以只保留在当前进程环境中。

默认设置 `PRINTF_CLIENT_COMPOSE=./docker-compose.yml`，使用 `lprintf/printf:v0.2`。只有用户明确在 Windows Docker Desktop 环境使用验证过的 Alpine 变体时，才改为 `PRINTF_CLIENT_COMPOSE=./docker-compose.alpine.yml`，使用 `lprintf/printf:alpine-v0.2`。不要根据 shell 名称猜测平台。

确认当前 shell 中的目标配置不是示例值：

```bash
set +x
PRINTF_BRIDGE_NETWORK="${PRINTF_BRIDGE_NETWORK:-gateway}"
PRINTF_CLIENT_COMPOSE="${PRINTF_CLIENT_COMPOSE:-./docker-compose.yml}"
: "${PRINTF_TOKEN:?PRINTF_TOKEN is required}"
: "${PRINTF_TARGET_HOST:?PRINTF_TARGET_HOST is required}"
: "${PRINTF_TARGET_PORT:?PRINTF_TARGET_PORT is required}"
```

确认 `PRINTF_TARGET_HOST` 与应用 Compose 的 alias 完全一致。不要运行会输出完整环境的 `docker compose config`；Client Compose 只用 `docker compose ... config --quiet` 校验。

### 4. 启动应用

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

### 5. 验证 token 和 Client 归属

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

### 6. 启动 Bridge Client

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

### 7. 选择域名

```bash
domains_json="$(curl --fail-with-body --silent --show-error \
  -H "X-Printf-Token: ${PRINTF_TOKEN}" \
  "${PRINTF_SERVER}/api/client/domains")"
```

按 `domain_name` 精确匹配并使用对应整数 `id`。返回空数组时停止，提示用户先在控制面为 Client 绑定域名。不要依赖 `Host` header 推断域名。

### 8. 幂等写入 Mapping

重新读取 `/api/client/config`，按以下键查找：

- 默认站点：`is_root=true` 且 `domain_id` 等于所选域名。root 映射必须已经存在，只允许更新。
- 项目站点：`is_root=false`、`subdomain` 等于短名称且 `domain_id` 等于所选域名。

匹配超过一条时停止并报告重复数据。不要调用非幂等的 `/toggle`。

Compose Bridge 使用非空 Docker alias；更新已有映射：

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

Compose Bridge 的项目站点没有匹配项时创建映射；默认站点禁止创建空 `subdomain`：

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

Native 必须使用 `references/native.md` 中 `target_host: null` 的 payload，不得复用上面的 Compose payload。

`POST` 和 `PUT` 只返回操作状态。成功后必须再次读取 `/api/client/config`，用相同键取得 `id`、`target_host`、`target_port`、`is_active` 和 `full_subdomain`。Compose 的 `target_host` 必须等于 alias；Native 的 `target_host` 必须为 `null`。字段与请求不一致或 `full_subdomain` 为空时视为失败。

### 9. 验证公网链路

1. 调用 `GET /api/client/mappings/{id}/stats`，检查 `client_online`、`best.wg_healthy`、`best.target_offline`、`best.http_status` 和 `best.relay_status`。
2. 请求 `https://{full_subdomain}` 的已知健康检查路径，限制连接和总超时，并保留实际 HTTP 状态。
3. 只有映射已激活、所选模式的目标与端口一致且公网请求符合应用预期时才报告成功。
4. 失败时根据原始响应区分 Compose DNS/alias、Native 本地监听、Client、加密传输、Node upstream、DNS/TLS 和应用 HTTP 错误，不做猜测性修复。

最终只返回公网 HTTPS URL、所选运行方式、对应入口服务与端口和验证结果。不要返回 token、完整 `.env`、私钥或完整传输配置。只有问题符合 Native 发布包缺陷条件时才读取并执行 `references/report-issue.md`。

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
