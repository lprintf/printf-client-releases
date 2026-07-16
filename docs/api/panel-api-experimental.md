# Client Panel API（Experimental）

这些接口供 Client Panel 和用户自动化管理当前 token 所属资源。

它们尚未成为稳定 v1 API，可能修改 request、response、错误码或路径。不要在关键自动化中假设永久兼容。

## 认证

所有接口使用：

```http
X-Printf-Token: <client-token>
```

token 当前同时具备 Agent Runtime 和 Panel 权限。

## `GET /api/client/config`

返回 Client 的公开配置、capabilities 和全部 Mapping。

```bash
curl --fail-with-body \
  -H "X-Printf-Token: ${PRINTF_TOKEN}" \
  "${PRINTF_SERVER}/api/client/config"
```

示例结构：

```json
{
  "id": 7,
  "name": "office-client",
  "subdomain_prefix": "uname",
  "server_endpoint": null,
  "last_seen": "2026-07-16T10:00:00",
  "mappings": [],
  "capabilities": [],
  "bound_domain_ids": [1],
  "is_online": true
}
```

当前没有独立的 `GET /mappings`；Mapping 列表嵌在 config response 中。

## `PUT /api/client/config`

更新 Client 允许自助修改的字段。当前服务端会忽略用户提交的：

```text
subdomain_prefix
domain_ids
```

当前建议只使用：

```json
{"name": "new-client-name"}
```

## `GET /api/client/domains`

返回该 Client 被绑定和允许使用的域名：

```json
[
  {
    "id": 1,
    "domain_name": "example.com"
  }
]
```

## `POST /api/client/mappings`

创建非 root Mapping。

direct Mapping：

```json
{
  "subdomain": "app",
  "target_host": null,
  "target_port": 8080,
  "domain_id": 1,
  "is_active": true
}
```

Bridge Mapping：

```json
{
  "subdomain": "app",
  "target_host": "app",
  "target_port": 8080,
  "domain_id": 1,
  "is_active": true
}
```

规则：

- `subdomain` 不能为空。
- `target_port` 为 `1..65535`。
- `domain_id` 必须属于当前 Client。
- `target_host` 省略或为 `null` 表示 direct Mapping。
- `target_host` 只能是小写单级 Docker alias。
- `target_host` 需要 Client 注册 `bridge_tcp_relay_v1`。
- `localhost` 和 `host-docker-internal` 是保留值。

当前成功响应只返回：

```json
{"status": "success"}
```

不会返回新 Mapping ID。创建后需要重新调用 `GET /config` 获取列表。这是当前 API 被标记 Experimental 的原因之一。

## `PUT /api/client/mappings/{mapping_id}`

部分更新 Mapping：

```json
{
  "target_port": 8081,
  "is_active": true
}
```

可选字段：

```text
subdomain
target_host
target_port
domain_id
is_active
```

root Mapping 只允许修改：

```text
target_host
target_port
is_active
```

## `DELETE /api/client/mappings/{mapping_id}`

删除当前 Client 的非 root Mapping。

root Mapping 返回 `400`：

```json
{"detail": "Cannot delete root mapping"}
```

## `POST /api/client/mappings/{mapping_id}/toggle`

切换 `is_active`：

```bash
curl --fail-with-body \
  -X POST \
  -H "X-Printf-Token: ${PRINTF_TOKEN}" \
  "${PRINTF_SERVER}/api/client/mappings/17/toggle"
```

这是 toggle 而不是显式 set；并发自动化可能产生竞态。需要确定最终状态时应优先使用 `PUT` 明确提交 `is_active`。

## Mapping stats

综合状态：

```text
GET /api/client/mappings/{mapping_id}/stats
```

特定 Node：

```text
GET /api/client/mappings/{mapping_id}/stats/{node_id}
```

响应可能包含：

```text
client_online
node_count
node_stubs
wg_healthy
ping_latency_ms
target_offline
http_status
relay_status
```

stats response 当前是运维视图，不是稳定机器协议。

## 公共 API 后续方向

正式 v1 前建议增加：

- `/api/client/v1/...`；
- 独立 `GET /mappings`；
- create 返回 Mapping object/ID；
- 明确的 error code；
- 幂等创建键；
- token scope；
- rate limit 文档；
- 取消依赖 `Host` 自动推断 domain 的公共契约。
