Flink 的内存模型在 1.10 版本之后经历过一次大重构。简单来说，它是为了解决**‘容器化环境（如 K8s/YARN）下内存不可控’**的问题

##### 1. JVM 自己的开销

- [ ] JVM 运行环境预留
- [ ]  **Metaspace（元空间）** + **Overhead（执行开销）**
- [ ] 防止在容器里运行时，因为 JVM 自身占了内存导致被操作系统（OOM Killer）杀掉



##### 2. 堆内内存 (JVM Heap) 

- [ ] **任务堆内存** (Task Heap)
- [ ] 业务**代码**、算子里的各种**数据对象**，都在这里
- [ ] 如果这里满了，就会报我们熟悉的 **Java.lang.OutOfMemoryError**



##### 3. 堆外内存 (Off-Heap)

- [ ] **管理内存 (Managed Memory)**， 由 Flink 自己管理
- [ ] **状态后端：**如果用了 RocksDB 存状态，它占用的就是这块内存
- [ ] **批处理运算：** 比如排序、哈希表操作
- [ ] **网络内存：** 专门用来在不同节点之间传输数据用的缓冲区



##### 4. 如何管理？

- [ ] 在实际调优中，最关心的其实就是 **Task Heap（堆内）** 和 **Managed Memory（管理内存）** 的比例
- [ ] 如果你用的是 **HashMap 状态后端**，就得把堆内内存调大
- [ ] 如果你用的是 **RocksDB 状态后端**，就得保证 Managed Memory 足够大，否则 RocksDB 性能会变差甚至崩溃



##### 5.   为什么 RocksDB 喜欢用 Managed Memory？

- [ ] **避免不可预知的 OOM：**如果不加管理，RocksDB 会无限制地抢占堆外内存。通过托管内存，Flink 可以利用 `CacheCapacity` 限制 RocksDB 的 Block Cache 占用，确保总内存消耗在容器限额内
- [ ] **内存共享与动态分配：** 在同一个 TaskManager 中可能运行多个 Slot。托管内存允许所有 RocksDB 实例共享一个内存池。这意味着 Flink 可以根据 Slot 的负载情况，更灵活地分配 Block Cache、Write Buffer 
- [ ] **减少 GC 压力：** RocksDB 的数据存储在堆外。使用 Managed Memory 意味着大部分状态数据不需要经过 JVM 的垃圾回收（GC）管理，这对于处理大规模数据至关重要，能显著**降低 Stop-The-World 的频率**



##### 6.  TB 级状态下的增量快照原理

- [ ] RocksDB 的**增量快照**依赖其底层 **LSM-tree（Log-Structured Merge-tree） 的不可变性**
- [ ] **MemTable Flush：** 当 Checkpoint 触发时，RocksDB 先将内存中的 MemTable 刷写到磁盘，生成新的 **Sorted String Table 文件**
- [ ] **差异识别：** Flink 会记录自上一个成功的 Checkpoint 以来，哪些 SST 文件是**新增**的。由于 LSM-tree 的特性，旧的 SST 文件一旦写入就是不可变的（除了解压/合并操作外）
- [ ] **上传新文件：** Flink 只将这些新增的 SST 文件上传到持久化存储（如 HDFS 或 S3）
- [ ] **引用计数（Shared State）：** 状态后端会维护一个元数据文件，记录哪些 Checkpoint 引用了哪些 SST 文件。即使一个 SST 文件是多个 Checkpoint 共用的，只要还有 Checkpoint 在引用它，它就不会被删除
