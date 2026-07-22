# Astra × Git4Data：架构关系、能力边界与产品判断

> 日期：2026-07-22  
> 状态：基于当前代码的研究整理；目标设计与已实现能力分开表述  
> Astra 代码快照：`matrixorigin/astra@9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84`  
> 目的：统一回答 Astra 与 MatrixOne Git4Data 的关系，以及 Session、Run、Agent Binding、Delegation、Task、Reflect、Memoria、Branch、Snapshot 和 Rollback 分别承担什么职责。

## 1. 核心结论

Astra 与 Git4Data 的关系不能简单理解成“用户通过对话上传数据，再像 Git 一样维护这些数据”。更准确的定义是：

> **Astra 是 Agent Runtime，MatrixOne 是其持久状态和数据执行底座；Git4Data 是 MatrixOne 提供的数据版本原语。Astra 的价值是把 snapshot、diff、branch、verify、merge/restore 等原语嵌入 Agent 的 Run、Task 和数据工具调用生命周期。**

这套组合解决的是 Agent 执行真实数据修改时的四个问题：

1. 修改前是否存在可恢复的前态；
2. 多条执行路径能否隔离试验；
3. 修改结果能否经过 diff 和验证后再发布；
4. Agent 的决策、工具调用与数据变化能否形成同一条审计链。

但当前实现还不能被描述成“完整的 Agent Git 系统”。代码中已经存在数据库 pre-state snapshot、按轮回滚工具、耐久任务的 snapshot/diff/verify/rollback，以及 branch HTTP 接口；同时也存在明显边界：

- `/branches` 的当前实现更接近以 snapshot 包装的 branch façade，并非完整的数据分支生命周期；
- Session fork 可以生成数据分支名称和组合快照引用，但底层 MatrixOne 分支 DDL 仍由调用方执行；
- 没有把某个 Session 的全部状态原子恢复到任意时间点的统一能力；
- 数据库恢复、Session 撤销、文件撤销、Git 撤销、Memoria 恢复和外部副作用补偿仍是多个独立恢复域。

因此，当前最准确的产品判断是：

> **Astra 已经具备 Git4Data 驱动的局部安全执行闭环，但“Session/Task/Agent 与数据分支完全绑定、可统一合并和回滚”的能力仍处在部分实现阶段。**

## 2. 正确的整体心智模型

```text
用户对话 / API 请求
        │
        ▼
Astra Session：持续的会话与上下文容器
        │
        ├─ Run A：绑定 Agent Binding A，执行一条推理/工具路径
        │    ├─ Turn / tool calls
        │    ├─ Task Board / background process
        │    └─ Child Run / Delegation
        │
        └─ Run B：可以绑定 Agent Binding B，执行另一条路径

MatrixOne
  ├─ Astra 控制面事实：sessions、runs、events、bindings、delegations、tasks……
  └─ Agent 操作的数据：通过 snapshot / diff / restore 等 Git4Data 原语保护

Memoria
  ├─ working memory
  ├─ episodic memory
  └─ semantic memory，例如 astra:scene
```

这里需要区分三层：

| 层次 | 主要职责 | Git4Data 是否天然生效 |
|---|---|---|
| Astra 控制面 | 保存 Session、Run、事件、Agent 配置、任务、审计和恢复信息 | 不天然生效；存进 MatrixOne 不等于自动建立版本 |
| Agent 数据面 | Agent 查询或修改的 MatrixOne 数据库 | 只有显式创建 snapshot、branch、diff 或 restore 时才获得 Git4Data 语义 |
| Memoria 记忆面 | 工作记忆、会话经验和跨会话可复用知识 | 独立恢复域，不包含在普通 MatrixOne 数据库回滚中 |

所以，MatrixOne 对 Astra 的帮助首先是**结构化持久化、事务、查询、审计和恢复基础**；Git4Data 是在这些基础之上，由具体执行流程显式使用的增强能力。

## 3. 对话是不是数据上传与维护入口

对话可以作为控制入口，例如用户让 Agent 导入 CSV、执行 SQL、创建数据版本或比较差异，但对话本身不是 Git4Data 的存储协议。

更准确的链路是：

```text
自然语言要求
  → Agent 规划
  → 获得工具和数据权限
  → SQL / import / provider 执行真实数据操作
  → Git4Data 在修改前后建立 snapshot、diff、restore/merge 语义
```

因此：

- 数据内容不一定来自聊天附件，也可以来自 MatrixOne 已有表、文件导入、私网数据源或外部 provider；
- Astra 不会因为保存了聊天内容，就自动把聊天内容转成 Git4Data repository；
- Git4Data 的主要对象是 MatrixOne 中的数据状态，而不是自然语言消息；
- Astra 的差异是让 Agent 知道何时建立前态、何时验证、何时发布或回滚。

## 4. MatrixOne 中保存了哪些 Agent 运行对象

### 4.1 Agent 注册与 Agent Binding 是两类对象

`agent_agents` 是产品层 Agent 注册表，包含：

- `agent_id`、名称、类型和所有者；
- 是否启用；
- `agent_config`；
- `data_source`；
- 创建和更新时间。

对应实现见 [agents.rs](https://github.com/matrixorigin/astra/blob/9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84/crates/services/src/agents.rs) 和 [storage.rs](https://github.com/matrixorigin/astra/blob/9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84/crates/services/src/storage.rs)。

`agent_bindings` 则是一次运行可解析的 Agent Runtime 合约，当前字段包括：

- `agent_md`：Agent 的系统级角色、约束和行为指令；
- `capability_servers`：MCP 和 Skill Server 的端点引用；
- `runtime_policy`：当前只包含 `max_steps` 与 `tool_mode=mcp_gateway`；
- `metadata` 与 `binding_schema_version`；
- active、disabled 或 invalid 状态。

因此，Agent Binding 不只是“工具列表”，而是：

```text
Agent instructions
+ capability server references
+ execution step limit
+ tool access mode
+ schema/version identity
```

当前 API 只提供 create、get 和 disable，没有原地 update。要修改指令、capability 或 policy，更合理的方式是创建新 binding/version，让后续 Run 选择新 binding；已经启动的 Run 保留自己的 binding ID、名称和 schema version，不应被动态改写。实现见 [agent_bindings.rs](https://github.com/matrixorigin/astra/blob/9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84/crates/services/src/agent_bindings.rs) 和 [agent_binding_handlers.rs](https://github.com/matrixorigin/astra/blob/9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84/crates/runtime/src/server/agent_binding_handlers.rs)。

### 4.2 Session 和 Run 的绑定关系

`agent_sessions` 有可选的 `agent_id`，表达会话层的 Agent 归属；但 `agent_binding_id`、binding name、binding schema version、selected model、capability server refs 和 runtime profile 是记录在 `agent_runs` 上的。

所以不能简单理解成“一个 Session 永远只有一个 Agent Binding”。更准确的是：

- Session 是持续上下文和事件容器；
- Run 是一次具体执行实例；
- Agent Binding 在 Run 启动时解析并固化为运行事实；
- 同一个 Session 内，不同 Run 可以选择不同 Agent Binding；
- 同一 Run 执行中不应热切换 binding，否则会破坏重放和审计的一致性。

Run 对 binding 的持久化字段见 [runs.rs](https://github.com/matrixorigin/astra/blob/9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84/crates/services/src/runs.rs) 和 `agent_runs` 表定义。

### 4.3 Session Delegation 不是新 Session

`session_delegations` 记录：

- `parent_run_id`、`child_run_id` 和 `root_run_id`；
- 祖先路径和深度；
- 子 Agent ID、状态和重试范围；
- 最近摘要和可暴露给兄弟节点的 artifacts；
- request/trace identity。

它表达的是同一 Session 运行树中的父 Run → 子 Run 委派。可以把它理解成：

> 在当前任务或当前轮次中启动另一条相对独立的“思考—决策—工具”执行路径，子 Run 完成后把结果返回父 Run。

它通常不是另建一段用户对话，也不要求创建新 Session；如果需要真正隔离会话历史，应使用 Session fork。表定义见 [storage.rs](https://github.com/matrixorigin/astra/blob/9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84/crates/services/src/storage.rs#L3268)。

## 5. Git4Data 在 Astra 中实际嵌入的位置

### 5.1 MatrixOne SQL 工具的 pre-state snapshot

`mo_query` 会检测 SQL 是否包含 INSERT、UPDATE、DELETE、CREATE、DROP、ALTER、LOAD 等修改操作。执行修改之前，它会：

1. 为目标数据库创建 snapshot；
2. 把 snapshot ID、数据库和 turn index 写入回滚 journal；
3. snapshot 创建失败则拒绝继续执行 SQL；
4. 修改后可由 `rollback_database_snapshots` 按当前轮、指定轮或 snapshot ID 恢复。

这是一种明确的 fail-closed 设计：如果无法建立前态，就不允许进行受保护的数据修改。实现见 [tool_database_snapshots.rs](https://github.com/matrixorigin/astra/blob/9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84/crates/runtime/src/server/tool_database_snapshots.rs)。

### 5.2 Durable Task 的执行保护

耐久任务系统的理想闭环已经体现在代码路径中：

```text
创建 subtask
  → 执行前创建 snapshot
  → Agent 执行
  → 捕获相对 snapshot 的 diff
  → Verification Gate
       ├─ 通过：保留结果并清理 snapshot
       └─ 失败且重试耗尽：restore snapshot，再清理 snapshot
```

这对长任务尤其重要，因为失败影响可以从“整个长任务”缩小到“当前 subtask”。对应实现见 [durable_task.rs](https://github.com/matrixorigin/astra/blob/9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84/crates/services/src/durable_task.rs)。

需要注意：当前 `MatrixOneTaskBranchOps` 使用的是数据库级 snapshot；源码中的“per-agent branches”是架构目标，而核心生产实现仍以 snapshot、diff 和 restore 为主。

### 5.3 Branch HTTP API

当前服务暴露：

- `POST /branches`
- `POST /branches/diff`
- `POST /branches/merge`
- `DELETE /branches`
- `POST /branches/cost-estimate`

但必须按实现而不是按路由名称判断成熟度：

- create 当前为配置数据库创建 snapshot，默认名称为 `{branch_name}__snap`；
- diff 调用 `mo_diff(source, target)`，非 count 输出目前只返回空对象占位；
- merge 当前通过 restore source snapshot 到配置数据库实现，`target` 没有真正参与独立分支合并；
- delete 实际执行的是 `DROP SNAPSHOT`；
- 系统 E2E 目前只覆盖了 cost-estimate，没有覆盖真实 create/diff/merge DDL 路径。

所以 `/branches` 当前更接近“以 branch 术语包装的数据快照服务”，不能直接宣称已交付完整的 Git 式三方合并。实现见 [branches.rs](https://github.com/matrixorigin/astra/blob/9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84/crates/services/src/branches.rs)；测试覆盖边界见 [system-e2e-matrix.md](https://github.com/matrixorigin/astra/blob/9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84/docs/testing/system-e2e-matrix.md)。

## 6. “Branch”在 Agent 系统中的三种含义

当前代码和产品讨论中至少有三种 branch，必须避免混用：

| Branch 类型 | 分叉的对象 | 作用 | 当前边界 |
|---|---|---|---|
| Git/code branch 或 worktree | 文件与代码版本 | 让代码 Agent 隔离修改、提交和合并 | 属于文件/Git 工具域，不是 MatrixOne Git4Data |
| Session fork | 会话日志、上下文和运行谱系 | 从父 Session 的某个 turn 派生实验会话 | 可复制本地 journal/workspace/checkpoint；不自动完成数据分支 DDL |
| MatrixOne data branch/snapshot | 数据库状态 | 隔离数据修改、diff、发布或恢复 | 当前主要落在数据库 snapshot/restore，完整 merge 语义仍不成熟 |

Session fork 中的 `create_data_branch=true` 当前只生成类似 `session_fork_{new_session_id}` 的分支名，并明确把 MatrixOne DDL 执行责任留给调用方。它还可以构建 Composite Snapshot，引用 transcript、checkpoint、workspace、task、artifact、invocation 和 memory 等维度，但缺失维度会被标记为 gap，而不是假装已经完整捕获。见 [session_fork.rs](https://github.com/matrixorigin/astra/blob/9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84/crates/services/src/session_fork.rs) 和 [composite_snapshot.rs](https://github.com/matrixorigin/astra/blob/9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84/crates/core/src/composite_snapshot.rs)。

## 7. Snapshot 在什么时间点创建

不同 snapshot 的语义完全不同：

| Snapshot 类型 | 创建时间点 | 用途 | 是否能恢复数据库 |
|---|---|---|---:|
| Context snapshot | LLM 调用前记录模型实际看到的输入 | 审计、决策解释和重放证据 | 否 |
| `mo_query` pre-state snapshot | 修改型 SQL 执行前 | 按数据库恢复该轮数据修改 | 是 |
| Durable subtask snapshot | 子任务实际执行前 | diff、验证失败回滚 | 是或由选定 BranchOps 决定 |
| `/branches` snapshot | 调用 create branch API 时 | 保存配置数据库当前状态 | 是 |
| Session fork Composite Snapshot | 从父 Session 分叉的 turn 上 | 记录多个状态维度的引用 | 只在包含真实 data snapshot ref 时部分支持 |
| Task Manager snapshot | `task_board` 状态修改前 | 恢复任务列表和状态 | 否；这是逻辑状态快照 |
| Memoria snapshot | 记忆工具执行相关操作时 | 恢复记忆内容 | 否；属于 Memoria 恢复域 |

因此，“Astra 有 snapshot”不能直接推导出“整个 Agent 可随时恢复”。必须先回答：快照覆盖的是模型上下文、任务板、数据库、文件，还是记忆。

## 8. 数据库级回滚对 Agent 的影响

`rollback_database_snapshots` 最终使用 MatrixOne 的数据库 restore，把选定数据库恢复到 snapshot。它的优点是恢复范围明确、速度通常比逐条补偿 SQL 更可控；但影响面也比单行或单任务回滚大。

主要风险包括：

1. snapshot 之后该数据库中其他 Session 或其他 Agent 的合法写入也可能被撤销；
2. 如果 Astra 控制面表与 Agent 业务数据位于同一个被恢复数据库，Session、Run、事件、Binding、任务和审计记录都可能一起倒退；
3. 文件、Git、外部 API、消息发送和第三方系统副作用不会被数据库 restore 撤销；
4. Memoria 如果是独立服务或独立存储，也不会同步回滚；
5. 回滚 journal 当前是活动 Runtime 中的进程内索引，重启后不能把它当成完整历史目录；已创建 snapshot 可能仍在 MatrixOne，但需要另行发现和治理。

因此生产部署中应优先隔离：

```text
Astra control-plane database
≠ Agent business-data database
≠ Memoria store
```

并把 snapshot 与 `user_id / session_id / run_id / task_id / database / created_at / retention` 建立持久关联，避免只依赖进程内 journal。

## 9. Task 工具与 Git4Data 的关系

当前存在两组容易混淆的 Task 工具。

### 9.1 `task_board`

`task_board` 是耐久的 Session 任务板，用于 create、update、list、get、stop、adopt 和 archive。它管理的是逻辑任务状态、依赖关系、进度和所有权。

`task_board` 修改前可以捕获 `TaskManagerSnapshot`，再由 `rollback_session_state` 恢复；MatrixOne task store 还使用版本检查，避免把并发更新错误覆盖。这属于**应用状态回滚**，不是 MatrixOne Git4Data 数据库 snapshot。

### 9.2 `task_output`、`task_list`、`task_stop`

这三个工具控制本地或 Edge 上的 typed background process：

- `task_list`：查看后台进程；
- `task_output`：按 task ID 读取输出，可选择快照读取或等待；
- `task_stop`：终止后台进程。

代码明确把它们定义成 `BackgroundTaskProcess`，与 durable task board 分离。它们解决的是“长时间 shell/工具进程如何继续运行、取输出和停止”，不是数据版本管理。

### 9.3 与 Git4Data 的组合关系

合理的组合是：

```text
task_board 定义任务与验收标准
  → background task / child run 执行
  → Git4Data snapshot 保护数据前态
  → diff + verification 判断结果
  → task_board 更新状态
  → 成功发布或失败回滚
```

Task 系统负责“做什么、做到哪一步”；Git4Data 负责“数据改成什么、如何隔离与恢复”。两者互补，但不是同一个系统。

## 10. Reflect、Memoria 与 Git4Data 的关系

顶层 `reflect` 工具会读取 MatrixOne 中的 `agent_events`、决策审计、错误模式等持久证据，生成当前 Session 的分析报告。它本身不自动把结论写入 Memoria，也不创建 Git4Data snapshot。

真正的经验沉淀发生在 Session-end governance：

1. 将会话概述写入 Memoria episodic memory；
2. 清理已被安全持久化的 working memory；
3. 调用 Memoria reflect；
4. 把候选场景写入带 `astra:scene` 标签的 semantic memory。

见 [session_end_governance.rs](https://github.com/matrixorigin/astra/blob/9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84/crates/runtime/src/turn/cloud/session_end_governance.rs) 和 [memoria_compact.rs](https://github.com/matrixorigin/astra/blob/9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84/crates/runtime/src/turn/cloud/memoria_compact.rs)。

三者的职责可以概括为：

```text
MatrixOne evidence：发生了什么
Reflect：这些事实意味着什么
Memoria：以后应该记住什么
Git4Data：数据状态如何隔离、比较、发布与恢复
```

Reflect 对长任务的帮助是减少盲目重试和策略漂移；Git4Data 对长任务的帮助是缩小失败影响面、保留可验证前态。一个改善决策过程，一个保护数据结果。

## 11. 当前是否支持单 Session 撤销

支持局部撤销，但不支持完整的 Session 时间点恢复。

### 已有能力

- `/undo [N]`：从当前 Session 的活动历史中撤销最近 N 轮，并回退文件 journal 记录的对应文件修改；同时刷新 recovery snapshot 和 Session memory projection；
- `/redo`：恢复被 `/undo` 移出的轮次；
- `rollback_session_state`：按当前轮或指定 turn 恢复 config override、Task Board snapshot 和手动 context compression 标记；
- `rollback_file_edits`：恢复文件修改；
- `rollback_database_snapshots`：恢复 MatrixOne 数据库 snapshot。

### 不具备的能力

当前没有统一的：

```text
rollback_session(session_id, target_timestamp | snapshot_id)
```

来原子恢复 transcript、Run tree、events、decisions、task board、database、files、Git、Memoria 和 external side effects。`/undo` 主要改变当前可继续运行的会话投影，不等于删除所有旧事件和审计事实；`rollback_session_state` 的回滚 handle 也主要保存在活动 Runtime 的 journal 中。

因此产品文案应使用“按轮撤销”和“分恢复域回滚”，而不是“整个 Agent 任意时间旅行”。

## 12. 对哪些任务帮助最大

### 12.1 明显受益的任务

| 任务 | Git4Data 的具体价值 |
|---|---|
| 大批量数据清洗与修复 | 修改前留前态，验证失败直接 restore |
| 多步 ETL/数据迁移 | 每个阶段可建立 checkpoint，避免从头重跑 |
| 长周期 Durable Task | 失败范围缩小到 subtask，跨 Session 仍有事实链 |
| 多 Agent 并行探索 | 每条执行路径可以拥有隔离数据版本，验证后再发布 |
| 数据分析实验 | 比较不同 prompt/model/strategy 对同一基线数据的结果 |
| 高风险 SQL 自动化 | fail-closed snapshot 降低误更新和误删除风险 |
| 需要合规审计的流程 | 把 run、工具调用、snapshot、diff、verify 和 rollback 串联 |

### 12.2 帮助有限的任务

- 纯问答和 Web 搜索；
- 只读分析且不生成需要版本化的数据产物；
- 单次短任务、失败后重新执行成本极低；
- 副作用主要发生在外部 SaaS、邮件、支付或消息系统中；
- 文件/代码修改为主、数据数据库几乎不参与的任务，此时 Git/worktree 更直接。

### 12.3 为什么长任务收益更明显

长任务通常具有更多步骤、更长时间跨度、更多并行执行和更多不可预测故障。Git4Data 的收益不是让模型“更聪明”，而是把失败代价从：

```text
失败 → 猜测补偿 SQL → 可能二次破坏 → 整个任务重做
```

变成：

```text
建立前态 → 执行 → diff → 验证
                   ├─ 通过：发布
                   └─ 失败：恢复已知前态
```

任务越长、数据越大、并发越高、修改越危险，这个优势越明显。

## 13. 当前能力成熟度判断

| 能力 | 当前判断 | 依据与限制 |
|---|---|---|
| MatrixOne 持久化 Session/Run/Event/Agent/Task | 已实现 | 有明确表结构、服务和测试 |
| 每个 Run 固化 Agent Binding | 已实现 | binding 字段持久化在 `agent_runs` |
| 修改型 `mo_query` 自动 pre-state snapshot | 已实现 | snapshot 失败时 fail-closed |
| 按 turn/snapshot 恢复 MatrixOne 数据库 | 已实现但范围较粗 | 数据库级 restore，journal 主要为进程内 |
| Durable subtask snapshot/diff/verify/rollback | 已实现核心流程 | 当前生产 MatrixOne 路径以数据库 snapshot 为主 |
| 完整 MatrixOne data branch create/diff/merge | 部分实现 | HTTP 接口存在，但 create/delete/merge 主要映射 snapshot/restore，E2E 不完整 |
| Session fork 自动创建真实数据分支 | 部分实现 | 当前生成分支名，DDL 由调用方负责 |
| 单 Session 完整时间点恢复 | 未实现 | 只有按轮、按恢复域的局部撤销 |
| Reflect 自动写长期经验 | 条件实现 | 顶层 reflect 不写；Session-end governance + Memoria 才沉淀 |
| 跨数据库、文件、Git、Memoria、外部 API 的统一事务 | 未实现 | 各恢复域独立，需要补偿编排 |

## 14. 产品化建议

若要把 Git4Data 发展成 Astra 的稳定竞品优势，优先级建议如下：

1. **建立持久 Data Version Registry**：把 snapshot/branch 与 user、session、run、task、database、baseline、status、TTL 和审批记录关联，替代仅依赖进程内 journal。
2. **隔离控制面与业务数据面**：默认禁止对 Astra control-plane database 执行 Agent 数据回滚。
3. **统一 Data Change Boundary**：任务执行前创建版本，执行后自动 diff，验证通过后 merge/publish，失败后 rollback，并将每一步写入同一审计事件链。
4. **补齐真实 branch 语义和 E2E**：明确 `source`、`target`、snapshot 和 conflict policy 的真实行为，避免用 restore 假装 merge。
5. **实现 Composite Recovery Plan**：不追求不现实的全局 ACID，而是对 database、task state、files、Git、Memoria 和 external effects 生成可审计的分域恢复计划。
6. **提供 Session-level Recovery API**：允许选择 target turn，先评估会影响的副作用，再执行已支持域的恢复并明确报告未恢复项。
7. **将 Branch 与 Delegation/Durable Task 绑定**：子 Run 创建时可选自动建立数据分支，子 Run 完成后执行 verify + merge，取消或失败时回滚和清理。
8. **验证真实优势而非只验证接口存在**：重点测主线污染、恢复成功率、恢复时延、冲突检测准确率、存储放大和跨域残留副作用。

相关 Benchmark 设计已经整理在 [Astra 技术差异与 Benchmark 设计](./technical-differences-and-benchmark-design.md) 中。

## 15. 一句话对外表述

现阶段建议使用：

> Astra 将 MatrixOne 的数据快照、差异比较和恢复能力嵌入 Agent 的 Run 与 Durable Task 执行过程，使高风险数据修改可以在执行前留前态、执行后验证并在失败时恢复；完整的数据分支合并和单 Session 全状态时间点恢复仍在演进中。

不建议使用：

> Astra 中的所有会话和 Agent 数据都天然拥有 Git 的完整 branch、merge 和 rollback 能力。
