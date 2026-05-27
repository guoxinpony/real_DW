### 整体架构

- [ ] Doris 是一个**MPP 架构的分析型数据库**
- [ ] **Frontend (FE)** 负责元数据管理、查询解析与调度
- [ ] **Backend (BE)** 负责数据存储与查询执行
- [ ] FE 和 BE 都支持**水平扩展**和**高可用**
- [ ] 整个集群**不依赖外部组件**，如ZooKeeper和HDFS



### Frontend 

| 角色                  | 作用                         | 是否参与选主 | 是否可写  |
| --------------------- | ---------------------------- | ------------ | --------- |
| **Follower (Leader)** | 处理所有元数据写入、查询调度 | ✅            | ✅         |
| **Follower**          | 同步元数据、参与选主         | ✅            | ❌（只读） |
| **Observer**          | 只同步元数据，不参与选主     | ❌            | ❌（只读） |

- [ ] 元数据管理：表结构、分区分桶信息、Tablet 位置、用户权限、统计信息等
- [ ] SQL 解析与优化：解析 SQL → 生成逻辑执行计划 → 基于 CBO (Cost-Based Optimizer) 选择最优物理计划 → 拆分为多个 **Fragment** 下发到 BE
- [ ] 查询调度：按照数据本地性原则（data locality）把 Fragment 调度到数据所在的 BE，减少网络传输。协调各个 BE 的执行并汇总最终结果返回客户端
- [ ] 集群管理：监控 BE 心跳、管理 Tablet 副本的创建/均衡/修复、处理节点上下线
- [ ] 为什么要区分 Follower 和 Observer：Paxos 协议要求**多数派**同意才能写入。如果全是 Follower，扩容时会影响选举效率（比如 5 个 Follower 要 3 个同意，7 个要 4 个同意）。Observer **不参与选举只同步数据**，可以无限扩展来分担读压力，而不影响写入性能



### Backend 

**职责：**

- [ ] **数据存储**：列存格式存储 Tablet 数据
- [ ] **查询执行**：接收 FE 下发的 Fragment，执行扫描、过滤、聚合、Join
- [ ] **数据导入**：接收 Stream Load / Broker Load / Routine Load 等各种导入方式



 **BE 存储模型：**

```
Database
   └── Table
         └── Partition (分区，按日期/范围划分)
               └── Tablet (分桶，数据分片的最小单位)
                     └── Rowset (一次导入产生一个)
                           └── Segment (实际列存文件)
                                 └── Column Data + Index
```



- [ ] **Partition（分区）**：逻辑概念，通常按**时间**分区（如按天），便于分区裁剪和数据淘汰
- [ ] **Tablet（分桶 / Bucket）**：**物理概念**，是**数据分片、副本、负载均衡的最小单位**。每个 Tablet 默认 **3 副本**分布在不同 BE 上
- [ ] **Rowset**：每次导入生成一个 Rowset，类似 LSM-Tree 的 memtable 刷盘
- [ ] **Segment**：真正的列存文件，内部按列存储 + 稀疏索引（类似 Parquet）



 **Unique Key 的两种实现：**

- [ ] Doris 1.2 之前 Unique Key 基于 **Merge-on-Read**（查询时合并多版本），性能较差。 
- [ ] Doris 1.2+ 引入 **Merge-on-Write**：写入时就处理好删除标记，查询性能接近 Duplicate Key，**强烈推荐使用**。



### 完整查询流程

1. Client 发送 SQL 到任意 FE
2. FE 解析 SQL → 生成 AST → 语义分析 → 生成逻辑计划
3. CBO 优化器基于统计信息选择最优物理计划
   - 谓词下推
   - 分区裁剪 (Partition Pruning)
   - 分桶裁剪 (Bucket Pruning)  ← 命中分桶列时只扫描部分 Tablet
   - Join Reorder（选择 Broadcast / Shuffle / Colocate Join）
4. 物理计划拆分为多个 Fragment，按数据本地性调度到 BE
5. 每个 BE 执行分配到的 Fragment（扫描 Tablet、过滤、局部聚合）
6. BE 之间通过 Shuffle 交换中间结果（如 Join、GROUP BY）
7. 最上层 Fragment 做最终聚合，结果返回给协调 FE
8. FE 返回给 Client







