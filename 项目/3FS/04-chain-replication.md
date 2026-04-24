# CRAQ 链式复制：强一致性与高吞吐并存

## 为什么选 Chain Replication 而不是 Raft

分布式存储系统的复制方案主要有两类：

| 方案 | 代表实现 | 优点 | 缺点 |
|------|---------|------|------|
| 基于日志的共识 | Raft, Paxos | 动态成员变更灵活 | 写路径经过 Leader，吞吐受限 |
| Chain Replication | CRAQ, CR | 读/写吞吐高，实现简单 | 节点变更需要外部协调 |

3FS 选择了 Chain Replication（带 CRAQ 扩展）。在 AI 训练场景下，读吞吐是最关键的指标（Checkpoint 加载、数据集读取），CRAQ 的 read-any 特性让每个副本都能承担读流量，吞吐随副本数线性扩展。

## Chain Replication 基础

### 写路径：head-to-tail 串行传播

```
Client ──write──▶ Head ──forward──▶ Middle ──forward──▶ Tail
                                                          │
Client ◀──ack───────────────────────────────────────────┘
```

- 写请求只发给链头（Head）
- Head 存储后转发给 Middle
- Middle 存储后转发给 Tail
- **只有 Tail 确认后，Head 才返回给客户端**

这保证了：在客户端收到 ack 时，数据已经在所有节点上持久化。

### 读路径（原始 CR）：只读 Tail

原始 Chain Replication 只允许从 Tail 读，保证线性一致性（linearizability）。但这样：
- Tail 成为读瓶颈
- 其他副本的带宽被浪费

### CRAQ 扩展：read-any

CRAQ（Chain Replication with Apportioned Queries）允许从任意节点读：

```
客户端读请求 → 任意选择链上的一个节点
    │
    ├─ 如果本节点的该 chunk 处于 COMMIT 状态
    │   → 直接返回数据（高速路径）
    │
    └─ 如果本节点的该 chunk 处于 CLEAN 状态（有飞行中的写）
        → 向 Tail 查询最新 commitVer
        → 如果本地版本 == Tail commitVer → 直接返回
        → 否则返回 Tail 上的数据（安全降级）
```

**实践中**：大多数读请求命中 COMMIT 状态（写完成后 chunk 快速进入 COMMIT），read-any 基本等于无额外开销的线性扩展。

## 3FS 的版本号系统

```cpp
struct ChunkMetadata {
    uint64_t commitVer;   // 最后一次提交成功的版本
    uint64_t updateVer;   // 最后一次发起的更新版本
    uint64_t chainVer;    // 链成员变更的版本（用于检测 stale 操作）
};
```

**版本号的语义**：

| 状态 | commitVer | updateVer | 含义 |
|------|-----------|-----------|------|
| 已提交 | V | V | 数据完整，可以直接读 |
| 写进行中 | V | V+1 | 有未完成的写，读需谨慎 |
| 链切换中 | - | - | chainVer 不匹配，拒绝操作 |

### commitVer 和 updateVer 如何工作

写请求流程（幂等实现）：

```
1. 客户端发送 write(chunkId, data, requestId, updateVer=V+1)

2. Head 接收：
   - 检查 chainVer 是否匹配（防止 stale 请求）
   - 写入数据，设置 chunk.updateVer = V+1（仍为 CLEAN）
   - 转发给 Middle

3. Middle 同样操作，转发给 Tail

4. Tail 接收并存储：
   - 调用 ChunkEngine.commit(chunkId, V+1)
   - commit 设置 chunk.commitVer = V+1（进入 COMMIT）
   - 返回 ack 给客户端

5. Head 收到 ack：
   - 调用 ChunkEngine.commit(chunkId, V+1)（更新自己的 commitVer）
   - 返回 ack 给客户端
```

**幂等性保证**：如果写请求重试（相同 requestId 和 updateVer），每个节点检查当前版本，如果已经等于 updateVer 则直接返回成功。

## 集群管理器的角色

Chain Replication 本身不包含节点失效处理，这部分由集群管理器（mgmtd）负责：

```
mgmtd 维护的 Chain Table（权威路由表）：
┌─────────┬─────────────────────────────────────┐
│ chainId │ nodes (按顺序)                        │
├─────────┼─────────────────────────────────────┤
│  1001   │ [node-a:HEAD, node-b:MID, node-c:TAIL] │
│  1002   │ [node-d:HEAD, node-e:TAIL]              │
│  1003   │ [node-f:HEAD, node-g:MID, node-h:TAIL] │
└─────────┴─────────────────────────────────────┘
```

**节点失效处理流程**：

```
1. node-b 心跳超时（lease 过期）
2. mgmtd 将 chainId 1001 更新为：
   [node-a:HEAD, node-c:TAIL]（跳过 node-b）
3. chainVer 递增（1001.chainVer++）
4. 通过 listener 通知所有客户端刷新路由
5. 客户端后续写请求携带新的 chainVer
6. node-a 和 node-c 上飞行中的 updateVer > commitVer 的 chunk
   需要通过 recovery 流程对齐
```

## 写路径的流水线优化

简单实现中，Head 等 Tail 的 ack 后才回复客户端，写延迟 = 链长度 × 单节点延迟。

3FS 通过**请求流水线**减少等待：

```
时间轴：
         Head           Middle          Tail
t=0  ← recv req1 →  forward(req1)
t=1  ← recv req2 →  forward(req2)  ← recv req1 → forward(req1)
t=2  ← recv req3 →  forward(req3)  ← recv req2 → forward(req2)  ← recv req1 → ack
t=3               ←───────────────────────────────────────── ack(req1)
```

多个写请求在链上同时流动，链长度带来的延迟摊销到流水线中。稳定状态下，吞吐约等于单节点吞吐，延迟约等于链长度 × 单跳延迟。

## ReliableForwarding：可靠转发机制

```cpp
// 存储服务的写路径实现
CoTask<void> StorageOperator::write(WriteRequest req) {
    // 1. 本地写入
    co_await chunkEngine.update(req.chunkId, req.data, req.updateVer);
    
    // 2. 转发给下一个节点（如果自己不是 Tail）
    if (!isTail(req.chainId)) {
        auto nextNode = getNextNode(req.chainId);
        co_await reliableForward(nextNode, req);  // 带重试的转发
    }
    
    // 3. 如果是 Tail，提交并 ack
    if (isTail(req.chainId)) {
        co_await chunkEngine.commit(req.chunkId, req.updateVer);
        co_await sendAck(req.clientAddr);
    }
}
```

`reliableForward` 包含重试逻辑：如果下一个节点暂时不可达（网络抖动），会使用指数退避重试，而不是立刻报错。

## 读路径的优化：read uncommitted

在某些场景（如 ML 训练中间 Checkpoint 的快速加载），允许读取尚未在所有副本提交的数据：

```cpp
struct ReadOptions {
    bool readUncommitted = false;  // 如果为 true，跳过版本检查
};
```

`readUncommitted` 模式下：
- 不检查 commitVer
- 直接返回本地磁盘上的最新数据（可能包含未提交的写）
- 性能更高，一致性降低（适合幂等的读场景）

## 与 Raft 方案的实际对比

在 3FS 的目标场景（大文件顺序读为主的 AI 工作负载）下：

**Chain Replication 的优势**：
- 读吞吐 = 副本数 × 单节点吞吐（线性扩展）
- 实现复杂度低（没有选主、没有日志回放）
- 写路径 CPU 开销低（无需维护 Raft 日志）

**Chain Replication 的劣势**：
- 节点增减需要外部协调（mgmtd）
- 链的最大长度受延迟影响
- 不支持任意节点失效后自动选主

**结论**：对于 3FS 的场景，Chain Replication 是正确的选择。如果你的场景需要频繁节点变更或对写延迟极度敏感，考虑 Multi-Paxos/Raft。
