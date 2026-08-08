# HTTP、S3 与应用层

## 1. HTTP 服务栈

`rustfs/src/server/` 是网络请求进入 RustFS 后的第一层：

- `http.rs`：HTTP server 的主要装配和请求服务。
- `hybrid.rs`：同一监听入口上的协议或 TLS/明文适配。
- `layer.rs`：中间件组合顺序。
- `cors.rs`、`compress.rs`：跨域和响应压缩。
- `rate_limit.rs`：请求级限流。
- `audit.rs`、`event.rs`：审计上下文和事件挂载。
- `health.rs`、`readiness.rs`：存活与就绪端点。
- `prefix.rs`：路径前缀处理。
- `service_state.rs`：server 可用状态。
- `tls_material.rs`：监听侧证书材料适配。
- `module_switch.rs`：按配置启停功能模块。

中间件顺序是行为的一部分。请求 ID、代理地址归一化和 body 限制通常应早于鉴权；审计需要看到最终身份和结果；压缩不能改变签名校验使用的原始请求表示；限流拒绝也应进入指标和必要的审计记录。

`trusted-proxies` crate 负责可信代理链。只有来源 IP 属于受信任网段时，才能接受转发头中的客户端地址。其 cloud metadata 模块可发现 AWS/Azure/GCP 网络范围，但外部元数据失败不能静默扩大信任边界。

## 2. 路由分类

主 router 至少承载四类路由：

1. S3 数据面：桶、对象、multipart、tagging、ACL、Select 等。
2. Admin 控制面：IAM、配置、诊断、heal、rebalance、KMS、插件等。
3. 运维端点：health、readiness、metrics、profiling。
4. 节点内部端点：storage RPC、peer REST/gRPC、控制信号。

这些路由不能只靠 URL 区分安全级别。每条路由还要绑定 body 上限、认证方式、动作权限、超时、审计策略和是否允许匿名访问。

## 3. S3 handler 分组

`rustfs/src/storage/s3_api/` 是主要 S3 HTTP 适配层：

- `bucket.rs`：创建、删除、列举桶以及桶级操作入口。
- `multipart.rs`：创建上传、上传 part、列 part、完成和中止。
- `tagging.rs`：对象与桶标签的 XML/HTTP 转换。
- `acl.rs`：ACL 兼容接口。
- `common.rs`：请求解析、响应构造和共享校验。

handler 的职责是协议适配，而不是实现存储算法。它应完成：解析 path/query/header/XML，校验 S3 约束，鉴权，构造 usecase 参数，把内部错误映射成 S3 error code，并生成兼容响应。

一次典型 `PUT Object`：

```text
HTTP request
  -> 识别 bucket/object 与 query
  -> 校验 Content-Length、checksum、SSE、metadata
  -> SigV4/策略鉴权
  -> 构造有限流和校验能力的 Reader
  -> ObjectUsecase::put_object
  -> ObjectIO/ObjectOperations trait
  -> EcStore 写入临时数据、元数据与最终 rename
  -> ETag/version/checksum 响应
```

一次 `GET Object` 还会处理 range、条件请求、版本、删除标记、解密、解压、checksum 和响应流取消。HTTP 连接断开必须传递到下层，否则昂贵的磁盘读和纠删码恢复会继续占用资源。

## 4. 应用用例层

`rustfs/src/app/` 位于协议与存储之间：

- `object_usecase.rs`：对象读写、复制、删除、stat/list 的业务编排。
- `multipart_usecase.rs`：multipart 生命周期和完成语义。
- `bucket_usecase.rs`：桶级操作。
- `admin_usecase.rs`：控制面业务编排。
- `select_object.rs`：S3 Select 对象查询入口。
- `metadata_route.rs`：对象元数据相关路径。
- `object_data_cache/`：对象数据缓存的规划、填充、失效和 mutation hook。

用例层的价值是把跨模块规则放在一个稳定位置。例如一个对象写入可能同时涉及 IAM、bucket versioning、object lock、SSE、quota、cache invalidation、notification 和 replication。若这些逻辑散落在 HTTP handler 或磁盘实现里，不同协议入口会产生不同语义。

## 5. AppContext 与依赖注入

`app/context/` 管理应用依赖：

- `interfaces.rs` 定义窄接口。
- `handles.rs` 保存共享服务句柄。
- `startup.rs` 在启动阶段组装上下文。
- `server_slot.rs` 管理 server 生命周期槽位。
- `global.rs` 提供必要的全局访问桥梁。
- `runtime_sources.rs` 提供动态配置来源。

请求通常从 server extension 中获取 context，而不是在 handler 内创建对象层或读取全局配置。这样同一请求能共享身份、trace、取消 token、请求 ID、remote address 和能力快照。

## 6. 鉴权与策略判定

`auth.rs` 处理 S3 请求认证，支持普通签名、预签名、流式签名和临时凭据等形态。`signer` crate 同时提供 V2、V4、streaming 和 unsigned trailer 的签名构造/校验基础。

策略判定不是“签名正确即允许”。典型顺序是：

```text
解析认证类型
  -> 定位 access key / session credential
  -> 校验签名、时间窗与 payload hash
  -> 得到主体及 session policy
  -> 形成 action/resource/condition 上下文
  -> IAM policy + bucket policy + 显式拒绝规则
  -> 允许或返回 S3 AccessDenied
```

匿名请求没有签名，但仍可由 bucket policy 授权。临时凭据还要验证 session token、过期时间和 session policy。已有对象标签、来源 IP、TLS、请求头等条件需要按具体 action 读取，不能用未经验证的客户端值替代存储中的真实状态。

## 7. Multipart 的关键不变量

Multipart 不只是“多个 PUT 后拼接”：

- upload ID 必须绑定 bucket/object 和创建时配置。
- 除最后一个 part 外，part size 受 S3 最小值约束。
- 完成请求的 part 顺序、编号和 ETag 必须与已落盘 part 一致。
- 同一 upload 的并发上传和 abort/complete 存在竞争，需要命名空间锁或事务边界。
- 完成后要原子发布对象版本，并清理临时 part；失败时不能暴露半完成对象。
- SSE、checksum、storage class 和 metadata 要从上传创建阶段一致传递到最终对象。

`multipart_usecase.rs` 负责业务语义，`storage-api::MultipartOperations` 定义能力，ECStore 决定 part 如何编码、落盘和合并。

## 8. 对象数据缓存

`app/object_data_cache/` 的 planner 决定请求是否可命中或填充缓存；`body.rs`/`cold_fill.rs` 处理响应体与冷填充；`invalidation.rs` 和 mutation hook 保证写、删、覆盖后旧数据不会继续命中。

缓存 key 必须包含会改变内容的维度，至少包括 bucket、object、version 和 range/编码相关状态。缓存不能绕过权限、object lock、版本和条件请求，也不能把一次请求的 SSE-C 密钥或敏感头持久化。

