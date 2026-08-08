# 后台服务与控制循环

## 1. Scanner 与数据用量

`crates/scanner` 遍历命名空间，驱动数据用量统计、生命周期评估和部分修复发现：

- `scanner.rs`：主扫描循环。
- `scanner_folder.rs`：目录级遍历。
- `scanner_io.rs`：存储访问适配。
- `scanner_budget.rs`：预算和节流。
- `sleeper.rs`：按负载让出资源。
- `remote_scanner.rs`：分布式扫描协议。
- `runtime_config.rs`：动态参数。

Scanner 是低优先级后台工作，不能压垮前台请求。它通过 budget、sleeper 和 workload admission 控制速率。分布式模式需要 leader epoch、session ID、sequence 和 protocol version 防止旧 leader 或重放请求重复提交结果。

`crates/data-usage` 定义持久化统计结构；`ecstore/data_usage/local_snapshot.rs` 管理本地快照。统计可最终一致，但快照升级必须保持旧版本可读。

## 2. Lifecycle 与 Tiering

`crates/lifecycle` 表达规则和计算；ECStore 的 `bucket/lifecycle/` 负责把规则应用到真实对象。流程通常是 scanner 发现对象，evaluator 根据对象年龄、标签、版本状态和规则产出动作，再由 runtime boundary 执行过期或 transition。

目录中的 boundary 文件分别隔离配置、metadata、object lock、tagging、replication 和 runtime，说明生命周期决策不能只看时间：

- object lock/retention/legal hold 可阻止删除。
- replication 状态可能要求延迟删除。
- noncurrent version、delete marker 和 free version 有独立语义。
- transition 需要远端 tier 写入成功后再提交本地 metadata。

`transition_transaction.rs`、`tier_delete_journal.rs`、`tier_free_version_recovery.rs` 等实现掉电恢复。一次 transition 可理解为 prepare 远端对象、验证结果、提交本地指针、之后回收本地数据。任何阶段失败都必须有可重试状态。

`services/tier/` 提供 warm backend 抽象和 S3/Azure/GCS/MinIO/aliyun 等后端适配，以及 admin、配置和 mutation intent/peer 协调。

## 3. Replication

`crates/replication` 定义配置、规则、对象/删除/multipart 复制任务、队列、MRF、resync、runtime 和 stats。ECStore `bucket/replication/` 把这些契约接入对象层、bucket metadata、target、scanner、lifecycle 和锁。

复制是异步状态机，不是一次远端 PUT：

```text
本地 mutation 提交
  -> 生成 replication decision
  -> 持久化/入队
  -> worker 发送到目标
  -> 更新版本级 replication status
  -> 失败进入 retry/MRF
  -> resync 对账遗漏或历史对象
```

版本 ID、删除标记、metadata/tagging、SSE、object lock 和 multipart completion 都要保持语义。站点复制还涉及 IAM 和 bucket 配置，不仅是对象数据。

队列必须有背压和持久化策略。远端长期不可用时，不能无限增长内存；任务重放必须幂等，并防止旧任务覆盖目标上的新版本。

## 4. Heal

`crates/heal` 定义 heal 请求、状态和调度。ECStore 的 erasure heal 读取健康 shards，验证元数据与 bitrot，重建缺失 shard，再安全替换目标磁盘内容。

Heal 需要区分：磁盘格式修复、bucket metadata 修复、对象 metadata 修复、数据 shard 修复和扫描式批量 heal。修复时不能把一个旧但可读的副本当作最新版本覆盖新数据；应依据 mod time、version、data dir 和 quorum 选择权威状态。

heal 控制 RPC 有单独协议版本与消息大小限制，便于混合版本集群协调。进度、取消和结果应可观察，避免 admin 请求断开后后台任务失控。

## 5. Notification 与 Targets

`crates/notify` 管理事件规则、队列和投递；`crates/targets` 实现 webhook、Kafka、NATS、MQTT、AMQP、Redis、Postgres、MySQL、Pulsar 等目标。

对象事件应在 mutation 成功后产生。事件过滤基于 event name、prefix/suffix 和配置；投递失败通常进入 queue store 重试。目标配置包含凭据，加载、Admin 返回和日志都必须脱敏。

每个 target 需要健康状态、超时、重连、背压和 delivery snapshot。同步等待外部 broker 会把外部尾延迟带入 S3 写路径，因此常采用解耦队列。

## 6. Audit Pipeline

审计与 notification 结构相似但语义不同。Audit pipeline 面向安全事件，必须保留请求主体和结果，并提供目标故障可见性。`registry` 管理目标，`factory` 从配置构造目标，`observability` 输出 pipeline 指标。

启动和关闭时要特别关注队列：启动是否恢复未投递事件，关闭是否有 bounded flush，满队列时是背压、丢弃还是降级，以及每种选择是否产生明确指标。

## 7. 容量、配额和负载准入

`rustfs/src/capacity/` 与 `crates/object-capacity` 聚合 pool、set、disk 和对象容量。`bucket/quota/checker.rs` 在对象操作前后检查桶配额。

容量值可能来自异步采样，因此只能作为准入近似，最终写入仍会遇到磁盘满。覆盖对象时净增量、multipart 临时空间、版本保留和并发写会让精确配额复杂化。

`workload_admission.rs` 和 `crates/concurrency::workload` 依据系统压力控制请求；`storage/concurrency` 管理 I/O permit；`data_movement/backpressure.rs` 限制后台迁移。它们共同目标是保护前台延迟和内存水位，而不是保证业务公平性的唯一来源。

## 8. Rebalance、缓存与维护任务

Rebalance 在池间迁移对象；object-data-cache 后台填充和淘汰缓存；table catalog maintenance 对 Iceberg 元数据执行规划、恢复和 worker 任务；`delete_tail_activity.rs`、`allocator_reclaim.rs`、`memory_observability.rs` 等处理局部维护。

所有后台循环都应回答同一组问题：谁启动、谁取消、状态是否持久化、重复执行是否安全、与前台 mutation 如何加锁、资源预算在哪里、失败是否可见、重启后如何恢复。

## 9. 可观测性

`crates/obs` 初始化 tracing、OpenTelemetry、metrics runtime 和清理器。`crates/io-metrics` 是指标定义和记录的主要事实来源，覆盖 I/O、S3、锁、backpressure、deadlock、internode、capacity、cache、timeout 和 allocator。

指标热路径应使用低分配的 label 和受控基数。bucket/object/request ID 不适合作为通用 metrics label，可放入 trace 或结构化日志。后台任务的关键指标包括队列深度、最老任务年龄、成功/失败/重试、处理延迟、扫描进度和取消状态。

