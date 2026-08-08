# 启动、运行时与关闭流程

## 1. 启动入口的分层

`rustfs/src/main.rs` 是可执行程序入口，职责应保持很薄：建立进程级环境，解析命令行，并把控制权交给库层。`rustfs/src/lib.rs` 暴露可复用的启动能力，使嵌入式运行、测试和正式二进制能够共享同一套装配逻辑。

启动代码被拆成多个 `startup_*.rs` 文件，这是有意把“一个很长的 main”拆成可审计阶段：

```text
main
  -> startup_entrypoint
  -> startup_preflight
  -> startup_runtime
  -> startup_observability / startup_audit
  -> startup_tls_material / startup_auth
  -> startup_storage
  -> startup_iam / startup_bucket_metadata
  -> startup_services / startup_background
  -> startup_server / startup_protocols
  -> readiness 发布
  -> startup_shutdown
```

这不是单纯的调用顺序。每一步都在建立下一步依赖的运行时不变量。例如，存储对象层发布前不能启动依赖对象层的 scanner；IAM 未加载完成时不能把受保护路由标为 ready；关闭时则要反向撤销这些依赖。

## 2. 配置如何进入系统

`rustfs/src/config/` 负责进程配置的读取和规范化：

- `cli.rs`：命令行结构和子命令入口。
- `opt.rs`：启动选项的解析结果。
- `config_struct.rs`：内部配置模型。
- `snapshot.rs`：把运行时配置固化为可传递快照，避免各模块反复读环境变量。
- `workload_profiles.rs`：按工作负载选择并发、限流和资源策略。
- `info.rs`：用于展示或诊断的配置信息。

`crates/config` 是更底层的配置语义库。`constants/` 集中保存 API、body 大小、磁盘、heal、TLS、代理、scanner、quota、压缩、零拷贝等约束；`audit/` 和 `notify/` 描述各类外部目标；`server_config.rs` 则承载可持久化或动态装载的服务配置。

阅读配置代码时应区分三种值：

1. 原始输入：CLI、环境变量、配置文件或持久化配置。
2. 已验证快照：类型和边界已经确认，可供启动阶段消费。
3. 动态运行时状态：允许热更新，通常经由 watch/channel、原子快照或专门 coordinator 发布。

不要从业务热路径直接读取环境变量。这样会让同一进程内的请求观察到不一致配置，也会绕开启动时验证。

## 3. Preflight 阶段

`startup_preflight.rs` 聚合“启动前必须失败”的检查。典型内容包括：参数组合是否合法、磁盘端点能否解析、文件系统是否满足要求、证书和密钥是否可加载、关键目录权限是否正确。

`startup_fs_guard.rs` 专门隔离文件系统安全检查。这里的失败应阻止服务进入监听状态，因为继续启动会把环境错误转换成运行期数据损坏或不完整写入。

`startup_deadlock.rs` 和 `allocator_reclaim.rs` 属于运行时自保护能力：前者配置挂起/死锁观察，后者处理分配器内存回收。它们不是数据正确性的替代品，而是让异常更早暴露并限制资源恶化。

## 4. Tokio 运行时与能力发布

`startup_runtime.rs` 建立异步运行时和进程级取消树。`startup_runtime_hooks.rs` 安装运行时钩子，`startup_runtime_sources.rs` 把配置来源适配为各模块消费的接口。

`runtime_sources.rs` 与各子模块自己的 `runtime_sources.rs` 体现了一个重要设计：模块不应随意抓取全局配置，而应依赖窄接口。这样测试可以注入替代来源，启动器也能明确列出依赖。

`runtime_capabilities.rs` 描述已启用或已就绪的能力。能力发布与对象构造不同：对象存在只说明初始化完成了一部分；能力状态还要反映依赖是否可用、协议版本是否匹配以及后台协调是否启动。

`crates/common::GlobalReadiness` 和 `SystemStage` 是全局 readiness 的核心表达。健康检查应基于阶段，而不是简单返回“进程还活着”。常见阶段可理解为：正在启动、基础存储可用、依赖服务已装配、对外就绪、正在关闭。

## 5. 可观测性、审计和 TLS 的启动顺序

`startup_observability.rs` 初始化 tracing、metrics 和导出器。它需要足够早，才能观察后续初始化失败；又必须在配置验证后，避免把不可信或敏感配置直接写入日志。

`startup_audit.rs` 组装审计 pipeline。审计与普通日志的区别是审计事件有安全与合规语义，通常需要结构化实体、目标投递状态和失败策略。

`startup_tls_material.rs` 构造 TLS 证书快照。`crates/tls-runtime` 将证书来源、指纹、server resolver、outbound client material、reload coordinator 和状态快照分离，使热更新可以采用“先验证新材料，再原子发布”的方式，而不是就地修改正在使用的对象。

## 6. 存储和控制面初始化

`startup_storage.rs` 读取 endpoint，创建 `EcStore`/对象层，完成 format 检查，并发布存储句柄。多磁盘或分布式模式下，这里会触发池和 set 的发现、格式一致性检查、远端磁盘连接与 quorum 判断。

`startup_bucket_metadata.rs` 在对象层可用后装载桶级配置，例如 versioning、policy、lifecycle、replication、object lock、quota、notification target 等。它们不能早于存储初始化，因为配置本身通常也持久化在对象系统中。

`startup_iam.rs` 构造 IAM manager、装载用户/组/策略和临时凭据。`startup_auth.rs` 则组装认证所需依赖。需要注意“认证材料可读取”和“IAM 缓存完成初始同步”是两个不同阶段。

`startup_services.rs` 启动共享服务；`startup_background.rs` 启动 scanner、heal、lifecycle、replication、容量采样等长任务。后台任务应持有派生 cancellation token，并在对象层 ready 后开始。

## 7. 服务监听

`startup_server.rs` 将 router、middleware、TLS material 和 listener 组合起来。`startup_protocols.rs` 启动 SFTP、FTP(S)、WebDAV、Swift 等可选协议。`startup_optional_runtime_sidecars.rs` 和 `startup_embedded_optional.rs` 承载可选扩展或旁路运行时，避免主启动链被 feature 细节淹没。

`startup_embedded.rs`/`embedded.rs` 支持将 RustFS 作为库嵌入其他进程。嵌入模式尤其要关注：谁拥有 runtime、谁触发 shutdown、监听地址是否由宿主提供，以及全局单例能否重复初始化。

## 8. 关闭语义

`startup_shutdown.rs` 是启动的逆过程：

```text
收到 signal 或宿主取消
  -> readiness 切换为 shutting-down
  -> 停止接受新请求
  -> 取消后台任务
  -> 等待在途任务在 deadline 内结束
  -> flush 审计/通知/遥测
  -> 关闭 listener、RPC channel 和存储资源
```

正确关闭的关键不是“所有 task 都 join”，而是明确每类任务的语义：可立即取消的观察任务、必须完成当前事务的持久化任务、可在重启后重放的队列任务，以及到期后必须强制终止的外部调用。

