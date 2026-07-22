# Astra 公共基准可用性审查

日期：2026-07-22  
状态：一轮事实审查完成，尚未完成本地冻结与 Runner 验证  
范围：仅服务 Astra 与 Hermes、Goose 的产品级比较

## 1. 结论

没有一个现成公共基准能够直接证明 Astra 的“状态可维护、可控制”。公共基准最合适的角色是提供外部可比的任务结果，Astra 的核心主张仍需由平台中立的故障、权限、审计和并发恢复 Case 证伪。

首期决策如下：

| 基准 | 决策 | 在 Astra 计划中的角色 | 不能据此声称 |
|---|---|---|---|
| Terminal-Bench 2.1 | Pilot 条件纳入，8 题 | 终端任务外部可比性 | 会话可恢复、权限安全、外部副作用 exactly-once |
| SWE-bench-Live | Pilot 条件纳入，4 题 | 真实仓库问题修复 | 长期记忆正确、状态 Owner 唯一、Crash 后可继续 |
| MCP-Universe v1.1.3 | 独立专项门禁 | MCP 发现、组合、Schema 与结果执行 | 断线重连、响应未知时的幂等、租户隔离 |
| AgentDojo v1.2.2 | 独立安全门禁 | 间接提示注入下的效用与攻击成功 | Astra 云边链路、身份传播和权限作用域安全 |
| GAIA | MVP 暂缓 | 第二阶段通用助理 sanity check | 工程 Agent 的状态控制优势 |
| τ³-bench v1.0.1 候选 | 第二阶段条件纳入 | 澄清、政策遵循、交互式状态变更 | 进程恢复和耐久 Checkpoint |
| DeepPlanning v1.1 | 保留既定的 P1 适配门禁 | 动态规划诊断 | 持久化、安全、审计和故障恢复 |

“条件纳入”表示完成版本、任务、容器、评分器与三个产品 Runner 的冻结门槛后才可计分，不表示当前已可运行。

## 2. 审查标准

每个候选基准按以下问题审查：

1. 是否存在可锁定的任务、代码、评分器和环境版本；
2. Oracle 是否尽量是规则或执行结果，而非未经校准的 LLM 主观评分；
3. 三个产品是否可获得等价的任务信息、工具语义、审批路径和预算；
4. 基准是否真的测量目标构念，还是只与目标构念弱相关；
5. 网络、账号、动态网页、用户模拟器和隐藏答案会引入什么混杂；
6. 许可与访问条款是否允许保存、复现和分发所需工件。

证据优先级为：官方代码与版本化数据 > 官方论文与文档 > 项目发布说明。项目自己的排行榜数字只描述项目报告，不作为竞品优劣证据。

## 3. 分项审查

### 3.1 Terminal-Bench 2.1

官方发布说明称 2.1 修正了 2.0 的 89 个任务中的 28 个，涉及外部依赖漂移、资源不匹配和任务说明与测试不一致；但官方数据仓库 README 写的是 26 个任务被修改。这一冲突说明“Terminal-Bench 2.1”这个名字本身不是充分的复现标识，必须保存任务仓库 commit、Harbor 数据版本、镜像 digest 和任务清单。

适用性：

- 强：容器内终端操作、结果工件和任务测试；
- 中：长执行链中的错误恢复能力，但只有自然发生的工具错误；
- 弱：产品会话恢复、审批、跨租户隔离和外部副作用一致性。

首期使用 8 个分层任务，避免全部来自同一语言、资源级别或任务类型。MOI Pilot 的每题 3 次仅用于先导研究；官方仓库要求排行榜提交每题至少 5 次，因此 Pilot 结果不得写成官方排行榜可比结果。正式外部可比复跑再使用 `k=5`。

### 3.2 SWE-bench-Live

初始 Python 数据发布包含 1,319 个任务，论文中的 Lite 为 300 个。项目随后增加多语言 Linux 和 Windows 版本；官方仓库在 2026 年列出的规模分别为 743 个和 61 个任务。不同代际使用的执行脚本、语言、平台和任务构造不完全相同，不能把它们混成一个总体分数。

适用性：

- 强：理解真实 issue、浏览仓库、修改代码、通过 fail-to-pass 与 pass-to-pass 测试；
- 中：长上下文和多轮工具使用；
- 弱：状态所有权、Crash 恢复、权限边界和跨会话记忆。

首期必须从一个冻结发布和一个明确 split 中选择 4 题，并冻结基础 commit、测试 patch、镜像和评估代码。持续更新的 `full` split 不进入可复现 Pilot。若选择 Python、MultiLang 或 Windows，必须单独成层并使用其对应评估路径。

### 3.3 MCP-Universe v1.1.3

官方材料列出 231 个任务、6 个领域、11 个 MCP Server、133 个工具和 84 个独立 evaluator。评分包含格式、静态和动态执行式检查，并明确不以 LLM-as-judge 为主评分器。这使其适合做 MCP 能力专项。

主要问题是很多任务依赖真实服务、账号、API Key 和动态状态。成功率会同时受 Agent、服务可用性、鉴权、配额、网络和时间窗口影响。首期必须把本地或静态任务与 live-service 任务分榜，并记录每个服务的版本、权限、速率限制和采集时间。

官方任务改造为断线、超时、响应丢失或重复副作用后，得到的是 **MOI-derived** 故障 Case，不再是官方 MCP-Universe 分数。该派生层才可能触及 Astra 的状态控制主张。

### 3.4 AgentDojo

官方包的候选发布为 v0.1.35，而代码默认基准版本为 v1.2.2；两者是独立版本轴，必须同时冻结。论文发布快照含 97 个正常任务和 629 个安全测试 Case，但不能把论文数字直接当作当前 v1.2.2 的任务数，本地 checkout 后需要重新枚举。

AgentDojo 适合比较间接提示注入下的：

- clean utility；
- utility under attack；
- injection goal 是否达成。

这三者必须分别报告，不能与普通任务成功率平均成一个“安全总分”。它的环境是基准模拟环境，不能替代 Astra 特有的路径、身份、租户和子 Agent 权限继承测试。三个产品还必须使用同样的工具描述、恶意内容、审批交互、模型和重试预算；否则测到的是适配器或额外防御预算。

### 3.5 GAIA

论文报告 466 个问题，并包含网页、文件附件和多模态任务。数据访问需要接受 Hugging Face 的 gated 条款，数据卡未声明一个可直接据此再分发的标准开源许可证。公共验证答案可见，存在污染风险；测试答案隐藏，需要官方评分路径。

官方 scorer 不是一般意义上的严格 exact match，而是对答案归一化后做边界匹配，并对数值 token 做 bag-of-words 检查。因此必须锁 scorer commit，不能在报告中简写为“完全一致评分”。

GAIA 还强依赖实时网页、浏览器和附件能力，三个产品很难在首期获得真正相等的环境。它与 Astra 核心主张的距离较远，故从 MVP 暂缓；未来只作为通用助理 sanity check，且不进入 P0 总结。

### 3.6 τ³-bench

仓库已从 τ²-bench 演进为 τ³-bench。当前候选冻结点为 v1.0.1；项目说明该版本修订了 `banking_knowledge` 的评分，相关领域的旧结果不能直接与新版混用。仓库的任务和评分持续修复，所以最终任务数必须从冻结 checkout 枚举，不能引用旧论文数量替代。

它通过 LLM 用户模拟器与 Agent 交互，并按数据库状态、动作和任务结果评分，适合测量澄清、政策遵循和共享状态变更。但用户模拟器本身是第二个随机 Agent，会显著放大方差。

第二阶段只使用 airline、retail、telecom 的文字半双工任务；冻结用户模拟器模型、版本、Prompt、温度、Seed 策略和轮次预算。语音全双工与 `banking_knowledge` 暂不混入首个比较。它不直接制造进程 Crash，也不验证耐久恢复。

## 4. 与 Astra 核心主张的覆盖关系

| 基准 | G：状态可追溯/可版本化 | O：运行可观测/可干预 | P：权限/身份边界 | 任务结果外部可比 |
|---|---:|---:|---:|---:|
| Terminal-Bench 2.1 | 弱 | 弱 | 弱 | 强 |
| SWE-bench-Live | 弱 | 弱 | 弱 | 强 |
| MCP-Universe | 弱 | 中 | 弱 | 中 |
| AgentDojo | 弱 | 中 | 中，但不是产品基础设施边界 | 中 |
| GAIA | 弱 | 弱 | 弱 | 中 |
| τ³-bench | 弱 | 中 | 中，偏业务政策 | 中 |
| DeepPlanning | 弱 | 弱 | 弱 | 中 |

这张表直接否定一个危险推理：在上述公共榜单上任务成功率高，不能推出“状态更可维护、更可控制”。反过来，Astra 的状态机制更丰富，也不能推出任务结果更好。两类结果必须并列而非互相代替。

## 5. MVP 落地方式

### 5.1 主 Pilot

- 公开层保持 12 Case：Terminal-Bench 2.1 为 8 个，冻结版 SWE-bench-Live 为 4 个；
- 产品工程层保持 12 Case，独立覆盖会话/记忆、多 Agent、故障恢复、MCP、安全、审计；
- MCP-Universe、AgentDojo 和 τ³-bench 可提供任务与 Oracle 设计方法，但修改后的 Case 标为 MOI-derived；
- DeepPlanning 保持独立的 12 Case Runner 等价性门禁；
- GAIA 不进入 MVP。

### 5.2 专项扩展门禁

只有满足以下条件才新增专项计分：

1. 三个产品的输入、工具 Schema、结果语义、审批通道和预算已形成对照表；
2. 冻结版本能在离线 fixture 上重复评分；
3. live-service 失败可与产品失败分开归因；
4. 修改过的任务不冒充官方分数；
5. 安全结果、正常效用和恢复结果分别报告。

## 6. 尚未关闭的阻塞项

- 六个项目均尚未在本地保存 commit、任务清单和哈希；
- Terminal-Bench 2.1 的任务修改计数存在官方来源冲突；
- SWE-bench-Live 的首期 split 尚未决定；
- GAIA 的分发边界尚未完成许可审查；
- τ³-bench 的 v1.0.1 checkout、任务数和评分回放尚未完成；
- 三个产品的公共基准 Adapter 均未通过等价 Smoke Test。

因此，本文件只能支持“候选选择与实验设计”，不能支持任何产品排名。

## 7. 主要来源

- Terminal-Bench：[2.1 发布说明](https://www.tbench.ai/news/terminal-bench-2-1)、[2.1 数据仓库与提交规则](https://github.com/harbor-framework/terminal-bench-2-1)、[论文](https://arxiv.org/abs/2601.11868)
- SWE-bench-Live：[官方仓库](https://github.com/microsoft/SWE-bench-Live)、[论文](https://arxiv.org/abs/2505.23419)
- MCP-Universe：[官方仓库](https://github.com/SalesforceAIResearch/MCP-Universe)、[使用文档](https://mcp-universe.github.io/usage.html)、[任务说明](https://mcp-universe.github.io/tasks.html)、[论文](https://arxiv.org/abs/2508.14704)
- AgentDojo：[官方仓库](https://github.com/ethz-spylab/agentdojo)、[文档](https://agentdojo.spylab.ai/)、[论文](https://arxiv.org/abs/2406.13352)
- GAIA：[数据卡](https://huggingface.co/datasets/gaia-benchmark/GAIA/blob/main/README.md)、[官方 scorer](https://huggingface.co/spaces/gaia-benchmark/leaderboard/blob/main/scorer.py)、[论文](https://arxiv.org/abs/2311.12983)
- τ³-bench：[官方仓库](https://github.com/sierra-research/tau2-bench)、[τ² 论文](https://arxiv.org/abs/2506.07982)
- DeepPlanning 与 OpenViking：见 [deepplanning-and-openviking-usability.md](./deepplanning-and-openviking-usability.md)

## 8. AI 使用说明

本审查由 AI 辅助检索、归纳和交叉核对。版本、规模、许可和评分器描述优先取自项目官方仓库、文档与论文；存在冲突或尚未本地复核的内容已显式标记。正式执行前仍需由负责人完成版本冻结、许可确认和评分回放。
