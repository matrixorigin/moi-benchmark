# MOI Benchmark 五 Track 初步计划汇总

> 范围：Astra、Memory、RAG、NLP2SQL、文档解析
> 文档状态：初步规划汇总，具体配置和预算需在 Smoke 阶段后冻结
> 汇总日期：2026-07-22

## 1. 总体目标

本阶段围绕 MOI 的五项核心能力建立可复现的 Benchmark：

1. **Astra**：比较通用 Agent 的短任务基线、长任务稳定性、故障恢复和审计能力。
2. **Memory**：验证 Memoria 的长期记忆问答效果和 Git-like 记忆治理能力。
3. **RAG**：比较 MOI 与主流知识库产品的原生端到端问答质量。
4. **NLP2SQL**：比较各产品在真实业务表上的 SQL 生成、执行正确性和安全性。
5. **文档解析**：比较 MOI 与主流解析工具在公开数据集和行业文档上的解析质量。

首版对比对象统一如下：

| Track | 被评测对象 | 首版核心对比对象 | 候补或扩展对象 | 首版对比方式 |
|---|---|---|---|---|
| Astra | Astra | Hermes Agent、Goose | 暂无；其他 Agent 留待后续版本 | 三个产品在统一任务、工具、资源和故障条件下实跑，并分别报告 Native Product 与满足资格时的 Common Model 结果 |
| Memory | Memoria | Mem0、Zep | 暂无 | 实际运行 Memoria；Mem0、Zep 使用论文或官方公开结果作外部参考 |
| RAG | MOI RAG | Dify、FastGPT | RAGFlow | 各产品使用同一批原始文件独立建库并实际运行端到端问答 |
| NLP2SQL | MOI NLP2SQL | Wren AI Cloud、Chat2DB、XiYan GBI | Vanna OSS、DB-GPT | 核心对象先参加 Smoke，再从中选出 2～3 个与 MOI 进行正式实测；候补仅在核心对象无法有效接入时启用 |
| 文档解析 | MOI 解析后端 | MinerU、PaddleOCR/PP-StructureV3 | olmOCR、Marker | MOI 实跑公开集和私有集；竞品优先使用官方 API，时间不足时公开集可引用官方公开分数并披露限制 |

五个 Track 统一遵循以下原则：

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

### 2.6 测试集建设要求

五个 Track 中，只有公开 Benchmark 部分可以直接使用现成数据和 Golden；涉及 MOI 差异化能力或真实业务效果的部分，需要自行构建测试集。

| Track | 可直接使用的测试集 | 需要自行构建的测试集 | 首版规模 | Golden/标注要求 |
|---|---|---|---:|---|
| Astra | Terminal-Bench 2.1、SWE-bench-Live；DeepPlanning 仅作适配门禁 | 6 个长任务族的生命周期、无故障/单故障变体及故障夹具 | 12 个短任务、12 个长任务 Case | 公开题使用冻结官方评分器；长任务必须建设结果、状态、副作用、审计和权限 Oracle，并通过确定性参考实现及故障位置资格回放 |
| Memory | LoCoMo、LongMemEval-S | 快照与回滚、分支/Diff/Merge、矛盾检测与治理特性集 | 每类先建设一组覆盖正常、异常和边界条件的确定性 Case | 以预期状态、版本、来源、分支和检索结果作为程序化 Golden，优先使用确定性断言，不依赖 LLM Judge 代替状态验证 |
| RAG | 无直接满足当前端到端产品横评目标的现成测试集 | 基于当前 47 份原始文档构建开发集和隐藏测试集 | 150～300 题 | 每题包含问题、参考答案、Gold 文件及证据位置；采用去重、证据包含、答案支持性和泄漏检查等自动校验，不安排逐题人工审核 |
| NLP2SQL | 第一阶段不采用公开集 | 基于现有 MatrixOne 真实业务表构建私有测试集 | 50～60 题；现有约 10 题作为起点 | 每题编写并执行 Golden SQL，保存稳定的 Golden 结果；逐题人工验证业务口径，开发题和隐藏题按题族隔离 |
| 文档解析 | OmniDocBench v1.6 及其官方 Golden | 复杂行业文档私有 Case 集 | 20～50 个 Case | 可从 Parser 输出生成 Draft，但必须人工校正；Golden 覆盖文字、布局、表格、公式、图片/Caption、标题层级和阅读顺序 |

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
- Memory 三组差异化能力的状态型 Case 和程序化断言。

相关人天已计入后续工作量表。NLP2SQL 和私有解析仍需人工验证 Golden；RAG 改为自动生成和程序化校验，并在报告中披露由此带来的测试集质量风险。

## 3. 工作量预估

### 3.1 估算口径

- 人天按所有参与人员的实际投入合并计算，不区分主执行和配合角色。
- 不包含账号审批、网络开放、采购和跨团队排队等待时间。
- 本汇总按三个资源包分别封顶 15 人天：Astra；Memory + 文档解析；RAG + NLP2SQL。
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
| **合计** | **45** | Astra 15；Memory + 文档解析 15；RAG + NLP2SQL 15 |

### 3.3 排期建议

- 按一人等效投入串行完成：45 个工作日，约 9 个工作周。
- 三个资源包并行、每包一人等效投入：约 15 个工作日；账号审批、环境阻塞和长任务实际运行等待时间另计。
- Astra v0.3 原始完整方案为 20 个工作日、390 次产品运行和 60 次夹具回放；15 人天约束下，优先保证任务/Oracle、Runner、Smoke 和有效 Pilot，未达到原定重复次数时只报告 Pilot，不包装为正式版结果。
- 建议三条工作流并行：Astra；Memory + 文档解析；RAG + NLP2SQL。组内根据账号和数据准备情况穿插执行。

NLP2SQL 和私有解析 Golden 的人工验证仍是主要人力风险。RAG 不做逐题人工审核，因此其结果必须明确标注测试集由模型生成并经程序化规则校验。

## 4. 金额预算

### 4.1 预算范围

下表只计算 API、模型、产品订阅和失败重试等外部现金成本，不包含员工人力、已有 MOI 环境和公司内部服务器折旧。

| Track | 初步预算 | 依据与不确定性 |
|---|---:|---|
| Astra | **¥2,500～¥4,000** | 三个产品的短任务、长任务、Common Model 资格运行、Astra 消融和必要重试；15 人天内按实际准入规模控制调用量 |
| Memory | **¥700～¥1,400** | Memoria 的抽取/治理、Embedding、Reader、Judge 和必要重试；Mem0/Zep 不实际调用 |
| 文档解析 | **¥200～¥600** | MinerU 暂按免费额度，Paddle 按页计费，并为私有 Case 与失败重试预留额度 |
| RAG | **¥1,500～¥2,300** | 测试集生成、三个系统建库与问答、Embedding/rerank、自动 Judge 和重试 |
| NLP2SQL | **¥1,000～¥1,700** | 模型 API、Wren/Chat2DB/XiYan 可能的订阅或 credits，以及竞品 Smoke |
| **外部服务合计（含风险余量）** | **约 ¥5,900～¥10,000** | 总预算以 **¥10,000 为上限**；各 Track 可在总额内调剂，但不得同时按各自上限外追加准备金 |

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
