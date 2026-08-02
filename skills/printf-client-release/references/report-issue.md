# 报告 Native 发布包问题

只在问题可归因于官方 Native 发布包时读取本文件。缺少 Docker、没有 Native 启动方式、缺少 Token、权限不足、平台依赖未安装、应用未监听正确地址或普通 Mapping 配置错误都不是二进制缺陷。

## 可报告问题

- 官方资产与同名 `.sha256` 连续两次校验不一致。
- 支持的平台和架构下载到损坏、无法解压或内容错误的资产。
- 已满足平台要求后，二进制仍稳定复现启动失败、panic、崩溃或错误的平台行为。
- 实际行为与当前 Release 或平台文档明确约定不符。

macOS 未 notarize、Windows 未做 Authenticode 和不支持列表中的平台属于当前已知限制，除非实际行为超出文档说明，否则不要重复提交。

## 先分类安全问题

涉及 Token、私钥、权限绕过、任意代码执行、发布资产被篡改或其他潜在安全影响时，禁止创建公开 Issue。向用户展示脱敏摘要并引导使用 GitHub Private Vulnerability Reporting：

`https://github.com/lprintf/printf-client-releases/security/advisories/new`

不要把安全细节放进公开搜索、Issue、聊天或命令参数。

## 去重和收集

先搜索公开仓库的 open 和 closed Issues，按版本、平台、架构和核心错误文本判断是否重复。已有相同问题时向用户返回现有链接，不创建新 Issue。

新报告只收集：

- Client Release 版本和完整资产文件名。
- 操作系统版本和 CPU 架构。
- 预期与实际行为。
- 最小复现步骤。
- 发布包来源、预期 SHA-256 和实际 SHA-256。
- 已满足的平台要求。
- 完整但已脱敏的错误文本或最小日志片段。

必须删除或替换：

- `PRINTF_TOKEN`、认证 Header 和其他凭据。
- 私钥、完整传输配置和本地数据目录内容。
- 生产域名、内网地址、用户数据和不希望公开的路径或用户名。
- 与复现无关的环境变量和日志。

脱敏后再次扫描凭据特征；无法确认信息是否安全时不提交。

## 用户确认和提交

先生成 `[Bug] ...` 标题和完整 Markdown 正文，向用户展示最终公开内容。普通“发布服务”请求不授权创建公开 Issue；只有用户查看草稿并明确确认后才能提交。

确认后使用已认证的 GitHub 工具提交到：

`https://github.com/lprintf/printf-client-releases/issues/new?template=bug.yml`

使用 GitHub CLI 时，先确认 `gh auth status` 成功，再执行 `gh issue create --repo lprintf/printf-client-releases --title ... --body-file ...`。没有认证工具或提交失败时保留原始错误，只返回可复制的草稿，不尝试索取用户凭据或绕过认证。

提交成功后返回 Issue URL，并明确说明没有附带 Token、私钥或完整传输配置。不要擅自关闭、编辑或评论其他 Issue。
