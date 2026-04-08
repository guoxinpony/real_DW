**为什么需要算子链operator chain优化**

- [ ] 如果每个算子都独立运行在不同线程，算子之间的数据传递需要**序列化 + 网络/跨线程通信**，开销很大
- [ ] Operator Chain 的作用是：把**满足条件的相邻算子**合并到同一个 SubTask 线程里，数据以**函数调用**的方式直接传递，零序列化开销

```
合并前：[Source] → 网络 → [map] → 网络 → [filter] → Shuffle → [KeyBy+window]
合并后：[Source → map → filter]  →  Shuffle  →  [KeyBy+window]
         ↑ 一个SubTask线程                          ↑ 另一个SubTask线程
```



**合并成算子链的条件**

- [ ] 上下游算子之间是 **Forward 分区**策略：map、filter、flatMap等一对一传递，Rebalance/Rescale/keyBy/broadcast/global都不可以链化
- [ ] 上下游算子的**并行度相同**
- [ ] **算子链功能未被禁用**：未开启`disableChaining()` 或者 `startNewChain()`



**在我的项目中的应用**

在我的项目中：

```
Flink CDC Source
  → rebalance/rescale（解决Kafka分区倾斜）
  → filter（筛选维度数据变更）
  → map（解析Debezium JSON）
  → keyBy(userId)
  → window聚合
  → sink(HBase)
```



```
[CDC Source]   
      ↓ rebalance/rescale 打断（非Forward分区）
[rebalance后的处理算子]
      ↓
[filter → map]   ← 可以链化（Forward + 并行度相同）
      ↓ keyBy 打断（KeyGroup分区）
[window → 预聚合]              ← window内部可以链化
      ↓ WindowEnd全局聚合，keyBy打断
[全局聚合 → sink]              ← 可以链化
```

