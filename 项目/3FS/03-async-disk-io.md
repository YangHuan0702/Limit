# 异步磁盘 I/O：libaio、io_uring 与工作队列

## 磁盘 I/O 为什么必须异步

存储服务的性能上限由两个因素决定：
1. **网络带宽**：RDMA 网卡的吞吐（100Gbps 以上）
2. **磁盘带宽**：NVMe SSD 的顺序读吞吐（单盘 7GB/s，多盘叠加）

要让多块 NVMe 盘同时跑满，必须有足够深的 I/O 队列。如果用同步 `pread`，一个线程同时只能有一个未完成的 I/O，即使有 100 个线程也只有 100 个并发 I/O。而 NVMe 盘的最优队列深度通常在 32~128。

异步 I/O（`libaio`/`io_uring`）让单个线程可以同时提交数百个 I/O 请求，等待完成事件时不占用 CPU。

## AioReadWorker：读 I/O 调度器

### 整体设计

```
                 ┌──────────────────────────────┐
  batchRead()    │         AioReadWorker         │
  ─────────────▶ │                              │
                 │  ┌────────┐  ┌────────┐     │
                 │  │ queue  │  │ thread │ ×32 │
                 │  │(4096)  │  │  pool  │     │
                 │  └───┬────┘  └────┬───┘     │
                 └──────│────────────│──────────┘
                         │           │
              ┌──────────▼──┐    ┌───▼──────────────┐
              │  BoundedQueue │    │  io_uring / libaio│
              │  (job 缓冲)  │    │  (内核 I/O 引擎)  │
              └─────────────┘    └──────────────────┘
```

**关键配置**：
```cpp
struct Config {
    num_threads = 32;     // I/O 工作线程数
    queue_size = 4096;    // job 队列容量
    max_events = 512;     // 每次 poll 最多处理的完成事件数
    enable_io_uring = true; // 优先使用 io_uring
    min_complete = 128;   // io_uring 最小等待完成数
};
```

### I/O 引擎选择

```cpp
enum class IoEngine {
    libaio,   // Linux AIO，成熟稳定
    io_uring, // 更新，开销更低
    random,   // A/B 测试：随机选择引擎
};
```

**libaio vs io_uring**：

| 特性 | libaio | io_uring |
|------|--------|----------|
| 提交接口 | `io_submit` | `io_uring_enter` 或 sqpoll |
| 完成接口 | `io_getevents` | `io_uring_enter` |
| 系统调用开销 | 每批 1~2 次 | 可以降到 0（SQPOLL 模式） |
| 内存拷贝 | 提交时拷贝 iocb | 共享内存 ring，零拷贝 |
| 内核支持要求 | ≥ 2.6.22 | ≥ 5.1 |

3FS 在新内核上默认启用 io_uring，旧内核回退到 libaio。`random` 模式用于生产环境的在线 A/B 测试，在不停机的情况下验证引擎切换效果。

## UpdateWorker：写 I/O 调度器

### 核心设计：per-disk 队列

```
                        UpdateWorker
                    ┌───────────────────┐
  write(diskId=2)   │  queueVec_[0]  ──┼── disk0 专用线程池
  ─────────────────▶│  queueVec_[1]  ──┼── disk1 专用线程池
                    │  queueVec_[2]  ──┼── disk2 专用线程池  ← 路由到这里
                    │  queueVec_[3]  ──┼── disk3 专用线程池
                    └───────────────────┘
```

**per-disk 队列解决的问题**：
- 防止一块慢盘拖慢整个系统（队列隔离）
- 保证单盘内写操作有序（避免随机写放大）
- 便于按盘统计 I/O 延迟（精准定位问题盘）

**配置参数**：
```cpp
struct Config {
    queue_size = 4096;   // 每个 disk 队列的容量
    num_threads = 32;    // 主 I/O 线程（写操作）
    bg_num_threads = 8;  // 后台线程（chunk 回收、清理）
};
```

主线程池和后台线程池分离，避免 GC 操作延迟正常写 I/O。

## BatchReadJob：批量 I/O 的组织结构

### 数据结构层次

```
BatchReadJob                          // 一个 batchRead RPC 对应的所有 I/O
├── AioReadJob[0]                     // 对应一个 chunk 的读取
│   ├── State
│   │   ├── localbuf: RDMABuf        // 目标内存（RDMA 注册）
│   │   ├── chunkEngineJob           // chunk 元数据和文件描述符
│   │   ├── headLength: size_t       // O_DIRECT 对齐 padding（头）
│   │   ├── tailLength: size_t       // O_DIRECT 对齐 padding（尾）
│   │   ├── readLength: size_t       // 实际读取长度（含 padding）
│   │   ├── readFd: int              // chunk 文件描述符
│   │   ├── readOffset: size_t       // 磁盘读取偏移（对齐后）
│   │   └── chunkChecksum: uint32_t  // 预期 CRC32C
│   └── readUncommitted: bool        // 是否允许读未提交数据
├── AioReadJob[1]
│   └── ...
└── finishedCount: atomic<int>        // 完成计数，用于通知
    baton: folly::coro::Baton         // 协程等待点
```

### 对齐处理的细节

O_DIRECT 要求读取偏移和长度都按 block size（通常 512B）对齐：

```
用户请求：offset=1000, length=5000

                   ↓ 对齐到 512B 边界
实际读取：offset=512, length=5120  (headLength=488, tailLength=632)

内存布局：
┌───────┬──────────────────────┬──────────┐
│ head  │   实际数据（5000B）  │  tail    │
│(488B) │                      │ (632B)  │
└───────┴──────────────────────┴──────────┘
  ^                              ^
  读到但不属于用户请求的部分，需要被丢弃
```

返回给用户时只返回 `[headLength, headLength + 用户请求长度)` 这段。

## Checksum 验证：数据完整性保障

### 为什么在读路径做 checksum

NVMe SSD 有内部 ECC，但这只能检测到存储介质的位翻转，不能检测：
- 软件 bug 导致的写错位置
- 固件 bug 导致的数据静默损坏
- RDMA 传输过程中的数据损坏

3FS 在写入时计算 CRC32C 并与数据一起持久化，读取时重新计算并比对：

```cpp
// 读取完成后
uint32_t actualChecksum = crc32c(data, length);
if (actualChecksum != expectedChecksum) {
    return makeError(StatusCode::kChecksumError);
}
```

### CRC32C 的性能

现代 CPU 有硬件 CRC32C 指令（`crc32q` on x86_64）。3FS 在 CMakeLists.txt 中开启了 `SSE4.2`（包含 `POPCNT` 和 `CRC32` 指令），利用硬件加速把 checksum 计算开销降到可忽略的水平。

## 协程与 I/O 的集成

3FS 使用 `folly::coro`（C++20 协程）把异步 I/O 包装成同步语义的代码：

```cpp
// 看起来像同步代码，实际上是异步的
CoTask<void> handleRead(BatchReadJob& job) {
    // 提交所有 AIO 请求
    for (auto& aioJob : job) {
        submitAio(aioJob);
    }
    
    // 在这里"等待"，但线程不阻塞
    co_await job.baton;
    
    // 所有 I/O 完成，继续处理
    verifyChecksums(job);
    sendRDMAResponse(job);
}
```

`folly::coro::Baton` 在 AIO 完成回调中调用 `baton.post()`，协程在下一个调度点恢复执行。

这个模型让一个线程可以同时"等待"数百个 I/O 完成，而不需要为每个 I/O 分配一个线程。

## 线程池策略

### CPUThreadPoolExecutor 的使用

3FS 使用 `folly::CPUThreadPoolExecutor` 而非 `std::thread` 的原因：
- 内置工作队列，避免手写 bounded queue
- 支持线程数动态调整（hot update）
- 内置指标收集（延迟、队列长度、线程利用率）

### 线程数的选取原则

```
读线程数 = min(32, NVMe 盘数 × 最优队列深度 / 单线程并发)
写线程数 = min(32, NVMe 盘数)  // 写路径通常 CPU 更密集
```

实际调优需要用 `iostat -x` 观察每块盘的 `await` 和 `util` 指标：
- `util < 50%`：线程数可以减少
- `await` 突增：可能队列过深，适当减少并发
- 所有盘 `util ≈ 100%`：真正跑满了，横向加盘

## io_uring SQPOLL 模式（进阶）

标准 io_uring 每次提交仍需一次 `io_uring_enter` 系统调用。SQPOLL 模式启动一个内核线程持续轮询 SQ ring，完全消除提交侧的系统调用：

```cpp
struct io_uring_params params = {};
params.flags |= IORING_SETUP_SQPOLL;
params.sq_thread_idle = 2000; // 2ms 空闲后内核线程休眠
io_uring_setup(queueDepth, &params);
```

**适用场景**：I/O 请求非常密集（每秒数十万次）的高性能存储服务。

**代价**：内核线程持续占用一个 CPU 核，轻负载下是浪费。3FS 默认不启用，但接口设计上预留了支持。
