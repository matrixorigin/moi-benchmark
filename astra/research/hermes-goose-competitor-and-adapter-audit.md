# Hermes 与 Goose 竞品及 Runner 审查

日期：2026-07-22  
状态：Hermes 完成本地代码一轮审查；Goose 仅完成官方 Web 证据审查  
范围：仅用于 Astra 产品评测设计，不构成产品质量结论

## 1. 核心结论

Hermes 和 Goose 都已经具备会话、工具扩展、子 Agent、审批或安全模式等状态控制要素。因此，Astra 不能把“有持久状态”“能恢复会话”或“有 Checkpoint”本身当作独占优势。

更严格、可证伪的主张应是：

> 在相同任务、模型、工具与故障条件下，Astra 能否比 Hermes 和 Goose 更完整地暴露状态归属与迁移、允许安全干预，并在 Crash、重试、并发恢复和权限攻击后保持可解释的一致结果。

当前只证明了三者值得进入同一比较，没有证明 Astra 更好。

## 2. 冻结状态

| 产品 | 候选版本 | 证据状态 | 能否计分 |
|---|---|---|---|
| Astra | `9bf87cbede29bfaa7f12ebec978a71a4c9e7ec84` | 本地 checkout，但工作树有修改 | 否，需决定干净构建来源并完成 Smoke |
| Hermes | `f4df260f26c93f15694698869f3ea8e965eea301`，描述符 `v2026.7.20-63-gf4df260f2` | 本地干净 checkout | 否，需锁依赖、配置和 Runner |
| Goose | 发布候选 `v1.43.0` / `5a9eb7edea1e081e2d54473ae41481f0289b826a` | 官方发布页 Web 核对；本地浅克隆超时 | 否，必须本地复核并完成 Adapter Smoke |

Goose 的 commit 目前只是候选 pin，不是“已冻结系统”。任何评分前必须能从本地 checkout 复现构建、配置、会话和审批行为。

## 3. 状态能力不能只看功能名

| 可观察事实 | Hermes | Goose | 对 Astra 主张的影响 |
|---|---|---|---|
| 会话保存与重新载入 | SQLite 为规范 transcript store；有 `resume_pending` 与 `suspended` 生命周期 | ACP/REST 提供 session new/load/prompt/cancel；发布说明含 session edit/fork 与重连改进 | “可恢复会话”不是 Astra 独有，需要比较恢复后状态正确性 |
| 文件回滚 | 每轮可创建 shadow-git 文件 Checkpoint，并支持 rollback | 需本地审计后确认产品级能力 | 文件回滚不等于工具调用、数据库和外部副作用回滚 |
| 审批/权限 | 正常产品路径有审批；one-shot 会自动绕过 | `GOOSE_MODE` 支持 auto/approve/chat/smart_approve；默认 smart_approve | 安全赛道必须固定真实审批路径，不能用便捷 headless 默认 |
| 多 Agent | Agent/Subagent、delegate 等原生能力 | Subagent 与扩展作用域 | 并发上限、模型、工具与子 Agent 预算必须一致记录 |
| MCP/扩展 | MCP catalog 与技能/工具体系 | 70+ MCP extensions 的官方产品声明，ACP 支持 MCP | 扩展数量不是能力质量；比较同一工具 Schema 与失败语义 |
| 可观测性 | session/turn/tool/approval/subagent hooks | 每消息 usage/cost 与 session total 等发布能力 | 需要比较事件完整度、因果关联和故障后证据，而非仅是否有日志 |

### 3.1 Hermes 的已核对边界

本地 `docs/session-lifecycle.md` 显示：

- SQLite `SessionDB` 是 transcript 的规范存储；`sessions.json` 主要保存路由和运行时元数据；
- `resume_pending` 用于软恢复并保留现有 session id；`suspended` 是硬重置标志；
- 非正常关闭后可标记最近活跃会话，下一次访问继续同一 transcript。

这证明了“会话级连续性设计”，但没有证明被中断的工具调用可以 exactly-once 续接。恢复测试仍需分别注入：调用前 Crash、调用后响应前 Crash、响应已丢失、两个恢复者竞争等位置。

本地 `tools/checkpoint_manager.py` 实现了 shadow-git 文件快照，每轮同一目录至多自动建立一次，并限制最多扫描 50,000 个文件。它排除或限制部分文件类型和大文件。其能力边界是文件系统快照；数据库事务、远端 API、消息发送和 MCP Server 内部状态不在该 Checkpoint 的自然覆盖范围内。

最关键的公平性风险在 `hermes_cli/oneshot.py`：文件明确设置 `HERMES_YOLO_MODE=1` 与 `HERMES_ACCEPT_HOOKS=1`，自动绕过危险命令与 Hook 审批。因此：

- one-shot 只可用于不测审批与权限的普通 headless 任务，并在报告中披露；
- 安全、权限、取消和恢复 Case 必须走能呈现和记录审批的 Gateway/交互式产品路径；
- 不能把 one-shot 的自动通过当作 Hermes 的安全失败，也不能让它获得无审批摩擦的性能优势后与其他产品混榜。

### 3.2 Goose 的已核对边界

以下仅来自 Goose 官方仓库、配置文档和发布说明，尚未通过本地代码复核：

- 产品提供 Desktop、CLI、REST/ACP 等接口；ACP 暴露 session new/load/prompt/cancel 和 permission request，适合作为自动化 Runner 的候选接口；
- `GOOSE_MODE` 可配置 auto、approve、chat、smart_approve，官方文档默认值为 smart_approve；
- context strategy、最大轮次、扩展、Sandbox 和数据根目录均会改变结果；`GOOSE_PATH_ROOT` 可用于逐运行隔离状态；
- v1.43.0 发布说明包含 ACP 会话重连、session edit/fork、usage/cost 统计和审批期间的 compaction 处理等改动；这些是供应方能力声明，不是质量证据；
- 仓库已有 Harbor Adapter，但默认指向旧的 Terminal-Bench 2、单次 trial 和自报结果工作流，不能直接作为 MOI 的 2.1 Pilot 配置。

安全与恢复 Case 优先使用 ACP，因为它能表示 permission、cancel 和 session load。若本地验证发现 ACP 与 Desktop/CLI 的安全边界不同，需要把它作为 Runner 效应报告，而不能默认等价。

## 4. Runner 公平性设计

### 4.1 不使用一个接口覆盖所有 Track

| Track | Astra | Hermes | Goose | 必须保持相等的语义 |
|---|---|---|---|---|
| A 原生产品 | 原生推荐接口 | 原生推荐接口 | 原生推荐接口 | 任务与 Oracle 相同；允许默认产品策略，但完整披露 |
| B 受控模型/工具 | 可脚本化产品接口 | 普通任务可用 one-shot；安全任务禁用 | ACP/REST 候选 | 模型、Prompt、工具 Schema、预算、超时、重试 |
| C 故障/安全 | 能观测审批、取消、恢复的接口 | Gateway/交互路径，禁止 one-shot 自动放行 | ACP permission/cancel/session load | 注入位置、审批策略、身份、状态目录、恢复竞争条件 |

薄 Runner 只能做协议翻译、启动隔离、事件采集和结果提取。它不得：

- 替产品补造不存在的持久化、编排、重试或审批；
- 把某一产品的错误静默重试而不给其他产品相同预算；
- 将“进程退出但远端操作可能成功”直接标成安全可重试；
- 以适配器的共享记忆替代产品原生状态，又声称测得原生能力。

### 4.2 必须冻结的运行参数

每个结果记录至少包含：

- 产品 commit/tag、构建产物哈希、依赖锁与配置哈希；
- provider、精确模型版本、系统 Prompt 和上下文压缩策略；
- 工具/MCP Server 版本、Schema 哈希、权限与网络策略；
- 最大轮次、并行度、子 Agent 上限、重试、超时和 Token/费用预算；
- 审批模式、Sandbox、状态根目录和是否使用产品原生记忆；
- Session id、父子运行 id、每个工具调用 id 与副作用 id；
- 注入故障的位置、时刻、恢复者数量和最终 Oracle 证据。

## 5. 两种公平模式必须分榜

### 5.1 原生能力榜

允许产品使用自己的记忆、Checkpoint、子 Agent 和默认控制策略，回答“用户拿到该产品会获得什么”。缺点是差异无法单纯归因于状态架构。

### 5.2 受控机制榜

尽可能统一模型、工具、预算和任务状态后，禁用非共同的外部帮助，回答“控制变量后机制是否改变结果”。某产品确实不支持的能力标为 `Unsupported` 或 `Not Testable`，不能由 Runner 补齐，也不能从能力矩阵中删去。

两个榜都不产生跨 Track 总分。否则原生便利性与机制效果会被混为一谈。

## 6. 对“Astra 状态优势”的最强反驳

在实验前应主动承认以下替代解释：

1. Astra 只是暴露了更多状态对象，增加了复杂度，却没有提高最终正确率或恢复成功率；
2. Hermes 的 transcript resume 与文件 Checkpoint，或 Goose 的 session load/fork/reconnect，已经覆盖了用户真正需要的恢复场景；
3. Astra 的优势只在自家特制故障 Case 中出现，在 Terminal-Bench 和 SWE-bench-Live 等真实任务中没有收益；
4. 所谓“可控制”实际上来自更频繁的人工审批，代价是更慢、更贵、更多误阻断；
5. 观察到的差异来自 Runner、模型支持、默认重试或上下文压缩，而不是状态架构；
6. 状态可见不等于状态可信：事件缺失、因果关系错误或审计无法复演时，更多日志可能只是噪声。

因此 P0 Case 必须同时测量结果正确性、恢复一致性、重复副作用、状态 Owner 冲突、审计完整度、人工介入量、时延和成本。只有 Astra 在这些权衡上形成稳定、可重复的净收益，才能支持“状态更可维护、可控制”。

## 7. 计分前门槛

三个产品逐项达到以下状态后才能计分：

1. 冻结版本可从干净环境构建；
2. 状态目录可逐运行隔离并能在重启后重新加载；
3. 普通、审批、取消、Crash 和恢复各有至少一个 Smoke；
4. 工具输入、输出和副作用 id 能完整采集；
5. 自动重试和上下文压缩不会绕过故障注入器；
6. `Supported / Partial / Unsupported / Not Testable` 已由日志或代码证据支撑。

Goose 当前停在第 1 项之前；Hermes 和 Astra 也尚未完成全部门槛。因此本审查不提供能力胜负表。

## 8. 主要来源

- Hermes 本地证据：`external/hermes-agent/docs/session-lifecycle.md`、`external/hermes-agent/tools/checkpoint_manager.py`、`external/hermes-agent/hermes_cli/oneshot.py`，commit `f4df260f26c93f15694698869f3ea8e965eea301`
- Hermes 官方仓库：[NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
- Goose 官方仓库：[aaif-goose/goose](https://github.com/aaif-goose/goose)
- Goose 候选发布：[v1.43.0](https://github.com/aaif-goose/goose/releases/tag/v1.43.0)
- Goose 配置说明：[Environment Variables](https://block.github.io/goose/docs/guides/environment-variables/)
- Goose 定制分发与 ACP/REST：[CUSTOM_DISTROS.md](https://github.com/aaif-goose/goose/blob/main/CUSTOM_DISTROS.md)
- Goose Harbor Adapter：[evals/harbor](https://github.com/aaif-goose/goose/tree/main/evals/harbor)

## 9. AI 使用说明

本审查由 AI 辅助完成代码检索、官方资料归纳和反证设计。Hermes 结论基于指定本地 commit；Goose 因 checkout 超时，仅视为官方 Web 证据，所有实现层判断均保留为待本地验证项。供应方功能描述未被当作独立性能证据。
