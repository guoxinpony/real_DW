### Flink Local-Global 优化（Flink 1.13）

- [ ] 为了解决因group by引起的数据倾斜，使用两阶段聚合——Key加随机后缀预聚合 + WindowEnd二次聚合
- [ ] Local-Global就是把这件事**自动化**，Flink SQL优化器自动完成
- [ ] 如果某个关键词极度高频，所有包含这个词的数据都会被 KeyBy 路由到**同一个 SubTask**，该 SubTask 数据量是其他的几十倍



**Local-Global 的执行流程**

- [ ] 【Local Agg】— 在每个 SubTask 本地，按 Key 做部分聚合，不跨网络，无 Shuffle，充分利用本地数据局部性
- [ ] 【Global Agg】— 汇总各 SubTask 的局部聚合结果，得出最终结果



**如何触发 Local-Global 优化**

- [ ] 使用 **Flink SQL** 或 Table API
- [ ] **聚合函数必须满足"可分解性"**，即 Local 阶段可以产出部分结果，Global 阶段可以合并：COUNT，SUM，MAX / MIN，AVG； **COUNT DISTINCT 需额外配置**
- [ ] 开启配置开关："table.optimizer.agg-phase-strategy", "AUTO" ，AUTO = 优化器自动判断是否使用Local-Global



**COUNT DISTINCT 的特殊处理以开启 Local-Global**

- [ ] **开启 Split Distinct 优化：**将distinct key散列到N个桶，分桶后在桶内去重再汇总
- [ ] 工作原理：`MOD(HASH(user_id), 1024) AS bucket`，Local阶段分桶内去重，Global阶段汇总各桶结果





**什么情况下 Local-Global 不生效或效果差？**

- [ ] 聚合函数不可分解（自定义 UDAF 未实现 merge 方法）
- [ ] 数据倾斜在时间维度上是均匀的（热 Key 在实时流中均匀分布），Local 阶段聚合收益有限
- [ ] COUNT DISTINCT 未开启 Split Distinct，默认退化为单阶段

