# 自己实现 3FS：高性能与 RDMA 专题指南

本系列文档聚焦于 3FS 中最具技术深度的部分：**高性能 I/O 路径**、**RDMA 网络层**、以及其他在生产级分布式文件系统中才有的工程特性。

与 `docs/自己实现3FS/` 系列（讲"先做什么、后做什么"）不同，本系列假设你已经能跑通基本的读写流程，现在要把系统的吞吐和延迟推到极限。

## 面向读者

- 具备 Linux 系统编程经验（epoll、io_uring、mmap）
- 理解 RDMA 基础概念（QP、MR、WR、CQ）或愿意深入学习
- 正在实现或优化一个分布式存储系统的数据路径

## 文档列表

| 文件 | 主题 |
|------|------|
| [01-rdma-architecture.md](./01-rdma-architecture.md) | RDMA 网络层完整设计：IBDevice、IBSocket、RDMABuf |
| [02-zero-copy-io.md](./02-zero-copy-io.md) | 零拷贝 I/O：内存注册、用户态 buffer、USRBIO |
| [03-async-disk-io.md](./03-async-disk-io.md) | 异步磁盘 I/O：libaio、io_uring、UpdateWorker |
| [04-chain-replication.md](./04-chain-replication.md) | CRAQ 链式复制：强一致性与高吞吐并存 |
| [05-metadata-scalability.md](./05-metadata-scalability.md) | 元数据水平扩展：无状态 meta + FoundationDB |
| [06-client-io-ring.md](./06-client-io-ring.md) | 客户端 I/O Ring：用户态提交队列与批处理 |
| [07-performance-tuning.md](./07-performance-tuning.md) | 性能调优全景：关键参数、瓶颈定位、指标体系 |

## 阅读建议

1. 先读 [01-rdma-architecture.md](./01-rdma-architecture.md)，建立 RDMA 编程模型的整体认知。
2. 再读 [02-zero-copy-io.md](./02-zero-copy-io.md) 和 [03-async-disk-io.md](./03-async-disk-io.md)，这是数据路径的两端。
3. [04-chain-replication.md](./04-chain-replication.md) 讲清楚 3FS 为什么能在强一致性下保持高吞吐。
4. [05-metadata-scalability.md](./05-metadata-scalability.md) 解释为什么元数据不会成为瓶颈。
5. [06-client-io-ring.md](./06-client-io-ring.md) 是 3FS 最独特的功能之一，FUSE 之外的高性能接入方式。
6. 最后用 [07-performance-tuning.md](./07-performance-tuning.md) 做全局优化和指标建立。
