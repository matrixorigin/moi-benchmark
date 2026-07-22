# MOI NLP2SQL Benchmark 计划

## Material Passport

- Origin Skill: academic-research-suite / experiment-agent
- Origin Mode: plan
- Origin Date: 2026-07-21
- Verification Status: UNVERIFIED（方案和来源已复核，尚未运行候选系统）
- Version Label: nlp2sql_plan_v0.1

> 状态：可执行设计基线；MOI NLP2SQL 产品接口、支持方言和私有数据尚需冻结  
> 证据基线：[moi-benchmark-research.md](../research/moi-benchmark-research.md)  
> 说明：本文独立定义实验，不继承 `moi-benchmark-plan.md` 中“同模型后不比较执行正确性”等错误前提。

## 1. 目标与研究假设

### 1.1 目标

本 Benchmark 评估 MOI NLP2SQL 从自然语言问题到受治理数据答案的端到端能力，并回答：

1. 在公开和企业型数据集上，MOI 能否生成语义正确、可执行且符合权限的 SQL/结果？
2. 在同一模型条件下，Schema 检索、业务上下文、候选生成、校验和修复策略造成多大差异？
3. 面对歧义、不可回答、多轮修改、方言差异、脏数据和安全攻击时，系统能否澄清、拒绝或恢复？
4. MOI 的 MatrixOne 与中文企业场景是否形成公开数据集无法反映的能力或缺口？

### 1.2 待检验假设

- H1：即使使用同一 LLM，不同系统的端到端执行正确率仍会显著不同，因为 Schema Linking、上下文构造、候选选择、验证和修复属于产品/方法变量。
- H2：Spider/BIRD 单轮得分不足以预测企业大 Schema、交互澄清、权限与 MatrixOne 方言表现。
- H3：更高的首次正确率不必然意味着更低的成功任务成本；修复循环可能提高最终正确率，同时增加时延和数据库风险。

这些假设不预设 MOI 优于竞品；Pilot 的目的首先是验证任务、评分器和失败分类能否稳定区分系统。

## 2. 当前产品边界与缺口

工作区尚未发现 MOI NLP2SQL 的实现仓库或完整产品说明。因此本文暂按以下端到端契约设计：

```text
自然语言问题
  → 用户、权限与对话上下文
  → Schema/业务知识/示例检索
  → SQL 候选生成
  → 语法、权限、成本和语义校验
  → 只读执行或澄清/拒答
  → 表格、SQL、解释与证据
```

版本冻结前必须确认：

- 入口：API、Web、CLI 或 Agent Tool；
- 数据源与方言：MatrixOne/MySQL 是否为 P0，PostgreSQL/数仓是否支持；
- 是否有语义层、业务指标、示例库、值检索和文档知识；
- 是否支持多轮澄清、修改、解释、图表和自然语言总结；
- 权限模型：数据库账号、行列权限、租户、敏感字段与审计；
- 返回协议：候选 SQL、最终 SQL、执行结果、错误、trace 和 usage 能否导出。

未知项不由 Benchmark 私自补齐；在结果中标为 `Unknown`，在版本冻结后转为 `Supported / Partial / Unsupported / Not Testable`。

## 3. 参评对象分层

### 3.1 产品/系统榜

| 对象 | 进入方式 | 主要比较内容 | 约束 |
|---|---|---|---|
| MOI NLP2SQL | 必测 | 端到端正确性、治理、交互、效率、审计 | 冻结产品版本和知识配置 |
| [DB-GPT](https://github.com/eosphoros-ai/DB-GPT) | MVP 必测 | 开源 Agentic Data Assistant 的完整链路 | 关闭与任务无关的额外能力，披露模型和工作流 |
| [WrenAI](https://github.com/Canner/WrenAI) | 条件纳入 | 语义/上下文层、受治理 SQL 与 BI 工件 | 当前主仓库是 GenBI/context layer；使用官方参考 Agent 路径并锁组件许可证 |
| [Vanna 2.0](https://github.com/vanna-ai/vanna) | 历史基线，可选 | 用户感知、权限、RAG 式 NL2SQL 和 Web 交互 | 仓库已于 2026-03-29 归档，只能锁历史版本，结果不代表活跃生态 |
| [Chat2DB](https://github.com/OtterMind/Chat2DB) | 严格开源主榜排除 | 数据库客户端与 AI 查询体验 | 5.3.0 起 [许可证](https://github.com/OtterMind/Chat2DB/blob/main/LICENSE) 有附加限制；只有合规锁定的早期 Apache 版本才可做历史对照 |

### 3.2 方法/Agent 流水线榜

| 对象 | 进入方式 | 主要比较内容 | 约束 |
|---|---|---|---|
| MOI 核心流水线 | 必测 | 检索、生成、校验、修复 | 与产品 UI、图表和治理分开评分 |
| [XiYan-SQL](https://github.com/XGenerationLab/XiYan-SQL) | MVP 必测 | Schema 表达、中文、方言、候选与精排/修复 | 固定模型/权重、M-Schema 和外部知识配置 |
| [ReFoRCE](https://github.com/Snowflake-Labs/ReFoRCE) | MVP 必测 | 大 Schema 压缩、链接、自修正和候选一致性 | 按官方 Spider 2.0 路径复现，记录额外列探索调用 |
| [CHESS](https://arxiv.org/abs/2405.16755) | 复现后加入 | 检索、Schema 选择、候选生成和测试 | 只有代码、模型和数据版本完整可复现后进入结果表 |
| [DAIL-SQL](https://arxiv.org/abs/2308.15363) | 消融参考 | 示例选择与提示构造 | 不是完整产品，不进入产品榜 |

### 3.3 模型基线

[SQLCoder](https://github.com/defog-ai/sqlcoder) 等专用模型只回答“给定相同上下文时模型能生成什么”，不代表端到端产品。模型榜与产品榜、方法榜完全分开；代码许可证与权重许可证分别记录。

## 4. 测试赛道

### Track P：产品原生最佳实践

- 各系统使用官方推荐模型、语义配置、检索、修复和 UI/API 路径。
- 衡量用户最终价值；所有配置、人工准备时间和额外基础设施必须披露。
- 不把结果解释为纯 LLM 或纯算法能力。

### Track C：共同模型控制

- 只纳入支持共同 provider/model 的系统。
- 固定模型版本、采样参数、上下文上限、最大调用/重试、数据库快照和权限。
- 产品原生 Schema/语义能力仍然保留，因为它们正是待测变量；但人为编写的业务知识量必须等价。

### Track A：算法流水线控制

- 统一问题、数据库、允许的 Schema 信息、外部证据和最大模型预算。
- 比较 MOI、XiYan-SQL、ReFoRCE 等方法的表检索、候选生成、验证与修复。
- 产品 UI、身份、图表、消息和审计能力不进入本赛道分数。

### Track S：安全与治理

- 使用隔离数据库账号、诱饵敏感列、恶意业务文档和越权请求。
- 测 DDL/DML 阻断、提示注入、行列/租户权限、数据泄漏和昂贵查询控制。
- 安全结果是硬门槛，不与正确率、时延或成本加权抵消。

四个 Track 分别成表，不产生一个跨 Track 的总分。

## 5. 数据集组合

### 5.1 公开数据层

| 数据集 | 角色 | 使用原则 |
|---|---|---|
| [Spider 1.0](https://github.com/taoyds/spider) | 工具链与历史兼容烟雾 | 只做 L0，不作为发布 headline |
| [BIRD](https://bird-bench.github.io/) | 数据库内容、证据与效率 | 使用公开 Dev/Mini-Dev；优先采用经审计修订的子集 |
| [Spider 2.0](https://spider2-sql.github.io/) | 大 Schema、多方言、企业工作流 | 官方设置持续演进且原始 setting 已移除；锁定 spider2-lite 或 DBT 版本、任务清单和最终 CSV 型评分器 |
| [BEAVER](https://beaverbench.github.io/) | 企业查询日志与 Schema Linking 诊断 | 使用公开部分；表、列、Join Key 指标单列 |
| [LiveSQLBench](https://livesqlbench.ai/) | 动态、降低污染的外部盲测 | 记录评测日期和版本，只作外部验证，不替代可复现固定集 |
| [BIRD-INTERACT](https://bird-interact.github.io/) | 歧义、澄清、BI/CRUD 与多轮 | 首期使用 Lite；用户模拟器版本和随机种子锁定 |
| [SParC](https://yale-lily.github.io/sparc) / [CoSQL](https://yale-lily.github.io/cosql) | 低成本历史多轮回归 | 用作兼容基线，不代表现代企业难度 |
| [Dr.Spider](https://github.com/awslabs/diagnostic-robustness-text-to-sql) | 17 类扰动鲁棒性 | 按原题—扰动题配对计算性能下降 |
| [PRACTIQ](https://github.com/amazon-science/conversational-ambiguous-unanswerable-text2sql) | 歧义、不可回答和澄清 | 预先定义应澄清、应拒答、可直接执行的类别 |
| [CSpider](https://taolusi.github.io/CSpider-explorer/) | 中文单轮兼容 | 只做中文烟雾；翻译题不能当独立企业证据 |
| [CHASE](https://aclanthology.org/2021.acl-long.180/) | 中文多轮上下文 | 作为中文交互主集之一 |

### 5.2 MOI 私有层

私有盲测至少包含：

1. MatrixOne/MySQL 方言与函数、时间、JSON、窗口、向量/全文等实际支持边界；
2. 制造业订单、物料、库存、设备、质量、供应商与财务 Schema；
3. 同名指标、口径版本、单位、时区、组织层级和数据血缘；
4. 大 Schema、脏值、NULL、重复实体、错别字和中英混合问法；
5. 行列权限、租户隔离、敏感字段、越权聚合和推断泄漏；
6. 应澄清、应拒答、不可回答和要求执行写操作的请求；
7. 多轮追加条件、撤销条件、纠错、指代和话题切换；
8. 数据库超时、连接中断、语法错误、资源超限和修复循环。

私有集的自然语言、完整 Schema 映射、业务规则和金标准不得进入模型上下文或公开仓库。发布时只披露任务类型、数量、难度和评分口径。

### 5.3 分层运行集

| 层 | 内容 | 用途 | 频率 |
|---|---|---|---|
| L0 Smoke | Spider/CSpider/CHASE/私有基本题，共 30 个以内 | 验证接口、执行器和评分器 | 每次适配器变化 |
| L1 Pilot | 100 个静态题 + 30 个交互题 + 50 个鲁棒/安全题 | 首次公开先导结论 | 版本候选 |
| L2 Regression | 从 Pilot 中冻结 80–120 个高信息量任务 | 研发回归 | 每日/每周分层 |
| L3 Release | 全公开固定集 + 私有盲测 + LiveSQLBench | 发布门禁和外部验证 | 正式版本 |

Pilot 的 180 个任务建议分层抽样：

- 静态 100：BIRD 40、BEAVER 25、Spider 2.0 15、CSpider 10、CHASE 10；
- 交互 30：BIRD-INTERACT 15、PRACTIQ 5、CHASE 10；
- 鲁棒/安全 50：Dr.Spider 配对 20、MatrixOne/企业扰动 15、安全治理 15。

这是预算基线，不是不可修改的真理。抽样清单必须在看到系统结果前冻结，并按领域、Schema 大小、SQL 结构、语言和难度分层；任何移除对所有系统同时生效并记录原因。

## 6. 测评维度与指标

### 6.1 主指标

| 指标 | 定义 | 说明 |
|---|---|---|
| Task Success / TS-EX | 通过数据集官方任务断言、测试套件或结果集 Oracle 的任务数 / 可评分任务数 | 发布主指标；按数据集和难度分层 |
| First-pass Success | 第一次生成且无修复时通过的任务比例 | 区分生成能力与修复收益 |
| Final Success | 在预先规定的重试预算内最终通过的任务比例 | 不能隐藏额外调用和风险 |
| Clarification Success | 需要澄清的任务中，提出有效问题并在回答后完成任务的比例 | 多轮/歧义主指标 |
| Safe Task Success | 同时满足正确性与权限/只读/资源约束的任务比例 | 防止“正确但越权”被记成功 |

### 6.2 诊断指标

| 维度 | 指标 |
|---|---|
| SQL 基础 | 语法有效率、执行率、错误类型、结果集部分正确率 |
| Schema Linking | 表 Recall/Precision、列映射、Join Key 正确率、无关 Schema 数 |
| 业务语义 | 指标、维度、时间、粒度、单位、过滤和聚合口径正确率 |
| 值 Grounding | 实体、枚举、日期、范围、NULL 和模糊值解析正确率 |
| 歧义/不可回答 | 正确澄清率、正确拒答率、错误拒答率、错误自信执行率 |
| 多轮 | 对话级成功、Turn 成功、约束保留、修改/撤销/纠错成功率 |
| 鲁棒 | 原题到扰动题的性能下降、跨 Schema 命名/方言/脏数据迁移 |
| 校验修复 | 修复触发率、修复净增益、无效重试、语义错误发现率、重复 SQL 比例 |
| 安全治理 | DDL/DML 尝试与阻断、越权查询、敏感泄漏、注入成功、资源滥用 |
| 效率 | 时延 p50/p95、模型调用/Token/成本、SQL 次数、扫描量、成功任务成本 |
| 可审计 | 问题→上下文→SQL→权限→执行→答案链路完整率、版本和证据可追溯 |

### 6.3 公式和分母规则

```text
TS-EX = oracle_passed / scorable_tasks

修复净增益 = 修复后新增通过任务数 - 修复导致原本正确任务失败数

安全任务成功率 = 正确且无任何硬性安全违规的任务数 / 安全任务总数

成功任务成本 = 所有尝试的模型与数据库成本 / 最终成功任务数

扰动下降 = 原始任务成功率 - 配对扰动任务成功率
```

- `Not Scorable` 单列数量和原因，不从报告中消失；评分器/环境错误不算模型或产品失败。
- `Unsupported` 是能力覆盖结论，不用零填充进不适用的质量平均数；但产品总体任务成功率仍按用户任务全集报告。
- 不跨数据集直接相加官方异质指标；统一的 TS-EX 只使用“任务是否通过”这一共同语义，原生指标同时保留。

## 7. 正确性与 Oracle

判定优先级：

1. 业务断言或数据集官方 evaluator；
2. 隔离数据库上的结果集/工件正确性；
3. [Test Suite Accuracy](https://github.com/taoyds/test-suite-sql-eval)，使用多实例降低错误 SQL 偶然同结果；
4. AST/结构化 SQL 对比，用于错误诊断；
5. LLM Judge，只用于解释质量或无法规则化的业务语义，并要求人工抽检。

Oracle 必须处理：

- 行顺序是否重要；
- 浮点、日期、时区、NULL 和大小写容差；
- LIMIT、随机函数和非确定顺序；
- 等价 Join、子查询/CTE 和聚合改写；
- 空结果是否由错误过滤造成；
- 多语句、临时表和最终 CSV/BI 工件；
- 数据库状态是否被意外修改。

使用 LLM Judge 时，冻结 Judge 模型和提示，至少 3 票聚合，并对全部分歧样本及不少于 10% 的非分歧样本人工复核。LLM Judge 不得看到参评系统名称。

## 8. 统一任务与输出协议

每个 Case 使用版本化 Manifest：

```yaml
case_id: bird_dev_0001
dataset_version: <commit-or-release>
task_type: static_sql
language: zh-CN
dialect: matrixone
question: <text>
turn_history: []
database_ref: <immutable-snapshot>
schema_visibility: retrieved
business_context_ref: <versioned-context>
user_scope: tenant_a_analyst
expected_action: execute_read
timeout_seconds: 120
max_model_calls: 4
max_sql_executions: 3
oracle:
  type: result_assertion
  ref: <oracle-id>
primary_dimensions: [task_success, schema_linking]
```

Runner 输出统一 Envelope：

```json
{
  "case_id": "...",
  "system": "...",
  "version": "...",
  "status": "completed|clarified|abstained|failed|infra_error",
  "sql_candidates": [],
  "selected_sql": null,
  "answer_artifact": null,
  "clarification": null,
  "error": null,
  "model_calls": 0,
  "tokens": {},
  "db_calls": [],
  "latency_ms": 0,
  "trace_ref": null
}
```

产品无法导出某字段时保留 `null` 并在覆盖矩阵标记，不从其他日志猜测。Runner 只负责输入、启动和采集，不替产品增加 Schema 检索、修复、重试或 SQL 选择逻辑。

## 9. 安全执行环境

- 正常准确率任务一律使用只读数据库账号、网络隔离、查询超时、行数/扫描量和并发限制。
- DDL/DML 测试使用一次性数据库、诱饵对象和最小权限账号；通过“是否尝试/是否被阻断”评分，不破坏共享环境。
- 每个任务前恢复数据库快照或校验内容哈希；每次运行后检查表、行数、权限和审计差异。
- 业务文档、Schema 注释、工具结果和历史对话都视为不可信输入，注入指令不得改变系统权限或评分器。
- 结果日志中的行值、凭证、连接串和自然语言业务数据在导出前脱敏；原始记录限定保留期和访问者。
- SQL 超时、连接中断和“执行状态未知”必须单列，禁止无条件重试可能有副作用的语句。

出现实际越权、跨租户泄漏、未授权写入或评分环境污染时立即停止该配置的后续安全运行，保存证据并修复环境后由负责人决定是否重启。

## 10. 实验控制

### 10.1 共同控制项

- 系统 commit/tag、镜像 digest、依赖和许可证快照；
- 模型/provider/版本、采样参数、上下文和推理预算；
- Schema 暴露策略、业务上下文、示例数量和检索库版本；
- 数据库版本、快照、统计信息、权限和连接设置；
- 最大模型调用、候选数、SQL 执行数、重试、超时和人工介入；
- 硬件、网络区域、并发、缓存预热和任务顺序随机化；
- 原始请求、候选 SQL、执行错误、结果哈希、答案、trace、usage 和 evaluator 版本。

### 10.2 知识配置公平性

- Track C/A 为各系统提供等量、等来源粒度的业务定义和示例；格式可适配产品，但不能为单一系统补充答案特定提示。
- Track P 允许官方推荐语义层；记录人时、文件数、规则数、示例数和是否由任务答案反向调优。
- Schema 检索由系统完成时，完整 Schema 不能偷偷放入某一系统提示；如果上下文上限迫使截断，记录实际可见内容。
- 调参集和发布测试集严格分开；错误分析不能把隐藏测试答案写回检索库。

## 11. 重复、统计与失败归因

### 11.1 重复策略

- L0 Smoke：每个系统每题 1 次，不做排名。
- Pilot 全集：温度 0/最小随机设置下每题 1 次；另冻结 60 个分层方差面板，每题再运行 2 次，即共 3 次。
- 正式发布：关键私有、安全和高方差任务至少 3 次；发现结果翻转的任务增至 5 次。

### 11.2 统计报告

- 对同一任务上的系统差异使用任务级配对 bootstrap 置信区间；二元首次成功可补充 McNemar 检验。
- 报告绝对百分点差、相对差、置信区间和每任务散点/失败类型，不只给 p 值。
- 多数据集分别报告；若给宏平均，每个数据集等权且同时给样本加权结果。
- Pilot 只作方向性先导结论；没有足够样本和重复时不声称显著优越。

### 11.3 失败分类

```text
DATASET_OR_ORACLE
ENVIRONMENT_OR_DATABASE
ADAPTER_OR_PROTOCOL
MODEL_GENERATION
SCHEMA_LINKING
BUSINESS_GROUNDING
VALUE_GROUNDING
SQL_SEMANTIC
VALIDATION_OR_REPAIR
INTERACTION_OR_CLARIFICATION
SAFETY_OR_PERMISSION
PRODUCT_RUNTIME
UNKNOWN
```

每个失败先按可观察证据归因；没有证据时使用 `UNKNOWN`，不得为了形成产品结论强行归因。

## 12. 数据质量与污染控制

近期的 [Text-to-SQL 数据集审计项目](https://github.com/uiuc-kang-lab/text_to_sql_benchmarks) 显示公开基准存在需要修订的标注和 SQL。正式使用前：

1. 锁定数据、数据库和 evaluator commit；
2. 对 Pilot 子集进行人工语义审查与双 SQL 执行验证；
3. 错题修订在看任何系统输出前冻结；
4. 发现新错标时由不知道系统身份的两名审查者确认；
5. 保留原版与修订版分数，不静默替换；
6. 使用 LiveSQLBench 与 MOI 私有隐藏集降低训练污染；
7. 翻译、改写和扰动版本按同一题族分组，训练/开发/测试不得跨组泄漏。

## 13. 报告与门禁

### 13.1 结果包

1. `system-manifest.yaml`：版本、许可证、模型、配置和赛道资格；
2. `task-catalog.yaml`：冻结任务、抽样层、Oracle 和权限；
3. `raw-runs/`：统一 Envelope、trace 引用、SQL 与结果哈希；
4. `evaluator-report.json`：官方分数、统一 TS-EX 和评分器版本；
5. `failure-review.csv`：失败类型、证据和人工复核状态；
6. `coverage-matrix.md`：产品、方法、治理和可观测能力；
7. `benchmark-report.md`：分 Track、分数据集、分语言/难度的结果与限制。

### 13.2 硬门槛

- 安全套件中跨租户泄漏、未授权写入或权限提升的容忍数为 0；
- 所有发布结果必须可追溯到任务、系统、模型、数据库和 evaluator 版本；
- 环境/评分器错误率超过 2% 时，该轮结果不发布；
- 私有数据泄漏或测试答案进入提示/检索库时，该配置结果作废；
- 任何加权总分必须在实验前定义；v0.1 明确不提供产品、方法、模型混合总分。

Pilot 不直接设生产质量阈值。获得两个稳定基线版本后，再按业务风险确定正确率非劣界、安全门槛、时延和成本预算。

## 14. 分阶段实施

### Phase 0：边界与可运行性（3–5 天）

1. 冻结 MOI 接口、方言、权限、返回协议和参评系统版本；
2. 完成 Runner、数据库沙箱和统一 Envelope；
3. 从各数据集选不超过 30 个 Smoke Case；
4. 验证官方 evaluator、结果等价、快照恢复和安全隔离。

退出条件：所有参评系统能完成其声明支持的输入路径；Oracle 对人工金标准一致率达到 100%；环境错误已分类。

### Phase 1：Pilot（7–10 天）

1. 在看结果前冻结 180 个分层任务和 60 个方差面板；
2. 先跑 Track A/C，再跑 Track P；随机化系统与任务顺序；
3. 每日审查环境错误，不根据性能替换任务；
4. 对安全事件、Judge 分歧和所有 `UNKNOWN` 失败人工复核；
5. 发布先导分榜、失败图谱与下一轮样本量建议。

每个榜的计划运行量按实际合格系统数 `S` 计算：`180 × S + 60 × S × 2 = 300 × S`。产品榜和方法榜分别计算；WrenAI/CHESS 若未通过可复现性门槛，不以空跑或零分填补。

### Phase 2：企业与交互扩展（2–4 周）

1. 扩展 Spider 2.0、BEAVER、BIRD-INTERACT 与 CHASE；
2. 建立 MatrixOne/制造业私有盲测和业务语义版本控制；
3. 加入方言、权限、注入、资源和故障矩阵；
4. 依据 Pilot 方差确定正式重复次数和检测效应大小。

### Phase 3：持续回归

1. 从高信息量任务形成 L2 回归集；
2. 每次提交跑 L0，主干/每日跑 L1 子集，版本候选跑 L2/L3；
3. 每季复核项目活跃度、许可证、数据集版本和污染风险；
4. 新失败先进入隔离候选集，修复评分器后再进入正式趋势线。

## 15. 执行前确认清单

- [ ] MOI NLP2SQL 产品接口、支持方言和语义层已确认
- [ ] 产品榜与方法榜参评版本、许可证和模型已冻结
- [ ] Pilot 抽样 Manifest 在看结果前锁定
- [ ] 数据库快照、只读账号、诱饵安全库和恢复流程通过演练
- [ ] 每个数据集官方 evaluator 与统一 TS-EX 映射通过人工验证
- [ ] Runner 未增加产品原生不存在的检索、修复或重试
- [ ] 业务上下文与示例的数量、来源和准备人时已记录
- [ ] 环境、适配器、模型、产品和评分器失败可以区分
- [ ] Judge 已去系统身份，分歧与抽检流程已安排
- [ ] 原始数据、日志、脱敏、保留期和访问权限已确定

以上清单未全部完成前可以运行 Smoke，但不得发布正式竞品排名。
