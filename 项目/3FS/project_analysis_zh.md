# 3FS 项目分析文档

## 1. 项目定位

3FS，全称 Fire-Flyer File System，是一个面向 AI 训练与推理场景的高性能分布式文件系统。项目的设计目标不是做通用对象存储，而是给训练数据读取、数据预处理、中间结果落盘、checkpoint 和推理 KVCache 这类高吞吐、强一致、共享命名空间的场景提供统一存储层。

从仓库内容和源码结构来看，3FS 的核心特征有三点：

1. 元数据与数据面彻底分离。
2. 元数据走事务型 KV 存储，数据块走基于 RDMA 的复制链。
3. 对外同时提供 FUSE 文件系统接口和更高性能的原生零拷贝 I/O 接口。

项目的主实现语言是 C++20，辅以 Rust 和 Python：

- C++ 是主控制面、元数据服务、存储服务、网络栈、客户端和 CLI 的主体实现。
- Rust 主要用于新的 `chunk_engine` 和一个 `trash_cleaner` 工具。
- Python 主要用于 FUSE/USRBIO 绑定、运维脚本和部署辅助工具。

## 2. 整体架构

根据 [README.md](../README.md) 和 [docs/design_notes.md](./design_notes.md) 的描述，以及 `src/mgmtd`、`src/meta`、`src/storage`、`src/fuse` 的入口代码，3FS 的运行时可以概括为五类核心角色：

### 2.1 `mgmtd`：集群管理面

`mgmtd_main` 是集群管理服务的入口。它负责：

- 维护集群成员关系和节点状态。
- 管理 chain、chain table、routing info 等全局路由信息。
- 接收 meta、storage、client 的心跳和租约续期。
- 负责配置模板的分发，以及部分动态配置更新。
- 在多副本部署时进行主备角色切换。

从 [src/mgmtd/MgmtdServer.h](../src/mgmtd/MgmtdServer.h) 可以看到，`mgmtd` 继承自通用 `net::Server`，默认暴露两组服务：

- RDMA 组，监听 `8000`，承载 `Mgmtd` 服务。
- TCP 组，监听 `9000`，承载 `Core` 服务。

`MgmtdOperator` 本身方法定义依赖 `fbs/mgmtd/MgmtdServiceDef.h` 展开的 RPC 接口，后台逻辑在 `src/mgmtd/background/`，包括：

- 心跳检查
- 链路更新
- 路由版本递增
- 客户端会话检查
- target 信息加载与持久化
- 指标更新

### 2.2 `meta`：元数据服务

`meta_main` 是元数据服务入口，服务主体在 `src/meta/service/MetaServer.h` 与 `src/meta/service/MetaOperator.h`。

它的主要职责是：

- 实现文件系统语义，包括 `open/create/mkdirs/list/rename/remove/truncate/sync/hardLink/symlink` 等。
- 维护 inode、目录项、会话、权限和布局信息。
- 从 FoundationDB 或 Hybrid KV 引擎中读写元数据。
- 在创建文件时分配 chain 和布局信息。
- 在删除、回收或迁移时协同存储层完成块清理。

`MetaOperator` 的接口直接暴露了它承担的文件系统元操作。它围绕以下几个组件组织：

- `MetaStore`：元数据读写与事务封装。
- `InodeIdAllocator`：inode ID 分配。
- `ChainAllocator`：文件数据布局与 chain 分配。
- `SessionManager`：文件会话管理。
- `GcManager`：垃圾回收相关逻辑。
- `FileHelper`、`Distributor`：元数据辅助逻辑。
- `UserStoreEx`：用户与鉴权相关能力。

设计上，meta 服务是无状态的。状态不保存在本机磁盘，而是保存在后端事务型 KV 中，因此它可以更容易地扩容、故障切换和滚动升级。

### 2.3 `storage`：数据存储服务

`storage_main` 是存储服务入口，服务主体是 [src/storage/service/StorageServer.h](../src/storage/service/StorageServer.h) 和 [src/storage/service/Components.h](../src/storage/service/Components.h)。

存储服务是 3FS 的数据面，主要职责包括：

- 按 chunk 粒度存取文件数据。
- 执行 CRAQ 风格的链式复制读写。
- 管理本地磁盘上的 chunk 元数据与物理空间。
- 处理数据恢复、同步、校验、异步读、批量更新等后台任务。

`Components` 说明了 storage 服务内部的真实组成，主要模块有：

- `StorageTargets`：管理本地 target 与磁盘目标。
- `StorageOperator`：处理核心读写 RPC。
- `ReliableForwarding`：负责可靠转发。
- `ReliableUpdate`：处理更新确认和一致性路径。
- `AioReadWorker`：异步读 worker。
- `ResyncWorker`：恢复与同步。
- `CheckWorker`：空间水位和健康检查。
- `DumpWorker`：导出元数据。
- `AllocateWorker`：空间分配。
- `SyncMetaKvWorker`：同步元数据 KV。
- `BufferPool`：RDMA buffer 池。
- 多套 `DynamicCoroutinesPool`：将读、写、同步等请求分池执行。

### 2.4 `hf3fs_fuse_main`：FUSE 客户端

FUSE 客户端是 3FS 对应用最友好的接入层。入口是 [src/fuse/hf3fs_fuse.cpp](../src/fuse/hf3fs_fuse.cpp) 和 [src/fuse/FuseApplication.h](../src/fuse/FuseApplication.h)。

它负责：

- 将 3FS 挂载为一个标准文件系统。
- 将 POSIX 风格文件操作转换为 meta/storage RPC。
- 管理 FUSE 内部 inode 缓存、写缓冲、周期性 sync、失效通知等。
- 为原生 USRBIO 提供共享内存队列、I/O ring 和 buffer 支持。

核心对象是 `FuseClients`，里面包含：

- `MgmtdClientForClient`
- `MetaClient`
- `StorageClient`
- inode 缓存与目录句柄状态
- IOV/IOR 表
- 写回缓冲和周期 sync 逻辑

因此 FUSE 客户端本质上不是一个简单代理，而是一个包含缓存、并发控制和零拷贝 I/O 基础设施的用户态客户端运行时。

### 2.5 `monitor_collector`：监控采集服务

`monitor_collector_main` 是一个单独的监控汇聚服务，用于接收其它组件上报的指标样本，并写入 ClickHouse。其主体在 `src/monitor_collector/service/MonitorCollectorService.h`。

源码和部署文档显示：

- 默认 reporter 类型为 `clickhouse`。
- 其它服务通过 TCP 将指标发送给 monitor collector。
- 部署脚本会先导入 `deploy/sql/3fs-monitor.sql` 创建表结构。

## 3. 一次读写请求是怎样流动的

### 3.1 元数据路径

文件系统语义相关请求先进入 meta 服务：

1. 客户端通过 `MetaClient` 发起操作。
2. `MetaOperator` 进入 `MetaStore` 和事务层。
3. 元数据持久化到 FoundationDB 或 Hybrid KV 引擎。
4. 如果请求涉及布局、truncate、GC 或 chunk 删除，会再协同 storage 层。

这条路径保证目录树操作、权限、inode 更新、rename 等文件系统语义由一个统一的元数据面来实现。

### 3.2 数据读路径

数据读不是每次都经过 meta：

1. 文件打开后，客户端先从 meta 拿到 inode 和布局。
2. 客户端根据 inode、chunk index、chain table 计算 chunk 位置。
3. `StorageClient` 根据 mgmtd 的 routing info 选择可读 target。
4. 请求经 RDMA/RPC 发给 storage 服务。
5. storage 返回 committed 版本的数据。

这也是设计文档中强调“元数据不在关键数据读路径上”的原因。

### 3.3 数据写路径

写请求的链路更复杂：

1. 客户端依据文件布局将写入切分为 chunk 级别请求。
2. `StorageClient` 将请求发往链头或合适 target。
3. storage 在本地读旧块、合成新块、写 pending 版本。
4. 写入沿复制链向后转发。
5. tail 提交 committed 版本并回 ACK。
6. ACK 沿链反向传播，前序节点将 pending 升级为 committed。

这套机制来自文档中的 CRAQ 思路，核心目标是用链复制保证强一致，同时提升 SSD 与 RDMA 网络利用率。

## 4. 代码仓库结构

下面按“对理解项目最有帮助”的方式说明目录，而不是机械列文件。

### 4.1 顶层目录

- `src/`：核心源码。
- `tests/`：C++ 单元测试、集成测试、FUSE 测试脚本。
- `configs/`：各服务配置模板。
- `deploy/`：手工部署说明、systemd 文件、数据布局工具、监控 SQL。
- `docs/`：架构说明和设计笔记。
- `specs/`：P 语言形式化规格与模型测试。
- `benchmarks/`：性能基准，尤其是 `fio_usrbio`。
- `hf3fs_fuse/`：Python 侧 FUSE/USRBIO 绑定包装。
- `hf3fs_utils/`：Python CLI 工具，提供 `rmtree`、`mv` 等辅助命令。

### 4.2 `src/` 核心模块

#### 通用基础层

- `src/common/`
  提供应用框架、配置系统、日志、监控、网络、序列化、通用工具。
- `src/core/`
  为具体服务提供复用的启动器、服务层和用户管理能力。README 明确说明它是 `mgmtd`、`meta`、`storage` 等服务的底座。
- `src/fbs/`
  放置协议定义与序列化相关代码，按 `meta/mgmtd/storage/core` 等域拆分。
- `src/stubs/`
  客户端/服务端 RPC stub 封装。

#### 存储与元数据主体

- `src/mgmtd/`
  集群管理器实现，包括 store、service、ops 和 background 任务。
- `src/meta/`
  元数据服务实现，包括事务操作、inode/目录项、组件和事件。
- `src/storage/`
  存储服务实现，包括 AIO、同步、更新、worker 和本地存储引擎封装。
- `src/fdb/`
  FoundationDB 封装以及 Hybrid KV 能力。
- `src/kv/`
  LevelDB、RocksDB、内存 KV 等抽象实现。

#### 客户端与对外能力

- `src/client/`
  管理、元数据、存储客户端，以及 `admin_cli` 命令行。
- `src/fuse/`
  FUSE 文件系统客户端。
- `src/lib/api/`
  C/C++ 侧 USRBIO API。
- `src/lib/py/`
  Python USRBIO 绑定。
- `src/lib/rs/`
  Rust FFI 绑定。

#### 扩展和示例

- `src/storage/chunk_engine/`
  新的 Rust chunk engine，负责 chunk 分配和元数据持久化。
- `src/simple_example/`
  新服务模板示例。
- `src/migration/`
  迁移服务框架，接口已经成型，但明显不是仓库主路径。
- `src/client/trash_cleaner/`
  Rust 编写的垃圾桶清理工具。
- `src/tools/`
  额外管理命令入口。

### 4.3 `tests/` 测试结构

测试按域分层，覆盖面较广：

- `tests/common/`：基础组件测试。
- `tests/kv/`、`tests/fdb/`：KV 和 FDB 相关测试。
- `tests/meta/`：inode、目录项、各种元操作测试。
- `tests/storage/`：客户端、存储服务、store、sync 等测试。
- `tests/mgmtd/`：管理面测试。
- `tests/client/`：客户端逻辑测试。
- `tests/migration/`：迁移服务测试。
- `tests/fuse/`：启动本地 FDB + mgmtd + meta + storage + fuse 的脚本与 Python 测试。

这说明项目并非只停留在“论文/架构原型”，而是有比较完整的工程级测试体系。

## 5. 构建系统与技术栈

### 5.1 C++ 构建主线

项目主构建系统是 CMake。顶层 [CMakeLists.txt](../CMakeLists.txt) 显示：

- 使用 `C` 和 `C++` 两种语言。
- C++ 标准为 C++20。
- 默认构建类型是 `RelWithDebInfo`。
- Clang 下启用协程支持，链接 `lld`。
- 大量第三方库以 `third_party/` 子目录方式直接纳入构建。

主要三方依赖包括：

- folly
- fmt
- RocksDB
- LevelDB
- zstd
- scnlib
- pybind11
- toml11
- mimalloc
- clickhouse-cpp
- liburing
- Boost
- FoundationDB C client
- libfuse

### 5.2 Rust 子工程

根 [Cargo.toml](../Cargo.toml) 定义了一个 workspace，当前成员包括：

- `src/client/trash_cleaner`
- `src/storage/chunk_engine`
- `src/lib/rs/hf3fs-usrbio-sys`

其中最重要的是 `src/storage/chunk_engine`，它通过 C++/Rust FFI 接入主存储服务。`src/storage/CMakeLists.txt` 里通过 `add_crate(chunk_engine)` 把 Rust crate 集成进 CMake 构建。

### 5.3 Python 辅助能力

仓库中有两条 Python 线：

- `setup.py`：构建 `hf3fs_py_usrbio`，把 CMake 产物封装为 Python 扩展。
- `setup_hf3fs_utils.py`：构建 `hf3fs_utils` 工具包。

因此 Python 在这个项目里不是核心执行层，而是为了提供运维工具和高性能 I/O 绑定。

## 6. 服务启动与配置模型

### 6.1 Two-phase 启动模型

`mgmtd`、`meta`、`storage` 统一使用 `TwoPhaseApplication<Server>` 作为应用骨架。

这个骨架反映了仓库里非常关键的一套约定：

1. 先由 `Launcher` 读取启动侧配置。
2. 再通过远端配置拉取器获取服务模板。
3. 初始化公共组件，如 IB、日志、监控、内存配置。
4. 构造具体 `Server` 实例。
5. 启动 server 并暴露 RPC 服务。

### 6.2 三层配置拆分

从 `ServerLauncher`、部署文档和测试脚本可以看出，服务通常有三类配置：

- `app_cfg`
  和实例自身强相关，典型字段是 `node_id`。
- `launcher_cfg`
  和启动过程相关，典型字段是 `cluster_id`、`mgmtd_server_addresses`、`fdb.cluster`、IB 设备信息。
- `cfg`
  服务真正运行时使用的完整配置，包括线程池、日志、端口、超时、GC、buffer 池、重试参数等。

这三层分离有两个明显收益：

1. 同一份服务模板可以被多个实例复用。
2. 真正的服务配置可以由 mgmtd 统一分发和热更新。

### 6.3 默认端口和网络

按配置样例：

- `mgmtd`：RDMA `8000`，TCP `9000`
- `meta`：RDMA `8001`，TCP `9000`
- `storage`：RDMA `8000`，TCP `9000`

这里不要把“端口相同”简单理解为冲突。它们通常部署在不同节点上；而 meta 的 RDMA 端口样例单独使用了 `8001`。

## 7. 核心一致性与存储设计

### 7.1 元数据一致性

元数据依赖事务型 KV 存储，设计说明文档明确指出生产环境用的是 FoundationDB。Meta 服务自身是无状态的，借助后端 KV 的事务特性实现文件系统语义上的一致性。

### 7.2 数据一致性

数据块采用 CRAQ 风格链式复制：

- 写走链头到链尾。
- 读可从合适副本读取。
- 每个 chunk 维护 committed/pending 版本。
- 故障时由 mgmtd 更新 chain 和 routing info。

这套机制的目标是兼顾：

- 强一致
- 高吞吐
- 多副本容错
- 存储节点故障恢复

### 7.3 新旧存储引擎关系

仓库同时存在传统 `src/storage/store/` C++ 存储实现和新的 Rust `chunk_engine`。

从 [src/storage/store/ChunkEngine.h](../src/storage/store/ChunkEngine.h) 可见，当前主服务已经通过 FFI 调用 Rust `chunk_engine::Engine` 来：

- 查询 chunk 元数据
- 获取文件描述符和偏移
- 更新 chunk
- 处理未提交块
- 清空 chain 上的块
- 导出所有元数据

这说明 Rust chunk engine 不是孤立实验代码，而是已经接入现有 storage 服务读写路径的重要组件。

## 8. 客户端体系

### 8.1 管理客户端 `admin_cli`

`src/client/bin/admin_cli.cc` 是仓库里非常重要的运维入口。它会按需初始化：

- `MgmtdClientForAdmin`
- `MetaClient`
- `StorageClient`
- `CoreClient`
- FDB 上下文

`registerAdminCommands.cc` 注册了大量管理命令，例如：

- 集群初始化
- 上传 chain/chain table
- 节点注册和摘除
- 配置下发与热更新
- 文件和目录管理
- chunk 查询、目标查询、session 查询
- GC、孤儿块和数据校验相关命令

这也是部署文档中几乎所有操作都围绕 `admin_cli` 展开的原因。

### 8.2 FUSE 客户端

普通应用主要通过 FUSE 使用 3FS。适合：

- 标准 POSIX 接入
- 最低迁移成本
- 数据分析和离线任务

代价是 FUSE 自身的队列锁、内核用户态拷贝、并发写限制等性能问题。

### 8.3 USRBIO / 原生 I/O 客户端

`src/lib/api/UsrbIo.md` 说明了项目提供 USRBIO 原生接口。它用共享内存 `Iov` 和共享 ring `Ior` 让应用和 FUSE 进程之间进行高性能协作，避免标准 FUSE 路径上的部分瓶颈。

适合：

- 小随机读较多的场景
- 对延迟与吞吐更敏感的应用
- 希望尽量靠近 `io_uring` 风格接口的上层程序

## 9. 部署方式

仓库提供的是“手工可控”的工程部署方案，而不是一键容器编排。

### 9.1 生产/准生产部署

[deploy/README.md](../deploy/README.md) 给出了一个 6 节点部署示例：

- 1 台 meta 节点
- 5 台 storage 节点
- 依赖独立或同机部署的 FoundationDB 和 ClickHouse

部署流程大致是：

1. 编译得到 `build/bin` 下的二进制。
2. 部署 monitor collector 并初始化 ClickHouse 表。
3. 部署 `admin_cli`。
4. 部署 `mgmtd` 并初始化集群。
5. 部署 `meta`。
6. 部署 `storage`。
7. 通过 `admin_cli` 创建设备 target、上传 chain 和 chain table。
8. 启动 FUSE 客户端。

仓库还提供了 `deploy/systemd/*.service` 文件，说明默认运维方式偏向 systemd 管理。

### 9.2 本地/CI 测试集群

`tests/fuse/run.sh` 是理解整个系统最有效的脚本之一。它会在本机自动：

- 启动一个临时 FoundationDB 实例
- 生成配置文件
- 用 `admin_cli` 初始化集群
- 启动 `mgmtd`、`meta`、`storage`、`hf3fs_fuse_main`
- 创建 target 和 chain table
- 等待挂载点可用后再跑测试

如果要快速理解系统“最小闭环怎么跑起来”，优先读这个脚本。

## 10. 测试与验证体系

### 10.1 C++/CTest 测试主线

顶层 CMake 启用了 `enable_testing()`，`tests/` 下按模块拆分子目录，说明 C++ 测试是主验证路径。测试覆盖：

- common 工具和网络层
- FDB 和 KV 引擎
- meta 操作
- storage 客户端和服务端
- mgmtd 行为
- migration 服务

### 10.2 FUSE 集成测试

`tests/fuse/` 包含：

- Python 脚本
- shell 脚本
- 一整套测试配置模板

这部分用于验证完整本地集群和挂载行为，覆盖真实文件系统视角的交互。

### 10.3 形式化规格验证

`specs/` 目录保存了 P 语言规格：

- `DataStorage`：验证 CRAQ 相关行为
- `RDMASocket`：验证 RDMA socket 实现
- `Timer`：计时器相关规格

这说明项目不仅有传统单元测试，还保留了协议/状态机级的形式化验证资产。

### 10.4 部署侧 Python 测试

`deploy/data_placement/` 目录包含 Python 模型和 pytest 用例，用于数据布局与链规划相关逻辑的验证。

## 11. 关键工程特征总结

从代码和文档综合看，这个项目有几个很鲜明的工程特点。

### 11.1 不是“单机文件系统改分布式”，而是典型控制面/数据面分离架构

`mgmtd`、`meta`、`storage` 的职责边界比较清晰：

- `mgmtd` 管路由、成员关系、配置与租约
- `meta` 管语义、目录树、权限和布局
- `storage` 管块数据、一致性和恢复

### 11.2 很重视运行时配置和在线变更

`TwoPhaseApplication`、`ServerLauncher` 和配置模板拉取逻辑说明，3FS 不是“把所有配置写死在本地文件里”的系统，而是从一开始就按分布式服务治理来设计。

### 11.3 FUSE 只是兼容层，不是性能终点

设计文档和 USRBIO API 都在说明一件事：FUSE 为易用性服务，真正追求性能时，项目提供了绕开 FUSE 瓶颈的路径。

### 11.4 仓库保留了明显的演进痕迹

例如：

- C++ 存储实现与 Rust `chunk_engine` 并存。
- `migration` 和 `simple_example` 提供扩展模板。
- Python、Rust、C++ 三条工具链并存。

这意味着仓库已经不是一个纯粹“从零干净设计”的状态，而是持续演进中的大型工程代码库。

## 12. 建议的阅读顺序

如果接手这个项目，建议按下面顺序阅读，而不是直接从 `src/` 头文件海里硬啃。

1. 先读 [README.md](../README.md)，建立项目目标和场景认知。
2. 再读 [docs/design_notes.md](./design_notes.md)，理解架构、一致性和恢复机制。
3. 看 [deploy/README.md](../deploy/README.md)，弄清楚服务角色和运维流程。
4. 看 [tests/fuse/run.sh](../tests/fuse/run.sh)，理解最小可运行集群。
5. 读 [src/common/app/TwoPhaseApplication.h](../src/common/app/TwoPhaseApplication.h) 和 [src/core/app/ServerLauncher.h](../src/core/app/ServerLauncher.h)，理解统一启动模型。
6. 读 `src/mgmtd`、`src/meta`、`src/storage` 三个入口头文件，把职责建立起来。
7. 最后再下钻 `MetaOperator`、`StorageOperator`、`MgmtdOperator` 和 `chunk_engine`。

## 13. 对这个仓库的客观判断

如果把这份仓库当作“项目成熟度评估”的输入，比较稳妥的结论是：

- 它是一个真实工程化的分布式文件系统仓库，不是只含论文原型代码。
- 核心链路覆盖了管理面、元数据面、数据面、FUSE 客户端、原生 I/O API、运维 CLI 和测试资产。
- 部署和验证路径都存在，但对环境依赖较重，尤其依赖 RDMA、FoundationDB、ClickHouse 和特定系统组件。
- 代码量大、模块多、技术栈混合，接手门槛不低，更适合“按链路阅读”而不是“按目录穷举”。

## 14. 可直接执行的上手建议

如果目标是尽快把项目跑起来并开始改代码，建议按下面节奏进行：

1. 先在一台开发机上把 CMake 主体编过，确认依赖齐全。
2. 跑通 `tests/fuse/run.sh` 这类本地最小集群脚本，优先验证系统闭环。
3. 用 `admin_cli help` 梳理可操作命令集。
4. 想看元数据语义，优先读 `tests/meta/store/ops/`。
5. 想看数据面，优先读 `src/storage/service/`、`src/storage/store/`、`src/storage/chunk_engine/`。
6. 想看性能路径，优先读 `src/fuse/` 和 `src/lib/api/UsrbIo.md`。

---

这份文档基于当前仓库源码、现有 README、部署脚本和设计说明整理，重点是帮助新接手的工程师快速建立“系统整体图”和“源码阅读路径”。
