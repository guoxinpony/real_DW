### Flink CDC 是什么

- [ ] 监测并捕获数据库的变动
- [ ] Flink CDC 内部内置了 Debezium 引擎，**将 changelog 转换为 RowData** 数据格式
- [ ] 基于日志的 CDC 能够捕获所有数据变化的完整记录，不会像基于查询的方式那样丢失两次查询之间的中间数据
- [ ] Debezium 的角色——它是 Flink CDC 底层实际负责解析 binlog 的引擎



### Flink CDC 1.x 的三大缺点

- [ ] 全局锁导致数据库阻塞：Flink CDC 1.x 在全量读取阶段需要同时获取**binlog 的起始位点，以及当前表的 schema 和全量数据** ，**加全局锁是为了确保在获取 binlog 位点和读取全量数据之间，没有其他写操作改变数据**，锁的本质是为了不重不漏读取
- [ ] 单并行度，大表极慢
- [ ] **全量阶段不支持 Checkpoint。** 全量读取和增量读取分为两个阶段，全量读取阶段不支持 Checkpoint，失败后需要从头重新读取



### Flink CDC 2.x 的核心设计

- [ ] **Chunk 切分，实现并行：** 通过表的主键对数据进行分片
- [ ] **单个 Chunk 的无锁一致性读：** 记录低位点 LP，开始 SELECT [K1, K10] 的数据并存入buffer, SELECT 完成并记录高位点 HP；
- [ ] 如果在执行 SELECT 阶段没有其他事务进行操作（LP 到 HP 之间的 binlog 就是空的），即没有读取到 binlog 数据，直接下发所有快照记录；
- [ ] 如果有并发写入，就用 binlog 中的 INSERT/UPDATE/DELETE 来修正 buffer 中的数据
- [ ] **消费增量binlog:**  每个chunk都汇报自己的HP， 为了不丢数据，增量读取的起始偏移量为所有已完成的全量切片中**最小的 Binlog 偏移量**，为了避免重复，Flink CDC 会做**过滤**，只有**没有被全量阶段处理过的变更**才会被下发到下游



### Flink CDC 与 Flink Checkpoint 的配合，如何保障一致性？

- [ ] Flink CDC Source内部维护了当前读取到的 binlog 位点
- [ ] 当 Flink 触发 Checkpoint 时，这个位点会被持久化到状态后端
- [ ] 如果作业失败，会从最近的 Checkpoint 恢复，重新从记录的 binlog 位点开始消费，保证 exactly-once 语义
- [ ] **项目中的流程：**Flink CDC 监听配置表变化 → 广播到所有并行度 → 动态感知维度表配置变更，全程无需重启作业







### 全量 + 增量同步的原理

##### 阶段一：快照阶段（Snapshot）

1. **分片（Chunking）：** 将一张大表按照主键（PK）拆分成多个较小的 **Chunks**（分片）
2. **并行读取：** 多个 TaskManagers 可以**并行地读取**这些 Chunks
3. **确定位点：** 在读取每个 Chunk 时，Flink 会记录该 Chunk 开始前和结束后对应的 Binlog 位点**（Low Watermark 和 High Watermark）**

##### 阶段二：增量阶段（Binlog）

1. 当所有分片读取完成后，Flink CDC 会**自动切换到增量模式**
2. 它会从快照阶段记录的位点开始，持续监听数据库的 Binlog（或类似的增量日志）
3. **流式消费：** 将增量变更（Insert/Update/Delete）实时发送到下游



### 如何解决“锁表”问题

**锁表:**

- [ ] 在 Flink CDC 1.x 版本中，为了保证全量和增量数据的强一致性，通常需要**对数据库施加 全局读锁（Flush Tables With Read Lock）**。这在生产环境中是非常危险的，会**导致业务数据库无法写入**



**如何解决：无锁增量快照算法**

1. **读取 Low Watermark：** 记录当前 Binlog 位点 L
2. **读取 Chunk 数据：** 执行 `SELECT * FROM table WHERE id > x AND id <= y`
3. **读取 High Watermark：** 记录当前 Binlog 位点 H
4. **修正数据：** 如果在读取 L 到 H 的过程中，该 Chunk 范围内的数据发生了变化（记录在 Binlog 中），Flink 会**根据 Binlog 对读取到的 Chunk 结果进行修正**
5. 由于是针对**分片级别的位点追踪**，因此不需要锁定整张表，只需保证分片内的一致性



### 断点续传原理（故障恢复）

1. Flink CDC 的断点续传依赖于 Flink 核心的 **Checkpoint（检查点）机制**
2. **状态持久化：**定期将读取的进度（State）持久化到外部存储
   - [ ] **在全量阶段：** 状态中记录了当前**已经完成的 Chunk 列表**以及**正在读取的 Chunk 的范围**
   - [ ] **在增量阶段：** 状态中记录了当前消费到的 **Binlog Offset（偏移量）**
3. **恢复流程：**
   - [ ] **加载状态：** Flink 从最近一次成功的 Checkpoint 中恢复
   - [ ] **定位读取：** 如果是全量阶段，它会跳过已经处理完的 Chunk，继续读取剩余的 Chunk；如果是增量阶段，它会根据记录的 Offset 重新连接数据库，从该位置继续请求 Binlog
   - [ ] **精确一次（Exactly-once）：** 结合 Flink 的**二阶段提交**或**幂等写入**，可以确保数据既不丢失也不重复