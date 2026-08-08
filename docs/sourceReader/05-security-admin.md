# 安全、IAM、KMS、TLS 与 Admin 控制面

## 1. 身份与凭据

`crates/credentials` 定义 access key、secret key、session token、过期时间和序列化。凭据对象的调试输出和日志必须脱敏。根凭据、长期用户凭据、service account 和 STS 临时凭据在生命周期及授权来源上不同。

`crates/iam` 的主要模块：

- `manager` / `sys`：IAM 生命周期与全局系统。
- `store`：IAM 对象的持久化。
- `cache`：用户、组、策略和绑定快照。
- `keyring`：访问密钥索引。
- `federation`：外部身份映射。
- `oidc` / `oidc_state`：OIDC 配置和状态。
- `root_credentials`：根身份边界。

IAM 数据通常也存储在对象层，因此启动存在“先有对象层、再加载 IAM”的依赖。缓存是加速结构，不是事实来源；更新必须先保证持久化成功并考虑多节点传播。

## 2. 策略模型

`crates/policy` 负责 policy 文档、statement、action、resource、condition 和变量替换。判定的核心规则是显式 Deny 优先，随后寻找 Allow；没有 Allow 即拒绝。

条件值来自经验证的请求上下文，例如 source IP、TLS、时间、签名版本、对象标签、请求标签和身份属性。资源 ARN 的 bucket/object 编码、通配符和特殊字符处理必须与 S3 兼容。

Bucket policy、identity policy、group policy、session policy 和 service-account 限制可能同时参与。代码阅读时应查清每一层是在扩大还是缩小权限，尤其 session policy 通常只能收窄父身份权限。

## 3. Keystone 与 OIDC/STS

`auth_keystone.rs` 和 `crates/keystone` 实现 OpenStack Keystone 集成。外部 token 验证结果要映射成内部主体和 policy，并处理 endpoint 故障、缓存、过期和时钟偏差。

OIDC/STS 路由签发临时凭据。需要验证 issuer、audience、签名算法、nonce/state、时间声明和角色映射。外部 identity 不应直接控制内部 policy 名称，除非经过受控映射。

## 4. KMS 与对象加密

`crates/kms` 把 KMS 抽象、密钥管理、加密上下文、缓存、审计和后端适配组织起来。对象加密通常采用 envelope encryption：生成数据密钥，用其加密对象，再用 KMS master key 加密数据密钥。

`rustfs/src/admin/handlers/kms*.rs` 覆盖 KMS 状态、key 管理、备份、动态配置、生命周期和审计。`kms_deletion_gate.rs` 对密钥删除施加额外保护，因为删除仍被对象引用的 key 会造成永久数据不可读。

KMS 失败应明确区分：未配置、不可达、未授权、key 不存在、密文损坏和超时。把这些都映射成通用内部错误会妨碍安全审计和恢复。

## 5. TLS 热更新

`crates/tls-runtime` 将 TLS 文件布局、证书解析、material snapshot、fingerprint、server resolver、outbound connector 和 reload coordinator 分开。

安全的热更新过程是：检测文件变化，读取完整材料，验证证书链/私钥匹配和有效期，构造新 snapshot，通知各 consumer，最后发布 generation。若某个 consumer 应用失败，状态端点应能看出 generation 偏差，不能宣称全局更新成功。

`rustfs/src/admin/handlers/tls_debug.rs` 提供诊断面，输出应展示状态和指纹而非私钥或敏感内容。

## 6. Admin 路由结构

`rustfs/src/admin/router.rs` 与 `route_policy.rs` 注册并约束 Admin API。`admin/auth.rs`、`access_key_identity.rs`、`site_replication_identity.rs` 处理控制面身份。

handlers 覆盖：

- 用户、组、策略、service account、STS、OIDC。
- 配置、模块开关、通知和审计运行时配置。
- bucket metadata、quota、lifecycle transition、replication。
- heal、rebalance、pool、durability、scanner、cluster snapshot。
- metrics、trace、profile、diagnostics、inspect archive。
- KMS、TLS、对象数据缓存、table catalog。
- plugin/extension catalog 与 instance。

Admin handler 通常应是薄适配：认证和 action 检查，输入大小与 schema 校验，调用 service/usecase，统一错误响应和审计。危险操作应支持 dry-run、明确目标范围和幂等标识。

## 7. Plugin 与 extension 边界

`crates/extension-schema` 定义 extension kind、runtime boundary、capability、S3 hook、运维诊断和 profiler contract。`crates/targets` 管理 manifest、catalog、plugin、instance store、sidecar protocol 和运行时 adapter。

`admin/plugin_contract.rs`、`plugins_catalog.rs`、`plugins_instances.rs` 和 `extensions.rs` 是外部扩展进入控制面的边界。必须验证 manifest/schema，限制外部安装策略，隔离 sidecar，保护 secret，并确保列表/诊断响应脱敏。

S3 hook 位于请求热路径，扩展超时或故障不能无限阻塞对象操作。hook 是否 fail-open/fail-closed 必须由具体安全语义决定，并在 contract 中明确。

## 8. 审计与敏感数据治理

`crates/audit` 由 entity、pipeline、registry、factory、system 和 observability 组成。审计事件应包含主体、动作、资源、结果、请求关联信息和目标投递状态，但不包含 secret、token、SSE-C key 或完整敏感配置。

`crates/security-governance` 提供 admin 权限矩阵、redaction、serde policy 和 supply-chain 约束。它的价值是把安全规则变成可复用/可测试的代码，而不是依赖 handler 自觉。

审计投递失败策略需区分：业务操作是否已经提交、事件是否可重放、队列是否持久化。不能因为外部审计目标短暂不可用而让已经落盘的对象写表现成失败，除非系统明确采用同步强审计语义。

