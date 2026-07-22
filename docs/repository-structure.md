# 五 Track 仓库结构

## 1. 根目录边界

根目录只有五个 Benchmark Track 与少量项目级治理文件：

```text
astra/
memoria/
document-parsing/
rag/
nlp2sql/
docs/
.github/
README.md
ROADMAP.md
```

第三方源码原则上不进入 Git。文档解析 Track 使用的 OmniDocBench 官方评测工具保存在本地 `document-parsing/OmniDocBench/`，该目录整体 Git 忽略，不属于仓库发布内容。

## 2. Track 自包含原则

一个 Track 的全部有效内容必须能在其目录内找到：

| Track 路径 | 内容 |
|---|---|
| `plans/` | 草稿、批准版本和被替代版本 |
| `research/` | 竞品、指标、数据、许可证与技术证据 |
| `benchmark/` | schema、task、adapter、evaluator、harness 与测试 |
| `datasets/` | manifest、版本、划分、校验和及生成方法 |
| `systems/` | 参评系统、模型、工具、运行环境与冻结版本 |
| `runs/` | 不可变运行记录、摘要、失败归因和工件哈希 |
| `reports/` | 分析稿、里程碑报告和发布版 |
| `decisions/` | 仅影响该 Track 的 ADR |
| `progress/` | Track 自身的阶段快照 |

根目录不再建立共享的 `benchmark/`、`datasets/`、`systems/`、`runs/` 或 `reports/`。

## 3. Plan 历史

```text
plans/
├── README.md
├── drafts/
├── approved/
└── superseded/
```

草稿通过 PR 变化；批准时生成不可变版本并记录版本号、Issue、PR、审批人与日期。新版本通过 `supersedes` 指向旧版本，不覆盖历史文件。

## 4. 数据与运行留痕

每个数据 manifest 记录 `dataset_id`、版本、来源、许可证、划分、SHA-256、访问级别、脱敏、保留期和 task-set version。大型或私有 payload 不进入 Git。

每个运行使用新的 `run_id`，并冻结 Track、plan、dataset、task、system、model、config、evaluator 和代码 commit。无效运行保留记录并标明 `invalid_reason`，不能删除后复用同一 ID。

## 5. 跨 Track 内容

只有同时约束多个 Track 的治理规则、项目路线图和 ADR 可以进入 `docs/`。旧的混合方案保存在 `docs/archive/cross-track/`，不得作为当前执行入口。
