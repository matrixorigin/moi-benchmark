# Astra 与竞品的 Benchmark Adapter 集成可行性研究

> **已过时的候选集研究：请勿按本文四平台名单实施。** 本文只保留 LangGraph/CrewAI/Dify 的历史接口分析；它们不属于当前 Astra 产品主榜。  
> 当前竞品范围、Runner 契约和执行计划以 [astra-benchmark-plan.md](../plans/astra-benchmark-plan.md) 为准；首期产品为 Astra、Hermes、Codex、Goose。

> 版本：v0.2  
> 日期：2026-07-21  
> 范围：Astra、LangGraph、CrewAI、Dify  
> 结论性质：架构和接口可行性结论，不是产品能力评分

## 1. 结论

将 Astra、LangGraph、CrewAI 和 Dify 接入同一套 Benchmark 系统是可行的。考虑首轮必须在 10 个工作日内完成，建议将目标从“建设通用 Benchmark Adapter 平台”收敛为“完成可复现的对比实验闭环”。半月 MVP 只统一以下边界：

- 部署制品登记与版本锁定；
- 输入 `TaskSpec`、启动单次运行和超时终止；
- 收集最终输出、原生运行 ID、错误和原始产物；
- 通过统一 Tool Gateway 记录工具调用、顺序、重试和副作用；
- 通过模型响应记录 Token 和模型成本。

Adapter 不应负责统一平台内部的 Agent 实现，也不能为缺少某项能力的平台补一个外部编排器。受控 Track 统一的是 Agent Blueprint 的执行语义，各平台仍应使用原生能力实现。

初步判断：

| 平台 | 推荐接入方式 | 运行控制接入 | 深度事件接入 | 主要限制 | 综合可行性 |
|---|---|---:|---:|---|---|
| Astra | `@astra/sdk` / REST + SSE | 高 | 高 | 需区分已实现接口与目标设计契约 | 高 |
| LangGraph | Agent Server + LangGraph SDK | 高 | 高 | Graph 本身不定义统一 Agent 架构；部署层与 OSS 库层需明确 | 高 |
| CrewAI OSS | Python Sidecar + Flow/Crew hooks | 中 | 中至高 | OSS 缺少与其他三者等价的标准远程控制面，Sidecar 只可做薄包装 | 中高 |
| Dify Community | 已发布 Workflow 的 Service API + SSE | 高 | 中 | Workflow 构建和深层状态查询不宜依赖未公开 Console API | 中高 |

表中接入方式适用于后续正式版。半月 MVP 不建设常驻远程 Adapter 服务：四个平台分别提供一个薄 CLI Runner，CrewAI 直接进程内运行，Dify 的 Workflow 人工导入并固定 DSL，Astra 和 LangGraph 使用现有公开 API。CrewAI AMP 可作为后续单独的 SaaS Track。

## 2. 比较对象的结构差异

### 2.1 Astra

Astra 的主要抽象是持久化 `Session -> Run -> Turn/Task/Checkpoint/Event`。多 Agent 通过 Parent Run 和 Child Run 的委派关系表达，SDK 已提供：

- 创建运行和查询运行状态；
- 暂停、恢复、取消和提交等待输入；
- 读取带游标的 Run 事件；
- 创建委派、读取 Child Run、暂停和恢复委派；
- 查询 Session 事件、因果链和运行投影；
- 读取 Token、工具调用和 Checkpoint 摘要。

这使 Astra Adapter 可以是薄客户端，不需要进入 Astra 内部数据库。权威接入证据见本地外部 checkout 中的 `astra/packages/sdk/README.md` 和 `astra/packages/sdk/src/client.ts`；正式冻结时以 `systems/manifest.yaml` 记录的 revision 为准。

需要注意：`astra/docs/design/` 中多个文档标记为 target design contract。Benchmark 能力覆盖只能依据固定版本实际 API、运行结果和故障实验，不能依据目标设计直接判定支持。

### 2.2 LangGraph

LangGraph 的核心抽象是带状态 Channel 的 Graph、Node、Edge、Super-step 和 Checkpoint。Graph 负责定义执行语义，Checkpointer 保存每一步状态；Subgraph 可表达子 Agent 或子流程。

用于 Benchmark 时应接入 Agent Server，而不是只在测试进程中调用 `graph.invoke()`。Agent Server 增加了 Assistant、Thread、Run、持久化、任务队列、取消和流式事件 API，使其与 Astra 的服务化运行边界更接近。

建议映射：

| Benchmark 概念 | LangGraph 概念 |
|---|---|
| Deployment | Graph + Assistant configuration |
| Session | Thread |
| Run | Thread Run |
| Agent/Child execution | Subgraph namespace or child traced run |
| State snapshot | StateSnapshot / Checkpoint |
| Pause/Resume | Interrupt + Command(resume=...) |
| Cancel | Run cancel, terminal status `interrupted` |
| Event stream | Run stream / graph stream events |

LangGraph Adapter 应通过 `langgraph-sdk` 使用公开 API，不直接读取 Checkpointer 数据库。若某些父子关系只能从 LangSmith Trace 得到，应将其标记为可选证据，不能让商业观测服务成为主榜必需条件。

### 2.3 CrewAI

CrewAI 有两层不同抽象：

- Crew 层：Agent、Task、Crew 和 Process，强调角色分工、委派和自治协作；
- Flow 层：Start、Listen、Router、State 和 Persistence，强调确定性控制流和持久化。

受控执行 Track 应主要使用 Flow 编排，并在 Flow 节点中调用 Crew；平台原生 Track 才允许纯 Crew 自主协作。否则使用纯 Crew 实现固定 DAG，会把模型规划波动错误归入 Runtime 指标。

CrewAI OSS 没有与 Astra、LangGraph Agent Server、Dify Service API 完全等价的标准服务控制面。因此需要一个随平台制品一起发布的 Python Sidecar：

- Sidecar 只负责接收统一运行请求、调用 `kickoff_async` 或 Flow、维护原生 Run ID 映射并转发事件；
- Agent、Task、Flow、重试、持久化和恢复必须由 CrewAI 原生配置承担；
- Sidecar 不得调度业务节点、重试 CrewAI 失败任务或补写平台状态；
- Sidecar 自身的 CPU、内存、时延和失败必须计入 CrewAI 实施及运行成本。

若使用 CrewAI AMP，则可以直接使用 Kickoff、Status、Resume 和 webhook/streaming API，但 AMP 无法与本地自托管环境进行同边界的进程 Crash 和资源限制实验，应单独报告。

### 2.4 Dify

Dify 的核心抽象是已发布的 App、Workflow/Chatflow、Node、Variable、Workflow Run 和日志。与代码框架不同，Adapter 不应在每次运行时动态创建 Workflow；应在构建阶段导入或人工审核固定 DSL，发布后记录 App ID、DSL 摘要和 API Key 引用。

建议映射：

| Benchmark 概念 | Dify 概念 |
|---|---|
| Deployment | Published Workflow/Chatflow version |
| Session | End-user ID；Chatflow 可额外映射 Conversation |
| Run | Workflow Run ID + Task ID |
| Step | Workflow Node execution event |
| Parallel branch | Parallel node / parallel iteration branch |
| Pause/Resume | Human Input pause + form submission |
| Cancel | Streaming task stop API |
| Event stream | Workflow SSE events，可从状态快照重新接流 |

Dify Adapter 可以使用公开 Service API 启动、查询、停止和重新接入事件流。深度审计只采用公开 SSE、运行详情、Workflow 日志和 OpenTelemetry；不把 Console 私有 API 或直接查数据库作为默认评分依赖。直接查库仅可用于诊断附录。

## 3. 统一测评系统边界

```text
Task Catalog + Agent Blueprint
              |
              v
       Benchmark Orchestrator
              |
   +----------+----------+----------+----------+
   |          |          |          |          |
 Astra     LangGraph   CrewAI      Dify        |
 Adapter    Adapter    Sidecar     Adapter      |
   |          |          |          |          |
   +----------+----------+----------+----------+
              |
      Canonical Event Collector
              |
   +----------+-----------+------------------+
   |                      |                  |
Native platform      Tool/Model Gateway   Fault/Resource
evidence             ground truth         ground truth
   |                      |                  |
   +----------------------+------------------+
                          |
                 Oracle + Metrics + Report
```

Benchmark Orchestrator 只控制 Case 生命周期，不执行平台内部业务编排。统一工具网关、模型代理和故障注入器属于 Benchmark 环境，它们提供独立于平台自报数据的 Ground Truth。

## 4. 正式版 Adapter 合约

本节是正式版目标接口，半月 MVP 不要求完整实现暂停、恢复、取消、事件流和部署自动化。MVP 使用第 8 节定义的文件级 Runner 合约，避免把一次对比实验做成平台工程项目。

建议将 Adapter SPI 分为必选控制面与可选能力面。

```python
class BenchmarkAdapter(Protocol):
    async def describe(self) -> PlatformManifest: ...
    async def health(self) -> HealthStatus: ...
    async def start(self, request: RunRequest) -> RunHandle: ...
    async def status(self, handle: RunHandle) -> RunSnapshot: ...
    async def events(self, handle: RunHandle, cursor: str | None) -> EventPage: ...
    async def signal(self, handle: RunHandle, signal: ControlSignal) -> SignalResult: ...
    async def artifacts(self, handle: RunHandle) -> ArtifactBundle: ...
```

必选动作：

- `start`
- `status`
- `events`
- `cancel`
- `artifacts`

可选动作通过 Capability Manifest 声明：

- `pause_resume`
- `human_input`
- `checkpoint_list`
- `checkpoint_resume`
- `child_runs`
- `native_tool_events`
- `native_model_usage`
- `event_replay`
- `deployment_import`

Adapter 遇到平台不支持的动作必须返回结构化 `UnsupportedCapability`，不能模拟成功。

## 5. Canonical 状态和事件

### 5.1 状态

统一状态只保留跨平台可证明的最小集合：

```text
PENDING
RUNNING
WAITING
PAUSED
SUCCEEDED
PARTIAL
FAILED
CANCELLED
UNKNOWN
```

每条状态记录同时保留：

- `canonical_status`
- `native_status`
- `mapping_rule_version`
- `observed_at`
- `source`

`WAITING` 必须携带结构化 `reason.kind`，例如 `human_input`、`external_dependency`、`capacity` 或 `permission`。`blocked`、`interrupted`、`partial-succeeded` 等平台特有状态不能丢失，放入 `native_status` 和结构化 `reason`。Dify 的 `partial-succeeded` 映射为 `PARTIAL`，最终任务是否通过仍由 Oracle 判定。

### 5.2 操作级映射

| 操作 | Astra | LangGraph Agent Server | CrewAI OSS | Dify Community |
|---|---|---|---|---|
| Start | `createRun` | create Thread Run | Sidecar 调用 Flow/Crew kickoff | `POST /workflows/run` |
| Status | `getRunStatus` | Runs get/join | Sidecar Run registry + native terminal event | `GET /workflows/run/{id}` |
| Events | Run SSE + Session events | Run/graph stream | Event Listener / streaming contract | Workflow SSE，可重新接流 |
| Cancel | `cancelRun` | Runs cancel | 若无原生传播则标记 adapter-mediated | Streaming task stop |
| Pause | `pauseRun` | Graph interrupt | 仅平台原生 HITL/Flow 能力计分 | Human Input pause，不等价于任意暂停 |
| Resume | `resumeRun` / `submitRunInput` | `Command(resume=...)` | Flow/HITL 原生恢复；Sidecar 重启不算原生 | Human Input form submission |
| State | Run projection / events | Thread state / checkpoints | Flow state / persistence | Run detail / state snapshot event |
| Children | Delegation list / child runs | Subgraph namespace / traced child run | Agent/Task/Crew execution | Node、iteration branch 或 sub-workflow |

每个 `SignalResult` 应包含 `implementation_origin`：

```text
platform_native
official_extension
adapter_runtime
unsupported
```

`adapter_runtime` 可以用于保证测试系统能够清理进程，但不能作为平台原生取消、暂停或恢复能力计分。

### 5.3 事件

```json
{
  "schema_version": "1.0",
  "benchmark_run_id": "...",
  "platform": "astra",
  "native_run_id": "...",
  "event_id": "...",
  "native_event_id": "...",
  "parent_event_id": "...",
  "sequence": 12,
  "occurred_at": "...",
  "observed_at": "...",
  "type": "tool.completed",
  "source": "platform_native",
  "evidence_level": "direct",
  "payload": {},
  "raw_artifact_ref": "..."
}
```

`evidence_level` 取值：

- `direct`：平台公开事件或 Benchmark 网关直接观察；
- `derived`：由多个直接事件确定性计算；
- `inferred`：根据日志文本或时序推断。

硬性 Oracle 只能使用 `direct` 或有公开算法的 `derived` 证据。`inferred` 仅用于诊断，避免某个平台因为日志文本更丰富而获得不公平优势。

核心事件族：

```text
run.created / run.started / run.waiting / run.finished
agent.started / agent.finished
step.started / step.finished
model.started / model.finished
tool.requested / tool.admitted / tool.dispatched / tool.finished
checkpoint.created / checkpoint.restored
control.requested / control.applied
fault.injected / fault.cleared
security.denied / security.violated
```

## 6. Ground Truth 设计

平台原生事件不能作为全部指标的唯一事实来源。建议设置三个 Benchmark 自有观察点：

1. Tool/MCP Gateway：记录工具 Schema、请求参数、准入、分发、结果、超时、幂等键和副作用计数。
2. Model Proxy：记录请求模型、参数、Token、缓存、首 Token 和完整响应时延。
3. Fault/Resource Observer：记录容器 Crash、网络扰动、CPU、内存、队列和进程状态。

这样即使 Dify 或 CrewAI 无法导出与 Astra 同粒度的 Tool Event，也仍能公平计算工具成功率和副作用；平台事件的缺失则单独进入“可观测性覆盖”指标。

## 7. 各 Track 的接入可行性

| Track | Astra | LangGraph | CrewAI OSS | Dify CE |
|---|---|---|---|---|
| 单 Agent 基线 | 高 | 高 | 高 | 高 |
| 固定顺序/分支 | 高 | 高 | 高，使用 Flow | 高 |
| 并行 fanout/fanin | 高 | 高 | 中高，Flow + Crew | 高，受节点能力约束 |
| Parent/Child 多 Agent | 高 | 高，Subgraph/child trace | 高，Crew Agent/Task | 中，通常表现为节点或子工作流 |
| Pause/HITL | 高 | 高 | 中高 | 高 |
| Checkpoint/Crash 恢复 | 高，需实测 | 高 | 中，依赖 Flow 持久化和 Sidecar 恢复边界 | 中高，需自托管实测 |
| 取消传播 | 高，需实测 | 高，需定义 Subgraph 行为 | 中 | 中高 |
| 深度审计 | 高 | 高 | 中高 | 中 |
| 权限/多租户 | 高，需实测 | 取决于 Agent Server Auth 实现 | OSS 需宿主系统提供 | 取决于 Dify 工作区和应用边界 |
| 进程故障注入 | 高 | 高 | 高，但包含 Sidecar | 高 |

表中的“高”只表示接口和实验设计可实现，不表示产品已经通过对应能力测试。

## 8. 半月 MVP 实施策略

### 8.1 最小系统边界

MVP 不实现通用 `BenchmarkAdapter` 服务、Canonical Event Store、CrewAI Sidecar、自动部署和跨平台暂停/恢复。每个平台只实现同一个命令行入口：

```text
runner --task task.json --output runs/<run_id>/
```

输入至少包含 `case_id`、`run_id`、模型配置引用、Tool Gateway 地址、随机种子和超时；输出一个 `result.json`，至少包含平台版本、原生 Run ID、`succeeded | failed | timeout | unsupported` 状态、起止时间、最终结构化结果、错误信息和原始产物路径。

统一 Tool Gateway 是主 Ground Truth，记录每次工具调用的 Run ID、参数摘要、时间、返回状态和副作用计数。MVP 直接汇总 Runner 结果与 Gateway 日志，不建设流式事件归一化层。

### 8.2 首轮任务集

首轮只做 6 个任务模板，每个模板提供 2 个确定性输入实例：

1. 顺序多工具执行与结构化输出；
2. 条件分支与授权决策；
3. `fanout -> fanin` 并行汇总；
4. 执行者与审阅者的修订循环；
5. 工具首次返回超时或 `500` 后的错误处理；
6. 写操作响应丢失后的重复副作用检测。

12 个受控 Case 在四个平台各运行 3 次，共 `12 × 4 × 3 = 144` 次。另选 2 个代表任务允许各平台采用原生最佳实践实现，各运行 3 次，共 24 次。总计 168 次运行。

主结果只报告受控 Track。原生 Track 单列展示，不与受控 Track 合并成一个总分。

### 8.3 半月内暂缓的内容

- 容器 Crash、Checkpoint 恢复和深度父子事件归一化；
- 暂停/恢复、HITL 和取消传播的重复压力测试；
- 权限攻击、多租户隔离、负载和长稳测试；
- MCP 多传输协议兼容性和完整资源成本采集；
- 30 Case/360 次运行的原 MVP，以及 48 Case/960 次运行的正式版。

这些能力在首轮只进入“原生支持 / 不支持 / 未验证”的证据矩阵，不进入量化主榜。

### 8.4 十个工作日排期

| 日期 | 交付 |
|---|---|
| Day 1 | 冻结平台版本、模型、6 个任务模板、输入输出 Schema 和公平性规则 |
| Day 2 | 完成 Tool Gateway、6 个 Oracle 和基准数据夹具 |
| Day 3 | 工程师 A 完成 Astra Runner；工程师 B 完成 CrewAI Runner，不建设常驻 Sidecar |
| Day 4 | 工程师 A 完成 LangGraph Runner；工程师 B 完成 Dify Workflow 导入、版本固定和 Runner |
| Day 5-6 | 完成 6 个任务的四平台实现、Oracle 联调和语义等价性审查 |
| Day 7 | 四平台跑通 12 个 Case 的单次 Smoke Test，修复语义差异 |
| Day 8 | 执行 144 次受控运行 |
| Day 9 | 执行 24 次原生运行、定向重跑失败项并完成归因 |
| Day 10 | 冻结数据，生成对比报告、失败案例和复现说明 |

## 9. 工作量估计

以下估计针对第 8 节的半月 MVP，已包含 6 个任务模板、运行和报告：

| 工作项 | 估计 |
|---|---:|
| TaskSpec、Result Schema 和公平性规则 | 1-1.5 人日 |
| Tool Gateway、数据夹具和 6 个 Oracle | 2-2.5 人日 |
| Astra 与 LangGraph Runner | 2-2.5 人日 |
| CrewAI 与 Dify Runner | 2.5-3 人日 |
| 6 个任务的四平台实现与等价性审查 | 3-4 人日 |
| 运行、定向重跑和失败归因 | 1.5-2 人日 |
| 分析、图表和复现报告 | 2-2.5 人日 |
| 合计 | 14-18 人日 |

建议配置 2 名工程师并行执行，10 个工作日内留出 2-6 人日缓冲。若只有 1 名工程师，应删除原生 Track，并将受控任务缩减为 4 个模板、8 个 Case，共 `8 × 4 × 3 = 96` 次运行。

最大不确定性是 Dify 的 Workflow 建模和四平台任务语义等价性。Day 7 设置硬性范围门：任何单个平台仍未跑通的任务从四个平台同时移除，不能只降低该平台标准。

## 10. 公平性规则

- Runner 代码、平台实现和 Dify DSL 公开并进入版本控制；
- Runner 只负责启动和采集，不承担业务编排、重试或语义聚合；
- 四个平台使用相同 Case、模型版本、模型参数、工具语义、超时和故障触发规则；
- 原始输出、平台日志和 Tool Gateway 日志必须保留并关联到唯一 Run ID；
- 私有 API、直接查库和日志文本解析不得进入主榜硬指标；
- 平台所需的额外服务开销计入对应平台，不从时延和成本中扣除；
- 功能缺失记为 `Unsupported`，观测不到记为 `Not Testable`，两者不能混用；
- 平台自报 Token、工具数和时延必须与模型响应和 Tool Gateway 对账。

## 11. Go/No-Go 结论

本节描述正式版架构可行性。半月 MVP 的执行 Go/No-Go 点设在 Day 7：四个平台必须对同一 Case 产生可被 Oracle 判定的结果，并能与 Tool Gateway Run ID 对账；否则该 Case 应记录原因并从四个平台的正式运行集中同时移除。

### Go

- 四个平台可以共享同一 Run、Status、Event、Control 和 Artifact 合约；
- 统一 Tool/MCP Gateway 可以解决工具调用和副作用 Ground Truth；
- LangGraph、CrewAI Flow、Dify Workflow 和 Astra 编排都能承载第一批受控流程；
- 自托管形态允许执行资源控制和进程级故障注入。

### 有条件 Go

- CrewAI 必须明确测试 OSS 还是 AMP；本方案主榜使用 OSS + 薄 Sidecar；
- Dify 深度审计只使用公开事件和 OTel，若证据不足则报告可观测性缺口；
- Parent/Child Agent 不能按 Astra 数据结构强行归一，应以 Branch/Execution lineage 的平台中立概念评分；
- 恢复能力必须通过真实进程终止实验确认，文档声明不算通过。

### No-Go 条件

- Adapter 开始替平台调度业务节点或自动重试失败任务；
- 为获得指标而依赖某个平台未公开且不稳定的内部 API；
- 无法将 Tool/Model Gateway 调用与平台 Run 唯一关联；
- 不同平台使用不同模型、工具语义或故障触发边界；
- 将无法观测误记为执行成功。

## 12. 官方资料

资料检索日期：2026-07-21。正式实验应将以下页面对应到实际使用的发布版本或 commit。

- Astra：本地外部 checkout 中的 `astra/packages/sdk/README.md`、`astra/docs/design/runtime-lifecycle.md`、`astra/docs/design/multi-agent-runtime.md`；对应版本见 `systems/manifest.yaml`。
- LangGraph：[Overview](https://docs.langchain.com/oss/python/langgraph/overview)、[Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)、[Streaming](https://docs.langchain.com/oss/python/langgraph/streaming)、[Agent Server](https://docs.langchain.com/langsmith/agent-server)、[Runs](https://docs.langchain.com/langsmith/runs)、[MCP](https://docs.langchain.com/oss/python/langchain/mcp)。
- CrewAI：[Documentation index](https://docs.crewai.com/llms.txt)、[Introduction](https://docs.crewai.com/)、[CrewAI AMP](https://docs.crewai.com/enterprise/introduction)。
- Dify：[Official repository](https://github.com/langgenius/dify)、[Workflow logs](https://docs.dify.ai/api-reference/workflows/list-workflow-logs)、[Stream workflow events](https://docs.dify.ai/api-reference/chatflows/stream-workflow-events)、[Plugin types](https://docs.dify.ai/en/develop-plugin/getting-started/choose-plugin-type)、[Agent strategy plugin](https://docs.dify.ai/en/develop-plugin/dev-guides-and-walkthroughs/agent-strategy-plugin)。
