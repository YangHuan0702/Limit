# 性能调优全景：关键参数、瓶颈定位、指标体系

## 调优的前提：先能测量

任何调优都应该从建立基线指标开始，而不是凭感觉改参数。3FS 提供了内置的 monitor 子系统（`monitor_collector`），收集以下维度：

```
┌──────────────────────────────────────────────────────────┐
│                    指标体系                               │
├───────────────┬───────────────┬──────────────────────────┤
│  RDMA 层      │  磁盘 I/O 层  │  元数据层                │
│  - 发送字节   │  - 读 IOPS    │  - 操作延迟              │
│  - 接收字节   │  - 写 IOPS    │  - FDB 事务延迟          │
│  - WR 队列深度│  - 读带宽     │  - 会话数                │
│  - CQ 事件数  │  - 写带宽     │  - 请求 QPS              │
│  - 连接数     │  - await 延迟 │                          │
│  - per-peer   │  - util       │                          │
│    延迟       │              │                           │
└───────────────┴───────────────┴──────────────────────────┘
```

## 关键参数速查表

### RDMA 层参数

| 参数 | 位置 | 默认值 | 调优方向 |
|------|------|--------|---------|
| `max_rdma_wr` | IBSocket::Config | 128 | 增大提升流水线深度，但消耗 QP 资源 |
| `max_rdma_wr_per_post` | IBSocket::Config | 32 | 增大减少 ibv_post_send 调用次数 |
| `max_sge` | IBSocket::Config | 16 | 按实际 scatter-gather 需求调整 |
| `send_buf_cnt` | IBSocket::Config | 32 | 增大提升并发，按内存配额限制 |
| `max_concurrent_transmission` | RDMAControl | 64 | 单设备飞行中 WR 上限，超过反而下降 |
| `record_latency_per_peer` | IBSocket::Config | false | 开启调试 per-peer 延迟问题 |

### 磁盘 I/O 层参数

| 参数 | 位置 | 默认值 | 调优方向 |
|------|------|--------|---------|
| `num_threads`（读）| AioReadWorker::Config | 32 | 按 NVMe 盘数 × 最优队列深度调整 |
| `max_events` | AioReadWorker::Config | 512 | 每轮 poll 批量，增大减少轮询开销 |
| `min_complete` | AioReadWorker::Config | 128 | io_uring 最小等待完成数，影响吞吐/延迟 |
| `enable_io_uring` | AioReadWorker::Config | true | 新内核保持 true，旧内核设为 false |
| `num_threads`（写）| UpdateWorker::Config | 32 | 写密集时增大，注意 per-disk 队列 |
| `bg_num_threads` | UpdateWorker::Config | 8 | 回收/清理后台线程数 |

### 存储客户端参数

| 参数 | 位置 | 默认值 | 调优方向 |
|------|------|--------|---------|
| `max_batch_size` | TrafficControlConfig | 128 | 增大提升批处理效率 |
| `max_batch_bytes` | TrafficControlConfig | 4MB | 按网络 MTU 和 chunk 大小调整 |
| `max_concurrent_requests` | TrafficControlConfig | 32 | 系统级并发上限 |
| `max_concurrent_requests_per_server` | TrafficControlConfig | 8 | 单 storage 并发上限 |

### FUSE/客户端参数

| 参数 | 位置 | 默认值 | 调优方向 |
|------|------|--------|---------|
| `io_depth`（IoRing）| IoRing 参数 | 32 | 大文件顺序读增大到 128+ |
| `timeout_us`（IoRing）| IoRing 参数 | 1000 | 低延迟场景减小，吞吐场景增大 |
| `prune_session_batch_interval` | MetaClient | 10s | 高会话数量时缩短清理间隔 |
| `check_server_interval` | MetaClient | 5s | 不稳定环境缩短以快速发现故障 |

## 性能剖析：逐层找瓶颈

### 第一步：确认瓶颈在哪一层

```bash
# 1. 检查网络带宽是否跑满
ibv_devinfo -d mlx5_0  # 确认设备和端口状态
perfquery -x            # 查看端口带宽计数器

# 2. 检查磁盘 I/O 是否跑满
iostat -x 1 | grep -E "(Device|nvme)"
# 看 util% 是否接近 100，以及 await 是否异常

# 3. 检查 CPU 是否成为瓶颈
perf top -p <storage_pid>  # 找 CPU 热点函数
```

### 第二步：RDMA 层诊断

**症状：带宽低，但 CPU 不高**

可能原因：
1. WR 队列深度不足（`max_rdma_wr` 太小）
2. 批处理效率差（`max_rdma_wr_per_post` 太小）
3. per-peer 流控限制（`max_concurrent_transmission` 不够）

诊断方法：
```
// 开启 per-peer 统计
record_latency_per_peer = true

// 观察 CQ 事件频率（理想：低频，每次批量处理多个）
// 如果 CQ 事件频率 ≈ I/O 请求频率，说明没有批处理
```

**症状：延迟高，RDMAPostCtx::waitLatency 大**

原因：`rdmaSem_` 信号量等待时间长，说明 WR 队列满了。

解决：增大 `max_rdma_wr` 或减少并发请求数。

### 第三步：磁盘 I/O 层诊断

**症状：吞吐低于磁盘理论值**

诊断：
```bash
# 检查每块盘的 I/O 深度
iostat -x 1 | awk '/nvme/{print $1, $7, $14}'
# avgrq-sz（平均请求大小）和 avgqu-sz（平均队列深度）

# 理想情况：avgqu-sz ≥ 8，avgrq-sz 尽量大
```

常见问题：
- `avgqu-sz < 4`：提交深度不够，增大 `num_threads` 或使用 io_uring SQPOLL
- `avgrq-sz < 32KB`：请求太小，检查 batch 策略

**症状：写延迟 p99 很高（写抖动）**

原因：GC/回收操作打断写路径。

解决：
- 确保 `bg_num_threads` 足够，回收任务不排队到主线程
- 检查 chunk 回收频率，考虑更大的 chunk size 减少 GC 频率

### 第四步：元数据层诊断

**症状：open/stat QPS 低**

诊断：
```
// 检查 FDB 事务延迟
// 正常：P50 < 2ms，P99 < 10ms
// 异常：P99 > 50ms，看 FDB 监控是否有热点 key

// 检查 meta 节点 CPU 利用率
// 如果 CPU 不高但延迟高，瓶颈在 FDB
// 如果 CPU 高，考虑增加 meta 节点
```

**症状：rename/create 操作频繁失败**

原因：FDB 事务冲突（Serializable 隔离级别下正常）。

解决：
- 检查应用是否大量并发操作同一目录
- 增大 FDB 的 `transaction_retry_limit`
- 考虑用更分散的目录结构（避免单目录热点）

## 生产环境调优案例

### 案例 1：大模型 Checkpoint 加载慢

**场景**：加载 200GB Checkpoint，速度只有 4GB/s，RDMA 网卡是 200Gbps

**诊断过程**：
1. `ibv_devinfo` 确认网卡状态正常
2. `iostat` 发现磁盘 util 只有 30%
3. 检查 `max_concurrent_requests_per_server = 8`，太低

**解决**：
```
max_concurrent_requests_per_server: 8 → 32
max_batch_size: 128 → 256
io_depth（IoRing）: 32 → 128
```

加载速度提升到 18GB/s（接近多盘聚合带宽上限）。

### 案例 2：训练数据读取不稳定（P99 高）

**场景**：P50 延迟 < 5ms，P99 延迟 > 500ms

**诊断过程**：
1. 开启 `record_latency_per_peer = true`，发现某个 storage 节点延迟异常
2. `iostat` 发现该节点有一块 NVMe 盘 await 偶发性飙升
3. 检查该盘的 `UpdateWorker` 队列，发现 GC 任务和写任务共享队列

**解决**：
- 增大 `bg_num_threads`，GC 任务不占用写线程
- 对问题盘做 SMART 检测（发现是盘出现早期故障，需要替换）

### 案例 3：元数据 QPS 不够，影响训练 data loader

**场景**：PyTorch DataLoader 开 32 个 worker，每个 worker 高频 `open/read/close`

**诊断过程**：
1. meta 节点 CPU 不高（<30%）
2. FDB 监控发现单个热点 key（数据集根目录的 inode）

**解决**：
- 在 DataLoader 层缓存目录列表（减少 `readdir` 频率）
- 把数据集拆分到多个子目录（分散 FDB key）
- 增加 meta 节点数量（从 2 增到 6）

## 端到端延迟分解

理解端到端延迟的组成，才能精准定位问题：

```
客户端发出读请求
  │
  ├── 网络传输（RDMA SEND）          ：0.5~2μs
  │
  ▼
storage 节点收到请求
  │
  ├── 请求解析与 chunk 定位          ：1~5μs
  │
  ├── ChunkEngine 查询元数据         ：1~10μs（内存操作）
  │
  ├── AIO 磁盘读取（NVMe）           ：50~500μs（主要瓶颈）
  │
  ├── Checksum 验证（CRC32C HW加速）：1~5μs
  │
  ├── RDMA WRITE 到客户端           ：1~10μs
  │
  ▼
客户端收到数据（总计：55~530μs）
```

NVMe 磁盘延迟（50~500μs）是主要来源。要降低 P99，关键是：
1. 避免热点盘（数据分布均匀）
2. 确保 GC 不阻塞读路径
3. 使用多盘并行 I/O（通过 stripe）

## 容量规划参考

### 存储节点

```
每节点配置：
- 8 × NVMe SSD（每盘 7GB/s 顺序读）= 56GB/s 磁盘带宽
- 2 × 100Gbps RDMA 网卡 = 25GB/s 实际传输带宽（含协议开销）
- 网卡是瓶颈（25 < 56）

调整：
- 使用 4 × 25GbE 网卡 = 12.5GB/s（磁盘反而过剩）
- 使用 1 × 400GbE 网卡 = 50GB/s（磁盘成为瓶颈）
- 理想匹配：2 × 100GbE = 25GB/s，8 盘 × 3GB/s per-disk util = 24GB/s ✓
```

### 副本数量

```
3 副本（默认）：
- 存储空间利用率 = 1/3
- 读吞吐 = 3 × 单节点吞吐（任意副本可读）
- 写吞吐 = 1/3 × 单节点写带宽（需要同时写 3 份）
- 推荐：读多写少的训练场景

2 副本（可选）：
- 存储空间利用率 = 1/2
- 读吞吐 = 2 × 单节点吞吐
- 适合对成本敏感但还需要容错的场景
```
