### MySQL Binlog格式

- [ ] STATEMENT：记录SQL语句本身，函数/随机值无法重放
- [ ] ROW：记录每行数据变更前/后的值，也是CDC标准选择
- [ ] MIXED：混合模式，部分场景回退STATEMENT，不稳定
- [ ] **生产环境 CDC 必须用 ROW 格式，Flink CDC 默认也要求这个**





### Debezium/Canal/OGG 对比选型

```json
# Debezium-JSON
{
  "before": {
    "id": 1001,
    "name": "Alice",
    "age": 25
  },
  "after": {
    "id": 1001,
    "name": "Alice",
    "age": 26
  },
  "source": {
    "db": "retail_db",
    "table": "users",
    "ts_ms": 1711771200000,
    "gtid": "xxx"
  },
  "op": "u",
  "ts_ms": 1711771200123
}
```



```json
# Canal-JSON
{
  "data": [
    {
      "id": "1001",
      "name": "Alice",
      "age": "26"
    }
  ],
  "old": [
    {
      "age": "25"
    }
  ],
  "database": "retail_db",
  "table": "users",
  "type": "UPDATE",
  "ts": 1711771200000,
  "es": 1711771200000
}
```



```json
# OGG-JSON
{
  "table": "RETAIL_DB.USERS",
  "op_type": "U",
  "op_ts": "2024-03-30 10:00:00.000000",
  "current_ts": "2024-03-30 10:00:00.123000",
  "pos": "00000000000000001234",
  "before": {
    "ID": 1001,
    "NAME": "Alice",
    "AGE": 25
  },
  "after": {
    "ID": 1001,
    "NAME": "Alice",
    "AGE": 26
  }
}
```



**Debezium和Canal对比**

- [ ] 变更前/后数据：Debezium before/after包含完整行，Canal old只有变更的字段
- [ ] 操作类型字段名：Debezium：‘op:c/u/d/r',   Canal：'type: INSERT/UPDATE/DELETE'
- [ ] 数据类型：Debezium保留原始类型（int是数字），Canal **全部序列化为字符串**



**如何声明JSON格式**

```sql
CREATE TABLE kafka_users_canal (...) WITH (
  'connector' = 'kafka',
  'format' = 'debezium-json'        
);
```



**为什么最终选择了debezium-json**

- [ ] Canal 架构：MySQL Binlog → Canal Server（独立进程）→ Kafka → Flink 消费
- [ ] Flink CDC 架构：MySQL Binlog → Flink CDC Source（内嵌Debezium，无需中间件）→ Flink 算子
- [ ] **组件数量：**Canal JSON 需要独立Canal Server，Flink CDC 集成在flink体系中
- [ ] **全量+增量：** Canal只做增量，全量需另外处理；自动完成全量快照+增量无缝切换
- [ ] **容错性：** Kafka天然持久化，依赖Flink Checkpoint