# 测试版图与源码调试方法

## 1. 测试层级

RustFS 的测试大致分四层：

1. 模块内单元测试：验证 parser、数据结构、状态机和边界值。
2. crate 集成测试：验证公开 API、持久化兼容和多个模块组合。
3. `crates/e2e_test`：启动真实服务，使用 S3/Admin/协议客户端验证行为。
4. 故障与兼容测试：注入磁盘、网络、节点、旧格式和 MinIO fixture。

对存储行为，只有单元测试通常不够。一次正确的改动至少需要覆盖协议结果和持久化/故障语义中的相关部分。

## 2. E2E 测试地图

`crates/e2e_test/src` 的文件名就是需求索引：

- 鉴权：anonymous、admin auth、bucket policy、negative SigV4、presigned、STS、IAM filter。
- 对象：GET/HEAD/range、copy metadata/tagging/checksum、特殊字符、leading slash、archive。
- 版本：delete marker、version ID、list versions、copy version restore。
- Multipart：权限、storage class、checksum、SSE/KMS。
- 一致性：conditional writes、namespace lock quorum、overwrite cleanup、degraded read EOF。
- 集群：multidrive pool、distributed startup、node interaction、disk fault、heal rebuild。
- 后台：notification、lifecycle、tier transition、replication、data usage、stale multipart cleanup。
- 协议：SFTP、FTP(S)、WebDAV、Swift。
- 安全：security boundary、KMS authorization negative matrix、TLS reload。

查找回归测试时，先按用户可观察行为搜文件名，再搜目标 handler/usecase。不要仅以函数名搜索，因为端到端测试往往通过 HTTP 调用而不引用内部函数。

## 3. 从一个 S3 失败开始定位

建议按以下顺序：

```text
确认 method/path/query/header/body
  -> 找到 s3_api handler
  -> 确认认证类型和 policy action
  -> 查看 usecase 形成的 options
  -> 找 storage-api trait 方法
  -> 找 EcStore 实现和 quorum 分支
  -> 查看 FileInfo/xl.meta 与 reader pipeline
  -> 对照 S3 error 映射和 E2E 测试
```

若响应码正确但 body/headers 不兼容，问题通常在 handler 的协议映射；若单节点正常而少盘失败，重点看错误归约/quorum；若重启后才出现，重点看 commit、rename、fsync 和 metadata decode；若大对象或 range 才出现，重点看 rio wrapper、块边界、压缩/加密 index。

## 4. 启动失败定位

先确定最后到达的 `SystemStage`，再映射到 `startup_*.rs`。磁盘 format 问题进入 `startup_storage` 和 ECStore init；IAM/桶配置问题在对象层之后；监听和证书问题在 server/TLS 阶段；后台服务失败要判断是阻止 readiness 还是降级运行。

`rustfs diagnose` 使用 `log-analyzer` 做离线聚合。该 crate 设计为同步、无 Tokio、无内部 RustFS 依赖，并把 parse failure 当作数据统计，因此适合处理部分损坏或混合格式日志。

## 5. 分布式问题定位

对每个节点记录：部署/节点 ID、pool/set/disk 身份、协议 capability、readiness、请求 ID 和时间。重点检查：

- 请求在哪个节点进入，哪个节点拥有目标磁盘。
- local disk 与 remote disk 返回是否不同。
- quorum 统计是否把 timeout/offline/not-found 混为同类。
- peer 是否处于不同版本，是否走了 fallback。
- leader epoch/session sequence 是否拒绝了陈旧任务。
- cancellation 是否只在入口触发而未传到远端调用。

网络重试必须结合幂等性看。GET 和 capability probe 通常易重试；mutation 若没有 request ID、事务 token 或状态探测，盲目重试可能重复提交。

## 6. 元数据与对象损坏定位

保留原始 `xl.meta` 和相关 shard，不要先修复。使用 dump 工具查看每块盘上的版本列表，比较 version ID、mod time、data dir、erasure distribution 和 checksum。

常见模式：

- 部分盘缺少版本：提交/rename 部分失败或磁盘离线。
- 同一版本 data dir 不同：并发写、恢复或旧提交残留。
- metadata 存在但数据不足：写入顺序、清理或磁盘损坏。
- 数据可恢复但 bitrot 失败：个别 shard 损坏，应读取并 heal。
- 删除标记选择错误：版本排序或 null version 语义问题。
- tier 指针存在但远端失败：transition transaction 或 remote version 参数问题。

## 7. 并发与挂起定位

从 request guard、workload admission、I/O permit、namespace lock、pipe/channel、远端 RPC 依次检查等待点。一个请求可能尚未开始磁盘 I/O，却已经在队列或锁上等待。

重点确认：permit 是否由 RAII guard 释放，锁顺序是否一致，channel 是否仍有隐藏 sender，任务是否在持锁时 await 外部 I/O，progress 是否真实更新，timeout 是排队时限还是整个操作时限。

`concurrency/deadlock` 提供策略类型，实际 hang/deadlock 监视位于 `rustfs/src/storage/deadlock_detector.rs`。指标位于 `io-metrics` 的 queue、lock、timeout 和 deadlock 模块。

## 8. 性能阅读法

GET/PUT 热路径优先看：body 是否被完整缓冲、reader 是否多层装箱、是否重复 clone metadata、锁是否覆盖 I/O、CPU 密集纠删码是否占用 async worker、range 是否读取了过多 shard、日志/metrics label 是否产生高基数。

后台性能优先看：扫描批量、sleep/budget、队列边界、target 重连、rebalance/tier 并发和 data-usage snapshot 大小。性能优化不能破坏 backpressure、checksum、durability gate 或取消语义。

## 9. 高价值检索命令

```bash
# 从路由找 handler
rg -n 'route|Router|Method::' rustfs/src/server rustfs/src/storage/s3_api rustfs/src/admin

# 从 trait 找实现
rg -n 'impl .*ObjectIO|impl .*ObjectOperations|impl .*MultipartOperations' rustfs crates/ecstore

# 查错误归约与 quorum
rg -n 'quorum|reduce_err|read_quorum|write_quorum' crates/ecstore/src

# 查一个元数据字段的写入和读取
rg -n 'x-rustfs-internal|x-minio-internal' rustfs/src crates

# 查后台任务的启动、取消和 join
rg -n 'spawn|CancellationToken|cancelled|JoinHandle' rustfs/src/startup_* crates/scanner crates/replication crates/notify

# 查已有回归测试
rg -n '<行为关键词>' crates/e2e_test/src crates/*/tests rustfs/src
```

## 10. 修改前的最小阅读闭环

一个可靠的源码修改应至少读完：入口 handler 或任务入口、对应 usecase、storage/domain trait、具体实现、错误映射、已有测试和最近的同类边界处理。涉及持久化或 RPC 时，还要读 writer 与 reader 两端；涉及动态配置时，要读启动加载、运行时发布和 Admin 更新三条路径。

这套闭环能避免最常见的局部修复：只改写端不改读端、只改本地不改 remote、只改新协议不保留 fallback、只改 happy path 不改 quorum、只改请求路径不处理后台重放。

