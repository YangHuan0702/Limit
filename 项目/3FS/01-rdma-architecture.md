# RDMA 网络层完整设计

## 为什么 3FS 选择 RDMA

在分布式存储系统中，网络通常是最先遇到的瓶颈。普通 TCP/IP 栈有几个固有的开销：

- **内核拷贝**：数据在用户态和内核态之间复制（send/recv 都要经过）
- **中断处理**：每个数据包都要经过内核协议栈处理，产生大量中断
- **CPU 参与**：即使用 epoll 异步化，CPU 仍需介入每个 I/O 完成事件

RDMA（Remote Direct Memory Access）绕过这些开销：

- 网卡（RNIC）直接读写远端主机的用户态内存
- 数据传输不经过内核，也不需要对端 CPU 参与
- 延迟从 TCP 的微秒级降到纳秒级，吞吐从 10Gbps 提升到 100Gbps+

3FS 支持两种 RDMA 传输层：
- **InfiniBand**：专用硬件，延迟最低
- **RoCE（RDMA over Converged Ethernet）**：复用以太网基础设施，更易部署

## 3FS 的 RDMA 分层架构

```
┌─────────────────────────────────────────────────────┐
│                  应用层 RPC                          │
│            (serde::ClientContext)                    │
├─────────────────────────────────────────────────────┤
│                  IBSocket                            │
│   - QP 状态机（INIT→CONNECTING→READY→CLOSE）       │
│   - RDMA 发送/接收/读/写操作                        │
│   - 批量 WR 提交（RDMAReqBatch）                   │
├──────────────────┬──────────────────────────────────┤
│   RDMABuf        │   RDMARemoteBuf                   │
│   （本地内存）   │   （远端内存引用）                │
├──────────────────┴──────────────────────────────────┤
│                  IBDevice / IBPort                   │
│   - 设备枚举与拓扑感知                              │
│   - PD（Protection Domain）管理                     │
│   - CQ（Completion Queue）轮询                      │
├─────────────────────────────────────────────────────┤
│              ibverbs / libibverbs                    │
│              （用户态 RDMA 驱动 API）               │
└─────────────────────────────────────────────────────┘
```

## IBDevice：设备抽象层

### 核心职责

`IBDevice` 封装一块 InfiniBand/RoCE 网卡，管理：
- 设备打开与端口发现（`ibv_open_device`, `ibv_query_port`）
- 全局共享的 Protection Domain（`ibv_alloc_pd`）
- 设备级别的资源生命周期

### 关键设计：多设备支持

```cpp
static constexpr int kMaxDeviceCnt = 4;
```

3FS 最多支持 4 块 RDMA 网卡同时使用。`RDMABuf` 内部维护每个设备的 MR（Memory Region），允许同一块内存在多个设备上注册，从而实现多网卡负载均衡。

### 拓扑感知选路

IBDevice 通过 subnet 和 zone 配置实现拓扑感知：

```
zone A: [node1, node2, node3]  →  优先使用 device0
zone B: [node4, node5, node6]  →  优先使用 device1
```

同区域内通信选用低延迟路径，跨区域通信自动切换到对应网卡，避免不必要的 HOP。

## IBSocket：连接与传输层

### QP 状态机

Queue Pair（QP）是 RDMA 的通信端点，状态转换如下：

```
INIT → CONNECTING → ACCEPTED → READY
                              → ERROR
                 → CLOSE
```

- **INIT**：`ibv_create_qp` 创建 QP
- **CONNECTING**：调用 `qpInit()` 进入 INIT 态，`qpReadyToRecv()` 进入 RTR 态
- **ACCEPTED**：TCP 握手完成，交换 QP 信息（LID、QPN、PSN）
- **READY**：`qpReadyToSend()` 进入 RTS 态，可以发送数据
- **CLOSE/ERROR**：优雅关闭（Drainer 类）或错误处理

### 关键配置参数

```cpp
CONFIG_HOT_UPDATED_ITEM(max_sge, 16u);          // 每个 WR 的 scatter-gather 数量上限
CONFIG_HOT_UPDATED_ITEM(max_rdma_wr, 128u);     // 发送队列深度
CONFIG_HOT_UPDATED_ITEM(max_rdma_wr_per_post, 32u); // 每次 ibv_post_send 的批量大小
CONFIG_HOT_UPDATED_ITEM(max_rd_atomic, 16u);    // 并发 RDMA READ 数量上限
CONFIG_HOT_UPDATED_ITEM(buf_size, 16u * 1024);  // 每个 send/recv buffer 大小
CONFIG_HOT_UPDATED_ITEM(send_buf_cnt, 32u);     // send buffer 数量
```

这些参数的选取逻辑：

- `max_rdma_wr=128`：队列深度越大，流水线越满，但 QP 占用的硬件资源也越多
- `max_rdma_wr_per_post=32`：批量 post 减少 `ibv_post_send` 调用次数，降低软件开销
- `buf_size=16KB`：控制消息（RPC header、ACK）通常远小于 16KB，这个大小兼顾利用率和内存占用

### 发送/接收 Buffer 设计

`IBSocket` 内部维护两套 pre-posted buffer：

**SendBuffers（发送缓冲）**：
```
┌──────┬──────┬──────┬──────┐
│ buf0 │ buf1 │ buf2 │ ...  │   每个 buf = buf_size 字节
└──────┴──────┴──────┴──────┘
  tailIdx →          ← frontIdx
```
使用 cache-line 对齐的两个原子变量分离读写索引，避免 false sharing。

**RecvBuffers（接收缓冲）**：
使用 `folly::MPMCQueue` 存储已收到的 `(bufIdx, len)` 对，多消费者安全。

### RDMA 操作：批处理是关键

单次 RDMA 操作流程：

```cpp
// 构建 RDMAReqBatch
auto batch = socket.rdmaReadBatch();
batch.add(remoteBuf1, localBuf1);  // 添加第一个读请求
batch.add(remoteBuf2, localBuf2);  // 添加第二个读请求
co_await batch.post();             // 一次性提交
```

`RDMAReqBatch` 内部把所有请求编码为 `ibv_send_wr` 链表，一次 `ibv_post_send` 提交全部，而不是每个请求单独调用。

**WR 提交的实现细节**：

```cpp
// 每 max_rdma_wr_per_post 个 WR 提交一次
// 最后一个 WR 设置 IBV_SEND_SIGNALED，触发完成通知
// 中间的 WR 不设置 SIGNALED，减少 CQ 事件数量
```

这个优化把 CQ 事件从 N 减到 N/32，大幅降低轮询开销。

### WRId 编码：节省内存的 Tag 系统

```cpp
struct WRId : private folly::StampedPtr<void> {
    // 利用指针对齐的低位 bit 存储类型信息
    // 64 位 = 16 位 stamp（WRType） + 48 位 ptr（数据）
    
    static uint64_t send(uint32_t signalCount);  // SEND 类型
    static uint64_t recv(uint32_t bufIndex);      // RECV 类型
    static uint64_t rdma(RDMAPostCtx *ptr, bool last); // RDMA 类型
    static uint64_t ack();
    static uint64_t close();
};
```

`StampedPtr` 把 `type` 塞进了 64 位 WR ID 的高位，完全不需要额外的 map 查找。当 `ibv_poll_cq` 返回完成事件时，直接从 `wc.wr_id` 中解码出类型和上下文指针。

### ImmData：无额外 RTT 的控制信息传递

RDMA WRITE with IMMEDIATE 可以在数据传输完成时，额外携带 32 位立即数给对端，不需要对端主动 RECV。

3FS 利用这个机制传递 ACK 信息：

```cpp
class ImmData {
    // 高 8 位：消息类型（ACK 或 CLOSE）
    // 低 24 位：数值（ACK count）
    
    static ImmData ack(uint32_t count);   // 一次 ACK N 个 buffer
    static ImmData close();               // 通知对端关闭
};
```

对端收到 RDMA WRITE 完成后，从 `wc.imm_data` 解码，不需要单独发一个 send 来确认。

## RDMABuf：本地内存管理

### 核心原则：注册一次，使用多次

RDMA 操作要求内存必须预先向 RNIC 注册（`ibv_reg_mr`），注册操作很慢（需要 pin 住物理页、建立 IOMMU 映射）。错误的实现方式：

```cpp
// 错误：每次 I/O 都注册/注销，极慢
for (auto& req : requests) {
    auto mr = ibv_reg_mr(pd, buf, size, flags);
    do_rdma(mr);
    ibv_dereg_mr(mr);
}
```

3FS 的正确做法：预先分配并注册好内存池，所有 I/O 都从池中借用：

```cpp
// RDMABufPool：预注册的 buffer 池
auto pool = RDMABufPool::create(bufSize, bufCnt);

// 使用时从池中分配（阻塞等待直到有可用 buffer）
auto buf = co_await pool->allocate();
do_rdma(buf);
// buf 析构时自动归还给池
```

### Inner 类：多设备 MR 管理

```cpp
class RDMABuf::Inner {
    static constexpr int kAccessFlags =
        IBV_ACCESS_LOCAL_WRITE |   // 允许本地写（RDMA READ 需要）
        IBV_ACCESS_REMOTE_WRITE |  // 允许远端写
        IBV_ACCESS_REMOTE_READ |   // 允许远端读
        IBV_ACCESS_RELAXED_ORDERING; // 放松顺序约束，提升性能

    std::array<RDMABufMR, IBDevice::kMaxDeviceCnt> mrs_; // 每个设备一个 MR
};
```

`IBV_ACCESS_RELAXED_ORDERING` 是一个重要的性能优化：允许 RNIC 在不违反语义的前提下乱序完成操作，在现代多 PCIe 设备上能带来可观的吞吐提升。

### 用户 Buffer 注册

```cpp
// 允许用户提供自己的内存（零拷贝路径）
static RDMABuf createFromUserBuffer(uint8_t *buf, size_t len);
```

这个接口让应用可以把自己的内存注册为 RDMA buffer，从而实现真正的零拷贝：数据直接从应用内存通过网络传到存储节点，没有任何中间拷贝。

## RDMAControl：流控机制

```cpp
// 每个 IB 设备的并发传输限制
class RDMATransmissionLimiter {
    folly::fibers::Semaphore sem_; // 信号量控制并发数
    // 默认 max_concurrent_transmission = 64
};
```

**为什么需要流控**：RDMA 网卡的硬件资源是有限的（QP、CQE 数量），无限制地提交请求会导致硬件队列溢出，性能反而下降。信号量保证飞行中的 WR 数量不超过硬件能高效处理的上限。

## 实现时的关键决策

### 决策 1：何时用 SEND/RECV，何时用 RDMA READ/WRITE

| 操作 | 适用场景 |
|------|---------|
| SEND/RECV | 控制消息、RPC header、小数据（< 16KB） |
| RDMA WRITE | 数据从客户端推送到服务端（写路径） |
| RDMA READ | 数据从服务端拉取（读路径，服务端无感知） |

3FS 的读路径用 RDMA READ（拉取），写路径用 RDMA WRITE（推送）。服务端读不需要服务端 CPU 介入，极大减少服务端负载。

### 决策 2：CQ 轮询策略

```
                  ┌─────────────────┐
                  │  epoll / eventfd │  ← 有事件时才唤醒，省 CPU
                  └────────┬────────┘
                           │
              ┌────────────▼────────────┐
              │     ibv_poll_cq         │  ← 批量轮询，减少系统调用
              │   (每次最多 N 个 WC)    │
              └─────────────────────────┘
```

不要用忙等（spin poll）CQ，除非延迟极度敏感且 CPU 资源充裕。3FS 默认使用事件驱动模式（`ibv_get_cq_event`），在 epoll 事件循环中处理完成事件。

### 决策 3：连接建立用 TCP out-of-band

RDMA 连接建立本身不能用 RDMA（鸡生蛋问题）。3FS 用 TCP socket 交换：
- 本地 QPN（Queue Pair Number）
- 本地 LID（InfiniBand）或 GID（RoCE）
- 初始 PSN（Packet Sequence Number）

完成握手后，TCP socket 的职责结束，所有数据传输走 RDMA。

## 常见陷阱

### 陷阱 1：MR 注册的锁竞争

`ibv_reg_mr` 在某些驱动实现中有全局锁，高并发注册时会成为瓶颈。解决方案：
- 在系统启动时一次性注册好所有需要的 buffer
- 使用池化管理，避免运行时注册/注销

### 陷阱 2：忘记 post recv 导致 QP 进入错误状态

RDMA 的 RECV 必须提前 post 好，否则对端发送时本端 QP 会进入 IBV_QPS_ERR 状态。3FS 在连接建立后立刻 post 所有 recv buffer，并在每次消耗一个 recv buffer 后立即补充。

### 陷阱 3：忽略 RoCE 的特殊配置

RoCE 运行在以太网上，需要：
- **PFC（Priority Flow Control）**：防止丢包，否则 RDMA 性能极差
- **ECN（Explicit Congestion Notification）**：拥塞控制
- 正确配置 `traffic_class` 和 `gid_index`

3FS 在 `IBSocket::Config` 中专门为 RoCE 提供了 `roce_pkey_index` 和 `traffic_class` 配置项。
