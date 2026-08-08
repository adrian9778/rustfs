# Storage API 与 ECStore 内核

## 1. 先读契约

`crates/storage-api` 是数据面的稳定边界。它把能力拆成多个 trait：

- `ObjectIO`：对象字节流读写。
- `ObjectOperations`：对象 stat、copy、delete 等操作。
- `ListOperations`：对象和版本列举。
- `MultipartOperations`：multipart 操作。
- `BucketOperations`：桶生命周期。
- `NamespaceLocking`：命名空间锁能力。
- `HealOperations`：修复能力。
- `StorageAdminApi`：磁盘、池和控制面能力。

同时定义 `BucketInfo`、`ListObjectsInfo`、`ListObjectVersionsInfo`、`PartInfo`、`MultipartInfo`、`HTTPRangeSpec`、`HTTPPreconditions`、`ObjectToDelete`、`DeletedObject` 等跨层数据类型。

契约层不应依赖具体 ECStore 目录布局。这样 S3 handler、后台任务、测试兼容层和其他协议可以面向同一语义编程。

## 2. RustFS 适配层

`rustfs/src/storage/ecfs.rs` 是应用层到 ECStore 的主要适配器；`ecfs_extend.rs` 扩展补充能力。`rustfs/src/storage/storage_api.rs` 和顶层 `rustfs/src/storage_api.rs` 提供兼容或聚合接口。

这一层还组合：

- `access.rs`：访问和权限相关桥接。
- `options.rs`：将应用选项转换为底层选项。
- `request_context.rs`：请求级上下文向存储传播。
- `timeout_wrapper.rs`：按操作类型和规模施加 deadline。
- `backpressure.rs`：字节水位与管道背压。
- `concurrency/`：I/O 调度、请求 guard 和并发管理。
- `lock_optimizer.rs`：锁路径优化。
- `head_prefix.rs`：HEAD/prefix 相关快速路径。
- `sse.rs`：服务端加密适配。

## 3. ECStore 顶层结构

`crates/ecstore` 是工程最大的 crate。可以按层理解：

```text
Object API / Bucket systems / Services
                |
        ErasureServerPools
                |
           ErasureSets
                |
       erasure coding + quorum
                |
       StorageAPI / disk abstraction
                |
     local disk or remote disk RPC
```

`store/init.rs` 和 `store/init_format.rs` 初始化对象层与磁盘格式；`core/pools.rs` 管理多个 server pool；`core/sets.rs` 管理一个 pool 内的 erasure sets。对象 key 会通过稳定分布算法映射到 set，不能随意改变，否则升级后对象会“消失”在新的映射位置。

## 4. Endpoint、format、pool 与 set

`layout/endpoint.rs`、`layout/endpoints.rs` 和 `disk/endpoint.rs` 解析本地/远端磁盘地址。`layout/format.rs`、`disk/format.rs` 和初始化模块处理磁盘 format。

format 保存集群身份、pool/set/disk 索引和格式版本。启动必须验证：

- 磁盘是否属于同一部署。
- set 数量和每 set 磁盘数是否一致。
- disk ID 是否重复或错位。
- 格式版本是否可读，是否需要迁移。
- 可用磁盘是否达到读/写 quorum。

格式修复不能凭“多数值”盲目覆盖少数磁盘；必须区分新盘、离线盘、损坏盘和来自其他集群的盘。

## 5. 磁盘抽象

`disk/disk_store.rs` 定义磁盘操作边界，`disk/local.rs` 实现本地磁盘，`cluster/rpc/remote_disk.rs` 将远端节点包装成同样的接口。`disk/fs.rs`、`disk/os.rs` 处理文件系统操作，`disk/health_state.rs` 跟踪磁盘健康。

错误在 `disk/error.rs`、`error_conv.rs`、`error_reduce.rs` 中被分类和归约。分布式调用不能只保留最后一个错误；需要根据错误类型和数量判断：成功、可容忍少数失败、read quorum 不足、write quorum 不足、对象不存在，还是磁盘离线。

## 6. 纠删码读写

`erasure/coding/` 是 Reed-Solomon/bitrot 相关实现：

- `encode.rs`：把数据切成 data shards 并生成 parity shards。
- `decode.rs`、`decode_reader.rs`：按 range 读取足够 shard 并恢复数据。
- `heal.rs`：利用健康 shard 重建缺失或损坏 shard。
- `bitrot.rs`：块级完整性校验。
- `erasure.rs`：参数、分片大小和公共编排。

写路径的核心语义是“临时写入 + quorum + 原子发布”：先向目标磁盘写临时数据和元数据，达到 write quorum 后以 rename/commit 发布；失败则清理或留下可被恢复流程识别的状态。数据先于元数据还是元数据先于数据、目录 fsync 是否完成，决定掉电后的可恢复性。

读路径根据元数据选择版本和数据目录，读取达到 data shard 数量的健康分片，校验 bitrot 后恢复目标 range。若某些 shard 损坏，读可以成功但应触发或记录 heal，而不能把损坏字节静默返回。

## 7. Quorum 与错误归约

读写 quorum 不是固定的“磁盘数除二”。它会受到 data/parity 配置、对象状态和特定操作影响。删除、元数据更新、rename、heal 等操作对一致性的要求也可能不同。

阅读 `error_reduce.rs` 或调用点时，逐项确认：

1. 哪些错误算作缺席，哪些算作磁盘故障。
2. 是否把同类错误按 variant 归并。
3. 成功数达到阈值后，少数失败如何记录和修复。
4. 多种错误同时出现时，返回给上层的是哪一个。
5. 返回 `NoSuchKey` 前是否已有足够磁盘确认不存在。

## 8. 命名空间锁与并发

`crates/lock` 提供 local、distributed、namespace 和 fast lock。对象操作通常按 bucket/object/version 或内部资源形成锁 key。`GlobalLockManager` 选择本地或分布式实现。

锁必须覆盖“检查条件到提交状态”的完整区间。例如条件写若先检查 ETag、释放锁、再写入，就会产生 TOCTOU。反过来，锁内等待网络或大块 I/O 又会放大尾延迟，因此代码常把预处理放在锁外，只把状态相关的最小事务放在锁内。

多资源操作必须保持全局锁顺序。取消和超时路径要释放 permit/lock；基于 RAII 的 guard 是主要保障，但 spawn 出去的任务和 channel 持有者仍需单独检查。

## 9. 节点间 RPC

`rustfs/src/storage/rpc/` 暴露节点服务，按 bucket、disk、event、heal、health、lock、metrics 分组。`crates/ecstore/src/cluster/rpc/` 提供 client、peer REST/S3 client、remote disk、remote locker、认证和 context propagation。

`crates/protos` 保存生成的协议类型和协议常量。协议版本、消息最大值、capability probe、canonical mutation body 都是混合版本集群的兼容边界。升级时不能假设所有节点同时具备新 RPC；应先探测 capability，再选择新路径或兼容路径。

内部 RPC 也必须认证。签名通常覆盖方法、路径、时间和 canonical body；重放窗口、节点身份、body hash 与上下文传播是重点审查项。

## 10. 桶级系统

`ecstore/src/bucket/` 将桶元数据缓存为运行时系统：metadata、policy、versioning、object lock、quota、tagging、target、lifecycle、replication 等。磁盘上的配置是事实来源，内存 sys/cache 是服务请求的快照。

配置更新通常需要：验证输入、持久化、更新本地缓存、通知 peer、处理部分节点失败。不能先更新内存再假定持久化必然成功；否则重启后状态会回退。

## 11. Rebalance 和跨池迁移

`services/rebalance/` 包含控制、元数据、迁移、worker 和 runtime。rebalance 需要在旧位置读取对象，在目标 pool 写入并确认，然后才清理旧数据。版本、删除标记、对象锁、tier 状态和并发覆盖都必须保留。

跨池操作的 fence/capability 用于避免两个节点对同一对象作出冲突迁移决定。失败恢复依赖持久化进度，而不是仅靠内存队列。

