# Workspace Crate 全景

下表按职责说明 `crates/` 下的库。它适合在“不知道某段逻辑应该去哪找”时使用。

| crate | 核心职责 | 阅读关键词 |
|---|---|---|
| `audit` | 审计实体、pipeline、目标注册与可观测性 | entity, registry, delivery |
| `checksums` | S3 checksum 算法及 HTTP 表示 | CRC64NVME, SHA256, trailer |
| `common` | 跨 crate 全局类型、readiness、bucket stats、heal channel | `GlobalReadiness`, `SystemStage` |
| `concurrency` | 共享并发策略和快照类型，运行时实现位于 `rustfs/storage` | workload, workers, backpressure |
| `config` | 常量、服务配置、通知/审计目标配置 | env, body limits, server config |
| `credentials` | access/secret/session credential 模型 | expiration, redaction |
| `crypto` | 通用加解密、JWT、license token | stream encryption, JWT |
| `data-usage` | scanner 数据用量持久化模型 | cache snapshot, map encoding |
| `e2e_test` | S3/Admin/集群/KMS/协议端到端测试 | regression, fault proxy, chaos |
| `ecstore` | 对象存储内核、纠删码、磁盘、池、后台服务 | quorum, xl.meta, erasure sets |
| `extension-schema` | 插件/扩展的稳定 contract | capability, hook, profiler |
| `filemeta` | 对象版本与 `xl.meta` 编解码 | `FileInfo`, version, data dir |
| `heal` | 修复任务模型与控制 | heal status, sequence |
| `iam` | 用户、组、策略绑定、OIDC、IAM 持久化与缓存 | manager, store, keyring |
| `io-core` | 底层异步 reader/writer 和进度原语 | pipe, cancellation, progress |
| `io-metrics` | I/O 与系统指标的统一定义 | histogram, sampler, tuner |
| `keystone` | OpenStack Keystone 客户端和认证映射 | token validation, project |
| `kms` | KMS 抽象、后端、密钥生命周期和 envelope encryption | DEK, KEK, context |
| `lifecycle` | ILM 规则模型与时间/过滤条件 | expiration, transition |
| `lock` | 本地、分布式和命名空间锁 | lock quorum, lease, guard |
| `log-analyzer` | `rustfs diagnose` 的离线日志分析 | order-independent, redaction |
| `madmin` | MinIO Admin 协议兼容类型/客户端能力 | admin API, status types |
| `notify` | bucket notification 规则、队列与投递编排 | event filter, queue store |
| `object-capacity` | 对象/容量统计模型和高效聚合 | capacity snapshot |
| `object-data-cache` | 对象数据缓存核心数据结构 | admission, eviction, cache key |
| `obs` | tracing、metrics exporter、OTel 与运行时遥测 | `OtelGuard`, telemetry |
| `policy` | IAM/bucket policy 解析和判定 | action, resource, condition |
| `protocols` | SFTP、FTP(S)、WebDAV、Swift 等协议适配 | protocol server, S3 bridge |
| `protos` | 节点 RPC 生成代码、协议版本和 canonical body | gRPC, capability probe |
| `replication` | 对象/删除/multipart 复制、队列、MRF、resync | status, retry, target |
| `rio` | 对象流 reader wrapper、hash、ETag、压缩、加密 | limit reader, index |
| `rio-v2` | 新版压缩/加密 reader 的渐进替换层 | compatibility, S2 index |
| `s3-ops` | 可共享的 S3 操作定义 | operation classification |
| `s3-types` | S3 event name 等跨模块协议类型 | event compatibility |
| `s3select-api` | S3 Select API、查询计划和对象存储边界 | parser, planner, execution |
| `s3select-query` | SQL 查询引擎实现 | logical/physical planner |
| `scanner` | 命名空间扫描、预算、远端扫描 | cycle, leader epoch |
| `security-governance` | 权限矩阵、脱敏、Serde 与供应链安全规则 | redaction, deny unknown fields |
| `signer` | SigV2/SigV4/streaming 请求签名 | canonical request, presign |
| `storage-api` | 桶、对象、multipart、heal、admin 的稳定 trait | `ObjectIO`, operations |
| `targets` | 通知目标、插件 catalog/manifest/runtime | sidecar, target health |
| `test-utils` | 跨 crate 测试 fixture 和兼容适配 | ecstore test compat |
| `tls-runtime` | TLS 材料、热更新协调与状态 | generation, fingerprint |
| `trusted-proxies` | 可信代理链、云地址范围和 middleware | forwarded headers, CIDR |
| `utils` | 路径、网络、重试、hash、I/O、metadata 兼容 helper | semantic reuse |
| `zip` | 对象归档/ZIP 流式输出 | archive, streaming |

## 依赖方向

可把 workspace 粗略分为五层：

```text
协议/产品层：rustfs, protocols, s3select-api
业务子系统：iam, policy, lifecycle, replication, scanner, notify, audit, kms
存储实现层：ecstore, heal, object-data-cache
稳定契约层：storage-api, filemeta, concurrency, extension-schema, s3-types
基础设施层：io-core, rio, checksums, lock, protos, config, common, utils, obs
```

依赖应总体向下。基础设施 crate 不应反向依赖 `rustfs` 的 handler；`storage-api` 不应依赖 ECStore 实现；独立子系统通过 contract/runtime source 接入主程序，而不是读取主程序私有全局状态。

## 几组容易混淆的 crate

`io-core` 是低层异步 I/O 原语，`rio` 是对象流 wrapper，`io-metrics` 是指标；三者职责不同。

`lifecycle` 表达规则，ECStore 的 `bucket/lifecycle` 执行规则，`services/tier` 管理远端后端和 transition 协调。

`replication` 提供独立领域模型，ECStore 的 replication boundary 把领域模型连接到真实存储，`site_replication` 还会复制 IAM/配置等站点状态。

`notify` 决定事件和队列，`targets` 决定如何投递到外部系统，`audit` 是安全审计领域而非普通 bucket notification。

`config` crate 提供共享配置模型，`rustfs/src/config` 负责进程 CLI 和启动快照，ECStore 自己的 `config/` 是存储子系统适配。

`protos` 定义内部线上协议，`madmin` 面向 Admin 兼容，`storage-api` 是进程内 Rust trait，不能互相替代。

