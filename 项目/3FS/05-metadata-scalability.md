# 元数据水平扩展：无状态 meta + FoundationDB

## 元数据为什么容易成为瓶颈

传统分布式文件系统（如早期 HDFS）用单节点 NameNode 管理元数据。这导致：
- 单节点内存限制了命名空间规模
- 单节点 CPU 限制了元数据操作 QPS
- 单节点故障导致整个文件系统不可用

3FS 的设计目标是：元数据层可以水平扩展，添加 meta 节点就能线性增加吞吐。

## 3FS 的元数据架构

```
                     ┌──────────────────────┐
                     │    FoundationDB      │
                     │  (分布式 KV 存储)    │
                     │  - 分布式事务（SSI）  │
                     │  - 自动分片           │
                     │  - 高可用            │
                     └──────────┬───────────┘
                                │
         ┌──────────────────────┼──────────────────────┐
         │                      │                      │
  ┌──────▼──────┐       ┌───────▼─────┐       ┌──────▼──────┐
  │  meta-1     │       │   meta-2    │       │   meta-3    │
  │ (无状态)    │       │  (无状态)   │       │  (无状态)   │
  └─────────────┘       └─────────────┘       └─────────────┘
         │                      │                      │
         └──────────────────────┴──────────────────────┘
                                │
                         客户端（随机选择）
```

**关键设计**：meta 节点本身不持久化任何状态，所有状态都在 FoundationDB 中。增减 meta 节点只需修改服务发现配置，不影响数据完整性。

## FoundationDB：为什么选它

FoundationDB 提供了分布式系统最难实现的部分：

- **Serializable Snapshot Isolation（SSI）**：最强的隔离级别，防止写偏斜
- **全局事务**：跨多个 key 的原子操作
- **自动分片**：命名空间随数据增长自动扩展
- **故障自愈**：节点失效后自动重新复制

对 meta 层来说，只需要把文件系统操作翻译成 FoundationDB 事务，不需要自己实现分布式一致性。

### FoundationDB 事务示例

```
创建文件 /dir/file.txt 对应的 FDB 事务：

BEGIN TRANSACTION
  // 检查父目录存在
  GET dirInode = fdb.get(key("/dir"))
  ASSERT dirInode.exists()
  
  // 检查文件不存在
  GET existing = fdb.get(key("/dir/file.txt"))
  ASSERT !existing.exists()
  
  // 分配 inode ID
  inode_id = ATOMIC_INCREMENT(inode_counter_key)
  
  // 写入 inode
  SET fdb.set(key(inode_id), serialize(InodeAttr{mode, uid, gid, ...}))
  
  // 写入目录项
  SET fdb.set(key("/dir/file.txt"), serialize(inode_id))
COMMIT
```

如果两个客户端同时创建同名文件，SSI 保证其中一个事务失败（检测到冲突），绝不会静默地丢掉其中一个。

## Distributor：请求路由

客户端不是随机找 meta 节点，而是基于 inode ID 哈希进行路由：

```cpp
class Distributor {
    // 根据 inode ID 选择处理该请求的 meta 节点
    MetaServer* select(InodeId id) {
        return metaServers[id % metaServers.size()];
    }
};
```

**为什么要基于 inode 路由而不是随机**：
- 同一个 inode 的请求路由到同一个 meta 节点，利用节点的本地缓存
- 减少不同 meta 节点同时操作同一 FDB key 的概率（减少事务冲突）

## ChainAllocator：数据布局决策

创建新文件时，meta 需要决定文件的数据放在哪些 chain 上：

```cpp
class ChainAllocator {
    Layout allocate(FileConfig config) {
        auto chains = getAvailableChains();
        
        // 使用 MT19937 确定性随机（避免热点聚集）
        shuffle(chains.begin(), chains.end(), mt19937(seed));
        
        // Round-robin 选取 stripeCount 个 chain
        Layout layout;
        for (int i = 0; i < config.stripeCount; i++) {
            layout.addChain(chains[(offset + i) % chains.size()]);
        }
        
        // 递增全局分配计数器
        offset = (offset + config.stripeSize) % chains.size();
        return layout;
    }
};
```

**为什么用 Round-Robin + Shuffle**：
- 纯 Round-Robin：所有文件的第一个 chunk 都在同一批节点，产生热点
- 纯随机：不同文件可能集中在少数节点，分布不均
- Round-Robin + 确定性 Shuffle：均匀分布，同时保证可重现

### 动态 Stripe 扩展

文件创建时的 stripe 数可能不足（不知道文件最终大小）。3FS 支持运行时扩展：

```
初始：stripeCount=1  →  chain[A]
写入后发现超过 64MB，扩展为 stripeCount=4  →  chain[A, B, C, D]
```

扩展操作通过 FDB 事务原子完成，客户端通过 layout 的 `chainVer` 字段感知变化。

## 会话管理与租约

### 为什么需要会话

客户端打开文件后，服务端需要跟踪这个"打开"状态：
- 文件锁（write lock 互斥）
- 缓冲写的刷新时机
- 客户端崩溃时的资源清理

```cpp
struct Session {
    ClientUuid   clientId;     // 唯一标识客户端
    SessionId    sessionId;    // 会话 ID
    InodeId      inodeId;      // 关联的文件
    Timestamp    lastActive;   // 最后活跃时间
    SessionFlags flags;        // 读写模式等
};
```

### 租约过期与清理

客户端定期续约（heartbeat），服务端检测到租约过期：

```
MetaClient.CloserConfig:
    prune_session_batch_interval = 10s   // 每 10 秒扫描一次
    prune_session_batch_count = 128      // 每批最多清理 128 个会话
```

批量清理（128 个/批）而非逐个删除，减少 FDB 事务开销。

## 元数据操作的性能优化

### Versionstamp：因果一致性

FDB 的 Versionstamp 是一个单调递增的全局时间戳。客户端记录上次看到的 versionstamp，后续请求携带这个 stamp：

```
client: read(inode, afterVs=1234)
meta: // 确保 FDB 事务 readVersion >= 1234
      // 保证读到不早于上次写的状态
```

这实现了"read-your-writes"语义，避免客户端读到自己刚刚写入的数据的旧版本。

### 无状态的代价：每次操作都要访问 FDB

meta 节点本身不缓存元数据，每个操作都读写 FDB。FDB 的事务延迟通常在 1~5ms。

对于高频访问的目录（如模型权重目录），这会成为瓶颈。3FS 的部分缓解策略：
- 客户端侧缓存 inode（带 TTL）
- 批量 stat 操作合并成单个 FDB 事务
- 使用 FDB 的 snapshot read 降低冲突概率

## 与其他方案的对比

| 方案 | 代表系统 | 扩展性 | 一致性 | 实现复杂度 |
|------|---------|--------|--------|-----------|
| 单节点 | HDFS NameNode | 差（内存上限） | 强 | 低 |
| 分布式锁服务 | HDFS HA + ZK | 中（HA 不等于扩展） | 强 | 中 |
| 分片元数据 | CephFS MDS | 好 | 强（但有复杂性） | 高 |
| 无状态 + 外部 KV | 3FS + FDB | 很好 | 很强（SSI） | 中（外包给 FDB） |

3FS 的做法本质是把"分布式元数据"这个难题外包给了 FoundationDB，meta 层只做业务逻辑翻译。代价是对 FDB 的运维依赖。
