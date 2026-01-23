### 1. 全量 + 增量同步的原理

##### 阶段一：快照阶段（Snapshot）

1. **分片（Chunking）：** 将一张大表按照主键（PK）拆分成多个较小的 **Chunks**（分片）
2. **并行读取：** 多个 TaskManagers 可以**并行地读取**这些 Chunks
3. **确定位点：** 在读取每个 Chunk 时，Flink 会记录该 Chunk 开始前和结束后对应的 Binlog 位点**（Low Watermark 和 High Watermark）**

##### 阶段二：增量阶段（Binlog）

1. 当所有分片读取完成后，Flink CDC 会**自动切换到增量模式**
2. 它会从快照阶段记录的位点开始，持续监听数据库的 Binlog（或类似的增量日志）
3. **流式消费：** 将增量变更（Insert/Update/Delete）实时发送到下游



### 2. 如何解决“锁表”问题

**锁表:**

- [ ] 在 Flink CDC 1.x 版本中，为了保证全量和增量数据的强一致性，通常需要对数据库施加 **全局读锁（Flush Tables With Read Lock）**。这在生产环境中是非常危险的，会**导致业务数据库无法写入**



**如何解决：无锁增量快照算法**

1. **读取 Low Watermark：** 记录当前 Binlog 位点 L
2. **读取 Chunk 数据：** 执行 `SELECT * FROM table WHERE id > x AND id <= y`
3. **读取 High Watermark：** 记录当前 Binlog 位点 H
4. **修正数据：** 如果在读取 L 到 H 的过程中，该 Chunk 范围内的数据发生了变化（记录在 Binlog 中），Flink 会**根据 Binlog 对读取到的 Chunk 结果进行修正**
5. 由于是针对**分片级别的位点追踪**，因此不需要锁定整张表，只需保证分片内的一致性



### 3. 断点续传原理（故障恢复）

1. Flink CDC 的断点续传依赖于 Flink 核心的 **Checkpoint（检查点）机制**
2. **状态持久化：**定期将读取的进度（State）持久化到外部存储
   - [ ] **在全量阶段：** 状态中记录了当前**已经完成的 Chunk 列表**以及**正在读取的 Chunk 的范围**
   - [ ] **在增量阶段：** 状态中记录了当前消费到的 **Binlog Offset（偏移量）**
3. **恢复流程：**
   - [ ] **加载状态：** Flink 从最近一次成功的 Checkpoint 中恢复
   - [ ] **定位读取：** 如果是全量阶段，它会跳过已经处理完的 Chunk，继续读取剩余的 Chunk；如果是增量阶段，它会根据记录的 Offset 重新连接数据库，从该位置继续请求 Binlog
   - [ ] **精确一次（Exactly-once）：** 结合 Flink 的**二阶段提交**或**幂等写入**，可以确保数据既不丢失也不重复