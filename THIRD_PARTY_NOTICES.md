# Third-Party Notices

Printf Client 使用 Rust 生态中的第三方组件。第三方组件仍受各自许可证约束。

当前主要直接依赖包括：

| 组件 | 用途 | 常见许可证标识 |
| --- | --- | --- |
| `base64` | WireGuard key 编码 | MIT OR Apache-2.0 |
| `rand_core` | 操作系统随机源 | MIT OR Apache-2.0 |
| `reqwest` | 控制面 HTTPS Client | MIT OR Apache-2.0 |
| `serde` / `serde_json` | API JSON | MIT OR Apache-2.0 |
| `tokio` | 异步网络运行时 | MIT |
| `x25519-dalek` | WireGuard X25519 key | BSD-3-Clause |

传递依赖还包括 Rustls、ring、Hyper、Curve25519 Dalek、ICU4X 相关组件和其他基础库。

本文件是顶层说明，不替代各组件随版本发布的完整许可证文本。正式发布流程将逐步增加：

- version-specific dependency inventory；
- SPDX SBOM；
- 完整第三方许可证归档。

如果你认为某个 Release 缺少必要的第三方声明，请通过 Issue 报告；不要在报告中包含 token 或私钥。
