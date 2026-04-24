# 客户端 I/O Ring：用户态提交队列与批处理

## FUSE 的天花板

FUSE（Filesystem in Userspace）让 3FS 能以普通文件系统的方式挂载，兼容所有使用 POSIX 文件 API 的应用。但 FUSE 有固有的性能限制：

```
应用                     内核                   FUSE daemon
  │                       │                         │
  ├──read(fd, buf, n)──▶  │                         │
  │                       ├──/dev/fuse 请求──────▶  │
  │                       │                         ├── 执行 RDMA 读
  │                       │ ◀──写回 /dev/fuse────── │
  │ ◀──数据复制到 buf──── │                         │
```

**每次 read 的开销**：
1. 系统调用（用户→内核）
2. `/dev/fuse` 队列操作（内核→用户态 daemon）
3. 数据拷贝（daemon 缓冲 → 应用缓冲）
4. 系统调用返回（内核→用户）

对于大量小 I/O（如 tokenizer 读取样本），这些开销叠加起来是显著的。

## IoRing：用户态 I/O 队列

### 核心思想

复刻 `io_uring` 的设计哲学，在用户态实现一个提交/完成队列，让应用直接与 3FS native client 通信，绕过内核 FUSE 路径：

```
应用进程                              3FS Native Client
   │                                       │
   │   ┌──────────────────────────────┐   │
   │   │    共享内存（mmap）           │   │
   │   │  ┌──────────┬────────────┐  │   │
   │   │  │  SqRing  │  CqRing   │  │   │
   │   │  │ (提交队列)│ (完成队列) │  │   │
   │   │  └──────────┴────────────┘  │   │
   │   │  ┌────────────────────────┐ │   │
   │   │  │   IoBuffer (数据区)    │ │   │
   │   │  └────────────────────────┘ │   │
   │   └──────────────────────────────┘   │
   │                                       │
   ├──sqe(read, fd=3, off=0, len=4MB)──▶  │
   │                                       ├── RDMA 读到 IoBuffer
   │                              cqe ◀── │
   │   读取 IoBuffer 中的数据             │
```

**关键优化**：
- 应用和 native client 共享同一块物理内存（`IoBuffer`）
- 数据通过 RDMA 直接写入 `IoBuffer`，应用无需额外拷贝
- 多个 sqe 可以被自动批量处理，一次 RPC 携带多个读请求

## IoRing 数据结构详解

### IovTable：数据 Buffer 管理

```
IovTable 管理多个 Iov（I/O Vector）：

Iov[0]: 2GB 共享内存块
  ┌────────────────────────────────────────────────┐
  │ chunk0(64MB) │ chunk1(64MB) │ chunk2(64MB) │...│
  └────────────────────────────────────────────────┘
  
Iov[1]: 另一个 2GB 共享内存块
  ...
```

Iov 用 `mmap(MAP_SHARED | MAP_ANONYMOUS)` 分配，native client 和应用进程通过相同的虚拟地址访问同一块物理内存。

### SqeSection：提交队列

```cpp
struct Sqe {
    uint64_t   userData;  // 应用自定义，原样返回到 cqe
    int32_t    fd;        // 文件描述符（注册到 IoRing 的）
    uint64_t   offset;    // 文件偏移
    uint32_t   length;    // 读取长度
    uint32_t   bufIndex;  // IoBuffer 中的目标位置
    uint32_t   flags;     // 操作标志（如 READ_UNCOMMITTED）
};
```

SqeSection 是一个环形数组，应用写入 `tail`，native client 读取 `head`，使用原子操作保证无锁并发。

### CqeSection：完成队列

```cpp
struct Cqe {
    uint64_t   userData;  // 从 sqe 原样传回
    int32_t    result;    // 0 = 成功，< 0 = 错误码
    uint32_t   bytesRead; // 实际读取字节数
};
```

Native client 写入 `tail`，应用读取 `head`，同样使用无锁环形队列。

### ringSection：同步与配置

```cpp
struct RingArgs {
    uint32_t   sqHead;    // 原子变量，SQ 消费位置
    uint32_t   sqTail;    // 原子变量，SQ 生产位置
    uint32_t   cqHead;    // 原子变量，CQ 消费位置
    uint32_t   cqTail;    // 原子变量，CQ 生产位置
    uint32_t   iodepth;   // 最大飞行中的 I/O 数
    uint32_t   timeout_us; // 批处理超时（微秒）
    // 同步原语（信号量）
    sem_t      sqSem;     // 应用通知 client 有新 sqe
    sem_t      cqSem;     // client 通知应用有新 cqe
};
```

## 批处理策略：核心性能杠杆

### 为什么批处理是关键

假设应用需要读取 1000 个 4MB chunk（4GB 总数据）：

**不批处理（1 个 sqe = 1 个 RPC）**：
- 1000 次 RPC 往返
- 每次 RPC 有控制消息开销
- 网络利用率低（大量小包）

**批处理（32 个 sqe = 1 个 RPC）**：
- 32 次 RPC 往返
- 控制消息开销均摊
- 数据传输使用大包，网络效率高

### 双触发策略

```
IoRingWorker 的批处理策略：

                    ┌─ 积累 iodepth 个 sqe ─▶ 立刻发送
  新 sqe 到来 ─────┤
                    └─ 未满但超过 timeout ──▶ 超时发送
                    
  timeout = ringArgs.timeout_us（默认 1ms）
  iodepth = ringArgs.iodepth（默认 32）
```

**timeout 的设置哲学**：

- AI 训练 prefetch 场景：增大 iodepth（128+），timeout 可以放宽到 5ms
- 交互式查询场景：减小 iodepth（4~8），timeout 设到 0.5ms 以内
- 混合场景：通过多个 IoRing 实例，不同优先级使用不同参数

### 优先级感知调度

`IoRingTable` 管理多个 IoRing 实例，按优先级调度：

```cpp
class IoRingTable {
    // 高优先级 ring（交互请求，小 iodepth）
    std::vector<IoRing> highPriorityRings;
    // 低优先级 ring（训练 prefetch，大 iodepth）
    std::vector<IoRing> lowPriorityRings;
    
    // 调度：优先处理高优先级 ring 中的请求
    CoTask<void> worker() {
        while (true) {
            auto reqs = collectRequests();
            co_await sendBatch(reqs);
        }
    }
};
```

## 文件描述符注册

应用不能直接把路径字符串传给 IoRing（开销大，需要路径解析）。必须先把文件描述符注册到 IoRing：

```
1. fd = open("/mnt/3fs/data/file.bin", O_RDONLY)
   （走正常 FUSE open 路径，meta 解析 inode）
   
2. ioring.register(fd)
   （Native client 记录 fd → inode 的映射）
   
3. sqe = {fd=fd, offset=0, length=4MB, bufIndex=0}
   ioring.submit(sqe)
   （走高性能路径，跳过 meta 查询）
```

fd 注册只需要一次，后续所有 I/O 直接走 I/O ring，不再经过 FUSE 或 meta 查询。

## FuseClients 的写路径优化

### 写缓冲与 DynamicAttr

```cpp
struct RcInode {
    struct DynamicAttr {
        bool     written;       // 有未刷新的写
        bool     synced;        // 已同步到 storage
        bool     fsynced;       // 已 fsync
        Uid      writer;        // 当前写入者的 UUID
        uint32_t dynStripe;     // 动态 stripe 数
        uint64_t truncateVer;   // truncate 版本
        uint64_t hintLength;    // 预期文件长度（用于预分配 layout）
        Timestamp atime, mtime;
    };
};
```

应用 `write()` 时：
1. 数据发送给 storage（通过 RDMA）
2. `DynamicAttr.written = true`（标记有写）
3. inode 加入 `dirtyInodes` 集合
4. 立刻返回给应用（不等待 storage ack）

这是写回缓存（write-back）语义，延迟写 ack 换取更低的写响应时间。

### 周期性刷新

```cpp
// periodicSyncRunner 每隔一段时间扫描 dirtyInodes
CoTask<void> FuseClients::periodicSyncScan() {
    for (auto& [inodeId, rcInode] : dirtyInodes) {
        if (rcInode->attr.written && !rcInode->attr.synced) {
            co_await periodicSync(rcInode);
        }
    }
}
```

`periodicSync` 向 meta 更新文件大小（如果有写入使文件变大），保证文件属性最终一致。

## USRBIO API 实战

### 应用侧使用方式

```cpp
// 1. 创建 IoBuffer（共享内存数据区）
auto iov = hf3fs::Iov::create(2_GB);

// 2. 创建 IoRing（提交/完成队列）
auto ior = hf3fs::IoRing::create(/*iodepth=*/32, /*timeout=*/1_ms);
ior.bindIov(iov);

// 3. 注册文件描述符
int fd = open("/mnt/3fs/model.bin", O_RDONLY);
ior.register(fd);

// 4. 批量提交读请求
for (int i = 0; i < 32; i++) {
    hf3fs::Sqe sqe;
    sqe.fd = fd;
    sqe.offset = i * 64_MB;
    sqe.length = 64_MB;
    sqe.bufIndex = i;
    ior.submit(sqe);
}

// 5. 等待完成
for (int i = 0; i < 32; i++) {
    auto cqe = ior.waitCompletion();
    // 数据已经在 iov[cqe.bufIndex] 中，直接使用
    process(iov.buf(cqe.bufIndex), cqe.bytesRead);
}
```

## 与 FUSE 路径的性能对比

| 指标 | FUSE 路径 | USRBIO 路径 |
|------|----------|------------|
| 系统调用次数 | 每次 I/O 2次 | 几乎 0 |
| 内存拷贝次数 | 1~2 次 | 0 |
| 批处理支持 | 受 FUSE 队列限制 | 原生支持 |
| 延迟（单次）| 50~200μs | 10~30μs |
| 吞吐（顺序大文件）| 受 FUSE 线程限制 | 受网络/磁盘限制 |

**什么时候用 FUSE**：
- 兼容性优先（现有应用无需修改）
- 随机小 I/O（USRBIO 收益不明显）
- 开发调试阶段

**什么时候用 USRBIO**：
- 大文件顺序读（模型权重加载）
- 高吞吐数据集 prefetch
- 延迟敏感的推理服务
