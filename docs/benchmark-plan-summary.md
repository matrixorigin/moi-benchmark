# MOI Benchmark 七 Track 初步计划汇总

> 范围：Astra、Memory、RAG、NLP2SQL、文档解析、文档信息提取、Morpheus
> 文档状态：初步规划汇总，具体配置和预算需在 Smoke 阶段后冻结
> 汇总日期：2026-07-22

## 1. 总体目标

本阶段围绕 MOI 的七项核心能力建立可复现的 Benchmark：

1. **Astra**：比较通用 Agent 的短任务基线、长任务稳定性、故障恢复和审计能力。
2. **Memory**：验证 Memoria 的长期记忆问答效果和 Git-like 记忆治理能力。
3. **RAG**：比较 MOI 与主流知识库产品的原生端到端问答质量。
4. **NLP2SQL**：比较各产品在真实业务表上的 SQL 生成、执行正确性和安全性。
5. **文档解析**：比较 MOI 与主流解析工具在公开数据集和行业文档上的解析质量。
6. **文档信息提取**：比较 MOI 与商业产品基于预定义业务 Schema 提取结构化字段的质量、效率和成本。
7. **Morpheus**：验证结构化任务上的 adapter 效果，以及训练、评估、回放、门控、发布和回滚的受控自优化闭环。

首版对比对象统一如下：

| Track | 被评测对象 | 首版核心对比对象 | 候补或扩展对象 | 首版对比方式 |
|---|---|---|---|---|
| Astra | Astra | Hermes Agent、Goose | 暂无；其他 Agent 留待后续版本 | 三个产品在统一任务、工具、资源和故障条件下实跑，并分别报告 Native Product 与满足资格时的 Common Model 结果 |
| Memory | Memoria | Mem0、Zep | 暂无 | 实际运行 Memoria；Mem0、Zep 使用论文或官方公开结果作外部参考 |
| RAG | MOI RAG | Dify、FastGPT | RAGFlow | 各产品使用同一批原始文件独立建库并实际运行端到端问答 |
| NLP2SQL | MOI NLP2SQL | Wren AI Cloud、Chat2DB、XiYan GBI | Vanna OSS、DB-GPT | 核心对象先参加 Smoke，再从中选出 2～3 个与 MOI 进行正式实测；候补仅在核心对象无法有效接入时启用 |
| 文档解析 | MOI 解析后端 | MinerU、PaddleOCR/PP-StructureV3 | olmOCR、Marker | MOI 实跑公开集和私有集；竞品优先使用官方 API，时间不足时公开集可引用官方公开分数并披露限制 |
| 文档信息提取 | MOI 文档信息提取 | LandingAI Agentic Document Extraction | 暂无 | 两个产品使用相同源文件、业务 Schema、Golden 和评分规则，分别从原始文档端到端实际运行 |
| Morpheus | Morpheus 自优化闭环 | Base、LoRA-SFT、LoRA-SFT + Replay/Gate 三组内部 Baseline | GRPO、多轮自进化、完整 long-horizon Agent 留待后续 | 在相同冻结数据和基础模型下比较 Base、adapter 与完整闭环，验证效果增益、退化拦截和回滚能力；不做外部产品排名 |

七个 Track 统一遵循以下原则：

- 区分本项目实测结果和竞品公开结果。
- 数据、配置、模型、版本、运行命令和原始结果可追溯。
- 先运行小规模 Smoke，再冻结正式配置和预算。
- 质量、性能、成本和失败案例分别报告，不强行合成单一总分。
- 私有数据必须完成权限、脱敏和外发安全确认。

## 2. 各 Track 评测计划

### 2.1 Astra

详细方案：[Astra Agent 产品 Benchmark 计划 v0.3](../astra/plans/drafts/v0.3.md)

#### 评测目标

- 验证 Astra 在通用短任务上的任务效果、成本和时延是否达到可接受基线。
- 比较 Astra、Hermes、Goose 在无故障长任务和单故障长任务中的完成稳定性、状态持续性及恢复能力。
- 检验 Astra Introspect/Reflect 是否带来可验证的过程增益，同时控制健康任务误干预和额外成本。

#### 数据与对象

- 公开短任务：Terminal-Bench 2.1 选取 8 题，SWE-bench-Live 选取 4 题，共 12 题。
- 自建长任务：6 个任务族，每个包含无故障和单故障变体，共 12 个 Case；覆盖仓库维护、数据处理、多 Agent、MCP、人工审批和长期研究。
- 独立门禁：DeepPlanning 12 Case 只验证适配语义，不进入持久化结论。

#### 对比对象与口径

- **被评测对象**：Astra。
- **核心对比对象**：Hermes Agent、Goose。
- **对比方式**：三者先完成等价 Runner Smoke，再在相同用户目标、输入、工具语义、资源上限、故障位置和 Oracle 下实跑。
- **结果分层**：Native Product 条件用于比较完整产品体验；只有三个产品均能稳定使用同一模型时才运行 Common Model 条件。
- **内部消融**：在预注册的长任务 Case 上比较 Astra 开启和关闭 Introspect/Reflect；消融结果不与跨产品主榜混合。

#### 主要指标

- 通用指标：严格成功率、部分成功率、重复运行稳定性、时延、Token/成本和工具执行结果。
- 长任务指标：恢复成功率、RTO、已完成步骤保留率、状态丢失、重复副作用和恢复后业务成功率。
- 专项指标：根因分类与恢复动作正确率、健康任务误干预率、审计事件 P/R、Parent/Child 一致性、MCP 可靠性及权限安全。

#### 主要交付

- 三个产品的 Runner、冻结配置、任务目录和统一事件输出。
- 12 个长任务 Case 的 Lifecycle、Fault Schedule、Ground Truth Event Log 和 Oracle Bundle。
- 短任务、无故障长任务、单故障长任务、能力覆盖矩阵和 Astra 消融报告。

### 2.2 Memory

详细方案：[Memoria Agent Memory 评测计划](../memoria/plans/drafts/v0.1.md)

#### 评测目标

- 测试 Memoria 在长期、多会话记忆场景中的端到端问答效果。
- 验证快照与回滚、分支/Diff/Merge、矛盾检测与记忆治理等核心差异化能力。

#### 数据与对象

- 公开数据集：LoCoMo 全量、LongMemEval-S 全量。
- 特性数据集：围绕版本操作、冲突处理和隔离治理构造确定性用例。
- 第一阶段只实际运行 Memoria；Mem0 和 Zep 使用论文或官方公开结果作为外部参考，不重新运行。

#### 对比对象与口径

- **被评测对象**：Memoria。
- **核心对比对象**：Mem0、Zep。
- **对比方式**：首版不调用 Mem0、Zep 服务，仅将其论文或官方公开成绩列为外部参考；Memoria 结果属于本项目实测，二者不表述为严格同条件排名。
- **差异化能力对比**：快照/回滚、分支/Diff/Merge、矛盾处理和隔离治理等特性以 Memoria 的确定性用例验证为主；若竞品没有对应公开能力或可执行接口，则标记为“未验证”，不直接判为零分。

#### 主要指标

- 公开集：QA 正确率、类别分数、检索与回答成功率、时延及 Token 成本。
- 特性集：快照/回滚正确率、Diff/Merge 正确率、冲突检测与治理正确率、状态一致性。

#### 主要交付

- LoCoMo 和 LongMemEval-S Runner、Adapter 与原始结果。
- 三组特性用例及程序化断言。
- 公开集成绩、特性能力报告和失败案例。

### 2.3 RAG

详细方案：[MOI RAG 端到端 Benchmark 计划](../rag/plans/drafts/v0.1.md)

#### 评测目标

使用同一批原始非结构化文件，由每个产品独立完成解析、分段、Embedding、索引、检索、重排和回答，比较完整产品管线的最终效果。

#### 数据与对象

- 首版语料：当前已整理的 47 份原始文档，正式运行前冻结清单和哈希。
- 测试集：自动生成并通过程序化校验的 150～300 条问题，划分开发集与隐藏测试集，不安排逐题人工审核。
- 第一阶段产品：MOI、Dify、FastGPT；RAGFlow 作为后续增强项。

#### 对比对象与口径

- **被评测对象**：MOI RAG。
- **核心对比对象**：Dify、FastGPT。
- **扩展对象**：RAGFlow，仅在首版完成后且时间允许时补充，不计入首版必交范围。
- **对比方式**：MOI、Dify、FastGPT 使用相同原始语料和同一测试集分别独立建库，保留各产品原生解析、分段、Embedding、检索、重排和生成链路，进行端到端产品实测。
- RAGAS、DeepEval 等仅作为评测工具，不属于本 Track 的竞品。

#### 主要指标

- 主指标：Answer Correctness、Faithfulness、Answer Completeness、无答案正确率、任务成功率。
- 诊断指标：Document/Chunk Recall@K、MRR、nDCG、Context Precision。
- 工程指标：建库成功率、建库时间、P50/P95 时延、查询成功率和单次查询成本。

#### 主要交付

- 冻结语料清单、开发集和隐藏测试集。
- 各产品版本与配置、统一调用 Adapter 和原始结果。
- 自动评分、少量争议样本抽查、必要分组结果和代表性失败案例。

### 2.4 NLP2SQL

详细方案：[MOI NLP2SQL 产品能力评测计划](../nlp2sql/plans/drafts/v0.1.md)

#### 评测目标

在同一套真实业务表和问题上，比较 MOI 与竞品能否生成语义正确、可执行且安全的 SQL，并返回与 Golden 等价的结果。

#### 数据与对象

- 第一阶段不全量迁移公开数据集，直接使用现有 MatrixOne 真实业务表构建私有测试集。
- 将现有约 10 条问题作为起点，扩充并冻结 50～60 条人工验真的测试题。
- 候选竞品优先 Wren AI、Chat2DB、XiYan；根据 Smoke 结果选择 2～3 个正式竞品。Vanna、DB-GPT 作为替补。

#### 对比对象与口径

- **被评测对象**：MOI NLP2SQL。
- **核心 Smoke 对象**：Wren AI Cloud、Chat2DB、XiYan GBI。
- **正式对比对象**：从上述三个核心对象中，根据数据库连通性、结果有效性和运行可复现性选出 2～3 个，与 MOI 完成 50～60 题正式实测。
- **候补对象**：Vanna OSS、DB-GPT；仅当核心对象无法形成至少两个有效竞品时启用。
- **不纳入本轮产品横评**：XiYan-SQL 模型本身；如需评估，应另建 Text-to-SQL 方法评测，不能与 XiYan GBI 产品结果混用。

#### 主要指标

- Execution Accuracy、First-pass Success、Final Success。
- 澄清成功率、正确拒答率和 Safe Task Success。
- SQL 可执行率、Schema Linking、业务口径、自动修复净增益、时延、Token 和单题成本。

#### 主要交付

- 数据快照、只读账号、Schema/业务字典和 50～60 条正式测试集。
- Golden SQL、Golden 执行结果及逐题人工验证记录。
- MOI 与 2～3 个竞品的结果、失败归因、成本和安全报告。

### 2.5 文档解析

详细方案：[MOI 文档解析 Benchmark 方案](../document-parsing/plans/drafts/v0.1.md)

#### 评测目标

同时验证 MOI 在公开 Benchmark 和真实行业特殊 Case 上的文档解析能力。

#### 数据与对象

- 公开集：OmniDocBench v1.6，共 1,651 个页面样本，使用官方 Golden 和 Evaluator。
- 私有集：首版建设 20～50 个复杂表格、跨页表、公式、图文 Caption、多栏、扫描件、标题层级和行业术语 Case。
- 竞品：MinerU、Paddle/PP-StructureV3，并视时间补充 olmOCR、Marker 等。
- 第一阶段优先使用竞品官方 API；是否同环境重跑竞品取决于时间。

#### 对比对象与口径

- **被评测对象**：MOI 解析后端。
- **核心对比对象**：MinerU、PaddleOCR/PP-StructureV3。
- **扩展对象**：olmOCR、Marker，时间允许时补充。
- **公开集对比**：MOI 在 OmniDocBench 上实际运行官方评分器；MinerU、PaddleOCR/PP-StructureV3 优先通过官方 API 同集实跑，时间不足时可引用版本和配置明确的官方公开成绩，但必须与实测结果分栏展示。
- **私有集对比**：MOI、MinerU、PaddleOCR/PP-StructureV3 对同一批私有 Case 实际运行，通过 Adapter 统一到内部 Parse Blocks 后评分；olmOCR、Marker 是否加入私有集取决于时间。

#### 主要指标

- 公开集：Text/Formula/Table/Reading Order Edit、Table TEDS、Formula CDM 和官方 Overall。
- 私有集：总体及各维度 P/R/F1、Case 级分数、失败率、耗时和资源成本。

#### 主要交付

- MOI 到 OmniDocBench Prediction 的 Adapter 和官方成绩。
- MinerU/Paddle 等到内部 Parse Blocks 的 Adapter。
- 人工审核后的私有 Golden、统一 Runner、评分结果和失败案例报告。

### 2.6 文档信息提取

详细方案：[MOI vs LandingAI 文档信息提取产品对比测评计划](../document-extracting/plans/drafts/v0.1.md)

#### 评测目标

- 在相同原始文档、预定义业务 Schema 和评分规则下，比较 MOI 与 LandingAI 的字段提取准确率、完整率和幻觉情况。
- 覆盖低质量收据、多模板表单和多页法律合同三类场景，分析两个产品的能力边界。
- 比较 Schema 适配、批处理、异常处理、结果溯源、时延、调用成本和人工操作成本。

#### 数据与对象

- SROIE：100 份单页扫描收据，4 个固定字段。
- VRDU Registration：100 份、约 202 页的多模板注册表单，6 个固定字段。
- Kleister-NDA：100 份、约 602 页的多页法律合同，4 个固定字段，其中 `party` 为数组。
- 正式集共 300 份、约 904 页；三个数据集分别冻结 Manifest、Schema、源文件哈希和 Golden Adapter。

#### 对比对象与口径

- **被评测对象**：MOI 文档信息提取。
- **核心对比对象**：LandingAI Agentic Document Extraction。
- **对比方式**：两个产品均从相同 PDF/JPG/PNG 原始文件开始，使用语义等价的预定义 Schema；产品输出 Adapter 只转换格式，不修复或补全结果。
- **运行口径**：先完成 9 份 Smoke 和 30 份 Pilot，再冻结配置运行 300 份正式集；字段错误不允许重试选优，主质量分采用第一次有效业务响应。
- **发布边界**：正式运行及公开发布前需确认 LandingAI 关于 benchmarking/competitive analysis 的合同权限；未确认时只进行内部小规模功能验证。

#### 主要指标

- 字段级 Precision/Recall/F1、Normalized Exact Match、文档全字段正确率、缺失字段正确率和幻觉率。
- Schema 合法率、API/任务成功率、逐字段及场景切片结果，并对产品差异进行配对分析。
- P50/P95 时延、总运行时间、每文档/页成本、Schema 配置和异常处理人工时间。

#### 主要交付

- 三个 100 Case Manifest、中立业务 Schema、Golden Adapter 和统一 Scorer。
- MOI、LandingAI Runner 与输出 Adapter，以及可追溯的原始响应、usage、日志和统一结果。
- 总体、逐数据集、逐字段、场景切片和一对一 Case 对比报告。

### 2.7 Morpheus

详细方案：[Morpheus 测评方案](../morpheus/plans/draft/morpheus-evaluation-plan.md)

#### 评测目标

- 验证 Morpheus 在结构化领域任务上训练出的 candidate adapter 是否相对 Base 和上一版 stable adapter 稳定提升。
- 验证训练、评估、回放、门控、发布和回滚链路能否稳定、可复现、可审计地执行。
- 检验 Replay/Gate 是否能拦截低质量或退化 adapter，并验证发布失败后的安全恢复能力。
- 首版只证明受控的 adapter 级自优化闭环，不宣称已实现长期自主自进化或完整 Agent long-horizon 能力提升。

#### 数据与对象

- `train`：按结构化任务准备训练样本；仅在本轮覆盖训练链路时必需。
- `holdout`：50～200 条，与训练集严格隔离，用于泛化和主效果评估。
- `regression`：20～100 条历史核心、关键业务和已知风险样本，用于检查旧能力退化。
- `replay`：20～100 条真实链路、高风险及 Hard Case，用于上线前风险验证。
- 可选扩展：`trace_transition_eval`、`hardcase` 和 `reward_preflight`，不作为首版主结果的强制前置条件。

#### 对比对象与口径

- **被评测对象**：Morpheus 自优化与发布治理闭环。
- **内部 Baseline**：Base、LoRA-SFT、LoRA-SFT + Replay/Gate、Self-Evolve v1。
- **对比方式**：固定基础模型、冻结数据快照和评分逻辑，对 Base、candidate adapter、stable adapter 运行相同 Holdout、Regression 和 Replay；Gate/Release/Rollback 使用同一阈值和风险规则。
- **外部竞品**：首版不设置外部产品竞品，因此结论只回答 Morpheus 相对内部 Baseline 的增益和治理价值，不表述为行业产品排名。
- **后续增强**：GRPO、多轮自进化、Trace 决策和完整 Agent 长任务属于扩展项，不作为当前八个主指标的必要条件。

#### 主要指标

- 功能指标：`overall_score`、`schema_compliance`、`field_accuracy/action_quality`、`regression_pass_rate`、`replay_pass_rate`、`gate_verdict`。
- 系统指标：`training_job_success_rate`、`rollback_success_rate`。
- 扩展过程指标：`self_evolution_loop_completion_rate`，只证明计划轮次中闭环是否完成，不单独证明效果持续提升。

#### 主要交付

- 冻结的 `morpheus-eval-v1` Manifest，以及 Train/Holdout/Regression/Replay 数据快照和隔离说明。
- Base、SFT、candidate、stable adapter 的离线评估结果和统一报告。
- Replay/Gate/Release/Rollback 演练记录、失败分类和发布决策证据。
- 数据、训练产物、adapter、评估、门控和发布结果可追溯的闭环报告。

### 2.8 测试集建设要求

七个 Track 中，只有公开 Benchmark 部分可以直接使用现成数据和 Golden；涉及 MOI 差异化能力或真实业务效果的部分，需要自行构建测试集。

| Track | 可直接使用的测试集 | 需要自行构建的测试集 | 首版规模 | Golden/标注要求 |
|---|---|---|---:|---|
| Astra | Terminal-Bench 2.1、SWE-bench-Live；DeepPlanning 仅作适配门禁 | 6 个长任务族的生命周期、无故障/单故障变体及故障夹具 | 12 个短任务、12 个长任务 Case | 公开题使用冻结官方评分器；长任务必须建设结果、状态、副作用、审计和权限 Oracle，并通过确定性参考实现及故障位置资格回放 |
| Memory | LoCoMo、LongMemEval-S | 快照与回滚、分支/Diff/Merge、矛盾检测与治理特性集 | 每类先建设一组覆盖正常、异常和边界条件的确定性 Case | 以预期状态、版本、来源、分支和检索结果作为程序化 Golden，优先使用确定性断言，不依赖 LLM Judge 代替状态验证 |
| RAG | 无直接满足当前端到端产品横评目标的现成测试集 | 基于当前 47 份原始文档构建开发集和隐藏测试集 | 150～300 题 | 每题包含问题、参考答案、Gold 文件及证据位置；采用去重、证据包含、答案支持性和泄漏检查等自动校验，不安排逐题人工审核 |
| NLP2SQL | 第一阶段不采用公开集 | 基于现有 MatrixOne 真实业务表构建私有测试集 | 50～60 题；现有约 10 题作为起点 | 每题编写并执行 Golden SQL，保存稳定的 Golden 结果；逐题人工验证业务口径，开发题和隐藏题按题族隔离 |
| 文档解析 | OmniDocBench v1.6 及其官方 Golden | 复杂行业文档私有 Case 集 | 20～50 个 Case | 可从 Parser 输出生成 Draft，但必须人工校正；Golden 覆盖文字、布局、表格、公式、图片/Caption、标题层级和阅读顺序 |
| 文档信息提取 | SROIE、VRDU Registration、Kleister-NDA 的公开源文件和标注 | 不新标注业务文档；需自行抽样并构建三个 100 Case Manifest、中立 Schema、Golden Adapter 和统一归一化规则 | 300 份、约 904 页 | Golden 来自公开数据集标注，经 Adapter 映射为固定字段 Schema；不得参考 MOI 或 LandingAI 输出修改 Golden，争议标注需单独留痕 |
| Morpheus | 暂无可直接替代当前结构化业务闭环目标的统一公开集；Train 可按任务选用公开或业务数据 | 必须冻结 Train、Holdout、Regression、Replay 及 Manifest；可选构建 Trace Transition 和 Reward Preflight | Holdout 50～200、Regression 20～100、Replay 20～100；Train 按任务决定 | 各 Split 去重并严格隔离，记录来源、时间窗口和哈希；结构化样本需包含输入、Expected Output 和必要的决策约束，正式结果不能使用临时 Demo 数据 |

#### 自建测试集通用流程

```text
冻结原始数据与版本
  -> 生成候选 Case / Draft Golden
  -> 去重、难度与能力分类
  -> 按 Track 要求执行自动校验或人工审核
  -> 划分开发集与隐藏测试集
  -> 计算哈希并冻结版本
  -> 正式评测
```
自建测试集是本项目主要工作量之一，尤其包括：

- Astra 的 12 个长任务 Case、生命周期脚本、单故障夹具、Ground Truth Event Log 和多类 Oracle。
- RAG 的 150～300 条问题、答案和证据自动生成及程序化校验。
- NLP2SQL 的 40～50 条新增问题及全部 Golden SQL 执行验证。
- 文档解析 20～50 个行业 Case 的元素级人工校正。
- 文档信息提取三个 100 Case Manifest、固定 Schema、Golden Adapter 和归一化规则；无需重新人工标注 300 份文档，但需处理公开 Golden 噪声和争议样本。
- Memory 三组差异化能力的状态型 Case 和程序化断言。
- Morpheus 的 Train/Holdout/Regression/Replay 数据快照、任务 Schema、Expected Output、风险标签和 Split 隔离检查。

相关人天已计入后续工作量表。NLP2SQL 和私有解析仍需人工验证 Golden；RAG 改为自动生成和程序化校验，并在报告中披露由此带来的测试集质量风险。

## 3. 工作量预估

### 3.1 估算口径

- 人天按所有参与人员的实际投入合并计算，不区分主执行和配合角色。
- 不包含账号审批、网络开放、采购和跨团队排队等待时间。
- 本汇总保留三个既定资源包各 15 人天：Astra；Memory + 文档解析；RAG + NLP2SQL。文档信息提取作为新增独立工作包，严格限制为 5 人天。
- Morpheus 详细计划未给出明确人天；总计划按首版离线评测加一次闭环演练预留 10 人天，完成 Smoke 后再冻结，不包含真实业务长期试点和产品化补强。
- 各 Track 详细计划中的原始工期和运行规模是完整方案参考；若超过本汇总的人天上限，应在 Smoke 后缩小首版范围并记录未完成项，不降低评分标准或混淆 Pilot 与正式结果。
- 不包含后续长期回归平台化和大规模性能压测。

### 3.2 分 Track 人天

| Track | 总人天 | 主要工作 |
|---|---:|---|
| Astra | **15** | 版本与范围冻结、短任务接入、长任务与 Oracle、三个 Runner、故障注入、Smoke/Pilot、结果与报告 |
| Memory | **8** | 复核已有实验、Runner/Adapter、两套公开集、差异化特性用例、运行与报告 |
| 文档解析 | **7** | 复用 OmniDocBench，完成 MOI Adapter、竞品 API、代表性私有 Case、评分与报告 |
| RAG | **8** | 语料冻结、测试题自动生成与校验、三个产品建库、运行、自动评分与报告 |
| NLP2SQL | **7** | 数据与权限、私有 Golden、竞品 Smoke 与筛选、正式运行、复核与报告 |
| 文档信息提取 | **5** | 300 Case Manifest、Schema 与 Golden Adapter、两个产品 Runner、统一 Scorer、Smoke/Pilot、正式运行、分层复核与报告 |
| Morpheus | **10** | 数据快照与隔离、Base/SFT/Candidate 评估、Replay/Gate、一次 Release/Rollback 演练、结果汇总与报告 |
| **合计** | **60** | Astra 15；Memory + 文档解析 15；RAG + NLP2SQL 15；文档信息提取 5；Morpheus 10 |

### 3.3 排期建议

- 按一人等效投入串行完成：60 个工作日，约 12 个工作周。
- 分两个阶段执行时约 25 个工作日：第一阶段四个资源包并行，最长约 15 个工作日；第一阶段全部完成后，再串行执行 10 个工作日的 Morpheus 评测。账号/条款审批、环境阻塞、训练和实际运行等待时间另计。
- Astra v0.3 原始完整方案为 20 个工作日、390 次产品运行和 60 次夹具回放；15 人天约束下，优先保证任务/Oracle、Runner、Smoke 和有效 Pilot，未达到原定重复次数时只报告 Pilot，不包装为正式版结果。
- 第一阶段并行执行四条工作流：Astra；Memory + 文档解析；RAG + NLP2SQL；文档信息提取。组内根据账号、数据和算力准备情况穿插执行。
- 第二阶段仅执行 Morpheus：必须在第一阶段其他评测全部完成、结果和资源释放后启动，不与其他 Track 并行。

文档信息提取的 5 天依次用于数据与 Golden、两个 Runner、Scorer 与 Pilot、正式运行、分层复核与报告。首版不增加数据集或竞品；分歧 Case 过多时采用分层抽样复核，不能通过降低 Golden 校验、Smoke 或评分标准压缩时间。

NLP2SQL 和私有解析 Golden 的人工验证仍是主要人力风险。RAG 不做逐题人工审核，因此其结果必须明确标注测试集由模型生成并经程序化规则校验。

## 4. 金额预算

### 4.1 预算范围

下表只计算 API、模型、产品订阅和失败重试等外部现金成本，不包含员工人力、已有 MOI 环境和公司内部服务器折旧。

| Track | 初步预算 | 依据与不确定性 |
|---|---:|---|
| Astra | **¥2,400～¥3,600** | 三个产品的短任务、长任务、Common Model 资格运行、Astra 消融和必要重试；15 人天内按实际准入规模控制调用量 |
| Memory | **¥700～¥1,300** | Memoria 的抽取/治理、Embedding、Reader、Judge 和必要重试；Mem0/Zep 不实际调用 |
| 文档解析 | **¥200～¥500** | MinerU 暂按免费额度，Paddle 按页计费，并为私有 Case 与失败重试预留额度 |
| RAG | **¥1,400～¥2,200** | 测试集生成、三个系统建库与问答、Embedding/rerank、自动 Judge 和重试 |
| NLP2SQL | **¥900～¥1,600** | 模型 API、Wren/Chat2DB/XiYan 可能的订阅或 credits，以及竞品 Smoke |
| 文档信息提取 | **¥200～¥600** | LandingAI 2,000～2,500 credits 约 20～25 美元，扣除可能的免费额度后充值更低；另预留 MOI 调用和基础设施重试成本 |
| Morpheus | **¥0～¥200** | 首版复用现有 RTX 4090、数据库、对象存储和业务/Trace 数据，仅为必要的外部调用或小额辅助服务预留 |
| **外部服务合计（含风险余量）** | **约 ¥5,800～¥10,000** | 新增 Morpheus 后总预算仍以 **¥10,000 为上限**；各 Track 可在总额内调剂，但不得同时按各自上限外追加准备金 |

预算表已经包含必要重试和风险余量，不再额外叠加统一准备金。实际报销和报告使用账单发生日汇率。

### 4.2 不包含项目

- 员工与领域专家人力成本。
- 新购 GPU 或本地大模型部署成本。
- 私有数据脱敏、法务与安全审查成本。
- 后续持续回归、监控平台和生产级并发压测。
- 竞品价格或模型价格变化导致的差额。

所有 Track 在正式全量运行前必须设置 API 预算上限和告警，并保留实际 Token、页数、credits 和账单记录。最终报告使用实际金额替换估算值。

## 5. 服务器资源预算

### 5.1 总体策略

首版优先复用现有 MOI、MatrixOne 和开发机环境，竞品尽量使用官方 Cloud/API，统一使用外部 LLM/Embedding API。因此首版原则上**不需要 GPU**。Astra 需要额外保证三个产品运行环境、工作区、状态目录和账号彼此隔离。

### 5.2 分 Track 需求

| Track | 最小资源 | GPU |
|---|---|---|
| Astra | 1 台 16 核 CPU、32 GB RAM、200 GB 空闲磁盘的 Runner/容器宿主机，或 3 个等价隔离环境；另需统一 Tool Gateway、故障注入和结果存储 | 使用外部模型 API 时不需要 |
| Memory | 当前 M3、8 核 CPU、16 GB RAM；保留 30～40 GB 空闲磁盘 | 外部模型 API 时不需要 |
| RAG | 现有 MOI 环境；Dify/FastGPT 可先用已有或共享环境 | 外部模型/API 时不需要 |
| NLP2SQL | 现有 MOI/MatrixOne、Cloud 网页版和桌面客户端 | 不需要 |
| 文档解析 | API Runner 只需普通执行机和额外 4～10 GB 磁盘 | 不需要 |
| 文档信息提取 | 普通 API Runner；建议 8 核 CPU、16 GB RAM、20 GB 空闲磁盘，用于约 904 页文件、双产品原始响应和评分结果 | MOI 与 LandingAI 均走现有服务/API 时不需要 |
| Morpheus | 复用现有训练机、MatrixOne/SQLite、MinIO/对象存储和日志存储；首版优先 1.5B/3B LoRA/QLoRA，训练与 vLLM 推理错峰运行 | 需要 1 张 RTX 4090 24GB；不安排 13B+ 全参训练或大规模 GRPO |
