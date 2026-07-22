# MOI Benchmark

MOI Benchmark 用于持续、可复现地评估 MOI 的产品能力，并与同层级系统进行公平对比。仓库同时保存测评方案、可执行资产、系统与数据版本、实验记录和发布结论，形成可追溯的 GitHub 项目历史。

## 测评范围

Benchmark 功能层由五个独立 Track 组成：

1. **Astra**
2. **Memoria**
3. **文档解析**
4. **RAG**
5. **NLP2SQL**

五个 Track 分别定义参评范围、任务、数据、指标和发布结果；公共的版本、运行记录、数据治理和报告规则由仓库统一维护。不同 Track 的结果不默认合并为一个总分。

## 当前状态

- 项目阶段：**仓库与测评治理基线建设**
- 当前日期：2026-07-22
- 仓库目录、路线图、决策记录、系统清单和实验留痕规则已经建立
- Astra 与 NLP2SQL 已有方案草稿，但**尚未审阅，不代表已确认的测评范围或执行承诺**
- Memoria、文档解析和 RAG 尚需分别建立并审阅 Track 方案
- 五个 Track 均未进入正式版本冻结、实验运行或结果发布阶段

实时状态见 [当前项目状态](docs/status.md)，长期推进顺序见 [ROADMAP.md](ROADMAP.md)。

## 仓库地图

```text
.
├── benchmark/       # 五个 Track 的合约、任务、适配器、评分器与 harness
├── datasets/        # 数据来源、许可证、版本、划分与校验和
├── docs/
│   ├── plans/       # 各 Track 的方案草稿与审阅版本
│   ├── research/    # 调研与可行性证据
│   ├── decisions/   # 影响公平性、兼容性或架构的 ADR
│   ├── progress/    # 按日期追加的阶段快照
│   └── archive/     # 已废弃但需保留的问题线索
├── systems/         # MOI、竞品、模型和运行环境的版本清单
├── runs/            # 轻量实验记录、摘要、哈希和外部工件链接
├── reports/         # Track 报告、综合报告与对外发布版
└── .github/         # Issue 与 PR 留痕模板
```

计划中的 Track 资产边界：

```text
benchmark/
├── tasks/
│   ├── astra/
│   ├── memoria/
│   ├── document-parsing/
│   ├── rag/
│   └── nlp2sql/
├── adapters/
├── evaluators/
├── harness/
└── contracts/
```

目录的完整职责见 [仓库结构约定](docs/repository-structure.md)。空目录不会提前创建；每个 Track 在方案通过审阅并建立对应 Issue 后逐步落地资产。

## Track 方案状态

| Track | 当前状态 | 方案入口 |
|---|---|---|
| Astra | 草稿待审阅 | [Astra Benchmark 方案草稿](docs/plans/astra-benchmark-plan.md) |
| Memoria | 待建立方案 | — |
| 文档解析 | 待建立方案 | — |
| RAG | 待建立方案 | — |
| NLP2SQL | 草稿待审阅 | [NLP2SQL Benchmark 方案草稿](docs/plans/nlp2sql-benchmark-plan.md) |

上述草稿在审阅通过前只作为讨论材料，不据此冻结竞品、任务、指标、排期或资源。

旧的整体方案包含已发现的问题，只保留在 [archive](docs/archive/moi-benchmark-plan-v0.md)，不得作为当前执行依据。

## GitHub 留痕流程

```text
研究证据
  → Track 方案审阅
  → ADR / GitHub Issue / Milestone
  → PR 实现
  → 版本冻结
  → 不可变 Run Record
  → Track 报告
  → 综合报告与 Release
  → 持续回归
```

1. **工作先有 Issue**：任务、适配器、数据、评分器和实验均先定义验收标准。
2. **方案通过后实施**：未经审阅的 Track 方案不进入正式开发和结果发布。
3. **关键选择写 ADR**：参评边界、指标、Oracle、汇总口径和不兼容变更必须留下决策记录。
4. **版本必须冻结**：系统、模型、数据、任务、配置和 evaluator 都能追溯到版本或校验和。
5. **运行记录不可覆盖**：每次有效运行使用新的 `run_id`；修复后重跑也新建记录。
6. **大文件不进 Git**：提交 manifest、摘要、哈希和稳定链接；私有数据、密钥和大工件保存在受控存储。
7. **报告引用运行证据**：任何数字必须能反查 `run_id`、Track、任务版本和系统版本。

## 下一步

先确定五个 Track 的负责人和审阅顺序；分别完成方案审阅与边界确认，再为通过审阅的 Track 建立首批 GitHub Issues。统一基础设施与 Track 实施节奏见 [ROADMAP.md](ROADMAP.md)。
