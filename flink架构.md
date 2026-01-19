### 1. 核心组件

- **Yarn Resource Manager (RM):** YARN 集群的资源管理者，负责分配 Container
- **NodeManager (NM):** 运行在各节点上的 YARN 代理，负责管理容器的生命周期
- **Flink ApplicationMaster (AM):** 在 Application 模式下，AM 包含了 Flink 的核心控制层，包括：
  - [ ] **Dispatcher (分发器):** 接收作业并启动 JobMaster
  - [ ] **ResourceManager (Flink 内部):** 负责向 YARN 申请资源并管理 Slot
  - [ ] **JobMaster (作业管理器):** 管理单个作业的执行图转换和任务调度
- **TaskManager (TM):** 真正的“干活儿”的人，负责执行具体的 Task



### 2. 作业提交流程

**第一阶段：启动控制中心：**

1. **启动 AM:** 客户端向 YARN RM 提交申请，RM 在某个 NodeManager 上启动 ApplicationMaster
2. **初始化内部组件:** AM 启动后，内部的 **Actor 通信系统** 启动。随后，**分发器 (Dispatcher)** 和 **资源管理器 (ResourceManager)** 被拉起
3. **启动 JobMaster:** 分发器根据用户的程序启动 **JobMaster**

**第二阶段：生成执行计划：**

1. **StreamGraph:** 根据 API 生成的最初的**拓扑图**
2. **JobGraph:** 优化后的**逻辑图**
3. **ExecutionGraph:** 并行化后的图，是调度层面的核心参考

**第三阶段：资源申请与任务下发：**

1. **申请资源:** JobMaster 向 Flink RM 请求 Slot，Flink RM 发现资源不足，向 YARN RM 申请新的 Container
2. **启动 TaskManager:** YARN RM 分配资源，在其他 NodeManager 上启动 TaskManager 容器
3. **注册 Slot:** TaskManager 启动后向 Flink RM 注册自己拥有的 Slot 资源
4. **分配 Slot:** Flink RM 将可用的 Slot 分配给 JobMaster
5. **提供 Slot:** TaskManager 向 JobMaster 正式提供 Slot 使用权
6. **分发任务:** JobMaster 生成最终的 **Physical Graph (物理执行图)**，并将任务部署到 TaskManager 的 Slot 中开始运行