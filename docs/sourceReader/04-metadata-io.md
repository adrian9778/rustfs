# 元数据、流式 I/O、校验和与加密

## 1. FileInfo 是对象层的共同语言

`crates/filemeta` 描述对象元数据。`FileInfo`/`FileMeta` 及其版本、part、erasure 信息连接 S3 语义和磁盘格式：对象名、版本 ID、mod time、size、ETag、content type、用户 metadata、删除标记、data dir、纠删码布局、校验信息、transition 状态等最终都要在这里表达。

不要把 `FileInfo` 当普通 DTO。它同时承载：

- 对外可见的对象属性。
- 内部定位数据 shard 的信息。
- 版本和删除语义。
- heal、replication、lifecycle 的决策输入。
- 与 MinIO `xl.meta` 的兼容格式。

## 2. `xl.meta` 的角色

每个对象的 `xl.meta` 保存元数据和版本列表；对象数据可能 inline 在元数据内，也可能由 data directory 中的 shard 文件承载。版本条目可表示普通对象、删除标记或过渡状态。

读取流程大致是：读取 bytes，识别格式版本，MessagePack 解码，迁移旧字段，验证关键不变量，选择目标 version，再构造 `FileInfo`。任何持久化字节都不应因为“由本程序写入”而跳过校验：掉电、位腐烂、旧版本程序和部分写都可能制造异常值。

调试真实文件可使用：

```bash
cargo run -p rustfs-filemeta --example dump_fileinfo -- "/path/to/file/xl.meta"
```

重点观察版本 ID、data dir UUID、erasure data/parity、block size、distribution、parts、checksum、metadata 和 transition 字段。

## 3. 兼容性不变量

内部对象 metadata 同时存在 RustFS 和 MinIO 前缀。写入内部字段时，应通过 `crates/utils/src/http/metadata_compat.rs` 同时写 `x-rustfs-internal-*` 与 `x-minio-internal-*`；读取时 helper 优先 RustFS key，再兼容 MinIO key。

二进制 UUID 要按“缺失、空 bytes、nil UUID 都表示无值”处理。nil UUID 不能被当作合法 data dir 或 remote version。

远端 tier version 的 `None` 或空字符串表示远端桶未启用版本控制，发起 GET/DELETE 时不能携带 `versionId=`。空参数和无参数对对象存储服务可能具有不同语义。

scanner data-usage cache 使用手写 map 编码，以允许新增字段被旧 reader 忽略。若改成结构体默认的数组编码，字段追加会使旧版本无法解码整个缓存。

## 4. IO Core

`crates/io-core` 提供底层异步 I/O 原语和 writer 状态机。它关注有限缓冲、进度报告、取消、错误传播与 backpressure，而不理解 S3 对象语义。

`OperationProgress` 被并发控制层复用，用于判断一个操作仍在推进还是已经挂起。进度信号不能在没有实际字节推进时刷新，否则 deadlock/hang detector 会被虚假心跳蒙蔽。

## 5. RIO 管道

`crates/rio` 是对象流适配层：

- `limit_reader` / `hardlimit_reader`：限制可读长度，并区分到达限制与底层提前 EOF。
- `hash_reader`：边读边计算 hash/checksum。
- `etag_reader`：计算或解析 ETag。
- `encrypt_reader`：按块加解密。
- `compress_reader`、`compress_index`：压缩流与随机读取索引。
- `http_reader`：HTTP body 到内部 reader 的适配。
- `reader.rs`、`writer.rs`：统一动态流接口。

wrapper 顺序会改变结果。上传路径一般先限制原始请求长度并校验客户端 checksum，再根据配置压缩/加密，最后进入纠删码；下载路径按相反方向恢复，并对外部 range 和内部加密块边界做转换。

`rio-v2` 选择性替换压缩、加密和 S2 index，同时重导出 legacy API。它是一条渐进迁移边界：新实现要保持旧调用者的 trait 和边界语义，而不只是返回相同 happy-path 字节。

## 6. Checksum 与 ETag

`crates/checksums` 支持 CRC32、CRC32C、CRC64NVME、SHA1、SHA256、SHA512、XXHash 和 MD5 等算法，并提供 HTTP 编解码。S3 checksum 可能来自 header、trailer、multipart 聚合或服务端计算。

ETag 不总是内容 MD5：multipart、SSE、压缩或特定写入路径都可能产生不同语义。因此业务代码应使用已有 ETag/checksum 类型和 helper，而不是自行比较字符串。

校验需要区分三层：传输 payload hash、S3 checksum、磁盘 bitrot checksum。三者保护的字节表示和攻击/故障模型不同，不能互相替代。

## 7. 压缩、加密和 range

压缩后物理偏移不等于对象逻辑偏移，需要 index 将 GET range 映射到压缩块。加密按固定块添加认证开销，逻辑 range 也要扩展到完整加密块，解密后再裁剪。

组合对象可能是：逻辑明文 -> 压缩 -> 加密 -> EC shard。读取则是 shard 恢复 -> 解密 -> 解压 -> range 裁剪。元数据必须记录每层参数，否则无法随机读取或兼容旧对象。

SSE-S3/SSE-KMS 的数据密钥由服务端管理；SSE-C 的密钥来自请求且绝不能持久化或记录。元数据只保存加密后的数据密钥和算法材料。CopyObject 同时存在 source 解密和 destination 加密上下文，不能混用请求头。

## 8. Inline 数据与小对象

小对象可 inline 到 metadata，省去额外文件和 I/O。快速路径必须保持与普通对象相同的 checksum、版本、加密、复制、heal 和删除语义。inline 阈值或编码变化还涉及混合版本 reader，因此不能只按性能优化处理。

## 9. EOF、长度与取消

提前 EOF 往往表示请求体不足、磁盘 shard 截断或远端节点中断，不能当作正常结束。超出声明长度则可能是协议错误或走私风险。reader wrapper 应保留两者区别，并将错误映射到正确的 S3/存储错误。

取消应沿 HTTP body、transform pipeline、erasure worker 和磁盘 I/O 传播。channel 的发送端/接收端如果被额外 clone 持有，会导致 EOF 永远不出现，是排查流式请求挂起时的重要线索。

