# RustFS 源码阅读手册

本目录是一套面向源码维护者的 RustFS 阅读笔记。它不只罗列文件，而是回答四类问题：一次请求从哪里进入、状态由谁持有、数据如何落盘、故障在哪一层被处理。

## 阅读入口

| 分卷 | 主题 | 建议读者 |
|---|---|---|
| [sourceReader.md](sourceReader.md) | 工程总览、主链路与首次阅读地图 | 第一次接触工程的人 |
| [01-bootstrap-runtime.md](01-bootstrap-runtime.md) | 进程启动、配置、运行时装配、关闭 | 维护启动和部署逻辑的人 |
| [02-http-s3-application.md](02-http-s3-application.md) | HTTP 栈、S3 路由、应用用例与请求上下文 | 维护 S3 API 的人 |
| [03-storage-ecstore.md](03-storage-ecstore.md) | 存储契约、ECStore、纠删码、磁盘与集群 RPC | 维护数据面的工程师 |
| [04-metadata-io.md](04-metadata-io.md) | `xl.meta`、FileInfo、流式读写、校验和与加密 | 排查对象损坏和兼容问题的人 |
| [05-security-admin.md](05-security-admin.md) | 鉴权、IAM、策略、KMS、Admin API 与 TLS | 安全和控制面维护者 |
| [06-background-services.md](06-background-services.md) | 生命周期、复制、扫描、修复、通知、审计、容量 | 维护后台任务的人 |
| [07-workspace-crates.md](07-workspace-crates.md) | workspace 全部 crate 的职责与依赖位置 | 查找代码归属的人 |
| [08-testing-debugging.md](08-testing-debugging.md) | 测试版图、故障定位、阅读与调试方法 | 开发、测试和运维人员 |

## 一张图理解工程

```text
客户端
  -> TCP/TLS 与 HTTP server
  -> 中间件：请求标识、可信代理、限流、审计、压缩、指标
  -> S3/Admin/健康检查/内部 RPC 路由
  -> 鉴权与策略判定
  -> app usecase：编排对象、桶、分片上传等业务语义
  -> storage-api：稳定的存储能力契约
  -> EcStore/ErasureServerPools：池、set、磁盘、锁与 quorum
  -> filemeta + rio/io-core：元数据和字节流
  -> 本地磁盘或远端节点
```

后台任务与请求链路共享同一个对象层，但通常从 scanner、lifecycle、replication、heal、notification 等调度器进入。它们最需要关注取消传播、幂等性、锁顺序、版本语义和部分失败。

## 阅读原则

1. 先找 trait，再找实现。`storage-api` 给出“系统承诺什么”，`ecstore` 才回答“如何做到”。
2. 先看正常路径，再看错误归约。分布式存储最重要的行为常藏在 quorum、错误分类和回滚分支中。
3. 区分控制面和数据面。Admin/IAM/配置主要改变状态，S3/ECStore 主要搬运数据；两者在启动期和后台服务中交汇。
4. 把版本、删除标记、空 version ID、nil UUID、远端 tier 元数据视为独立状态，不要按普通可选值理解。
5. 看到全局句柄时追踪其发布时机。RustFS 的许多模块可被构造，不代表系统已经 ready；readiness 阶段才是对外可用性的依据。

