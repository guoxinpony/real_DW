# Flink 维度表Join 方案

### 1. 预加载维表

- [ ] **实现方式：** 在算子的 `open()` 方法中，通过 JDBC 或其他客户端**一次性将维度数据全部加载到内存**（如 `HashMap`）中
- [ ] **优化：** 定时更新， 可以配合 Flink 的 `ProcessFunction`，通过**设置定时器**（Timer）或启动一个单独的线程，**定期重新查询数据库**以更新内存中的数据
- [ ] **优点：** 查询性能极高，完全是内存操作，不产生网络开销
- [ ] **缺点：** 占用 TaskManager 内存；如果数据量大，会导致 OOM；无法实时感知维度更新



### 2. 热存储在高性能的外部 KV 系统Redis、HBase

- [ ] **Async I/O：** 由于外部查询是**同步阻塞**的，使用 Flink 的 `AsyncDataStream` 可以**显著提高吞吐量**，避免 CPU 等待 I/O，通过单个线程并发处理大量 I/O 请求
- [ ] **本地缓存（Guava Cache）：** 为了减轻外部系统的压力，通常在 Flink 算子内部设置一个带 TTL（过期时间）的 LRU 缓存，优先从缓存读取
- [ ] **缺点：** 使用缓存时需要权衡**数据一致性**和**性能**。缓存时间越长，性能越高，但感知维度变更的延迟也越大



### 3. 广播维表

- [ ] 将**维度数据表**转化为 DataStream **数据流**
- [ ] 调用 broadcast(MapStateDescriptor) 将其变为 BroadcastStream**广播流**
- [ ] 主流数据调用 connect() **连接该广播流**
- [ ] 在 BroadcastProcessFunction 中**处理关联逻辑**
- [ ] **维度表数据量适中**，通常在几十 MB 到几百 MB 之间
- [ ] **维度表**频繁变更**，且需要使用Flink CDC使主流实时捕捉**到这些变更



### 4. Lookup Join

- [ ] **Lookup Connector：** 必须使用支持 Lookup 接口的 Connector（如 JDBC, HBase, Redis）
- [ ] **Processing Time：** 目前主流的支持是**基于处理时间**（Processing Time）的关联
- [ ] **配置优化：** 在 SQL DDL 中通常可以配置 lookup.cache.max-rows（**缓存大小**）和 lookup.cache.ttl（**缓存过期时间**）
