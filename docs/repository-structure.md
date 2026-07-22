# 仓库结构与证据链约定

## 1. 设计目标

目录必须同时回答四个问题：为什么这样测、具体测什么、这次如何运行、结论从哪里来。为此，方案证据、可执行资产、运行证据和发布材料分层保存。

## 2. 目录职责

| 路径 | 保存内容 | 不应保存 |
|---|---|---|
| `docs/research/` | 竞品、数据集、指标、许可证和技术可行性证据 | 当前执行口径、原始运行日志 |
| `docs/plans/` | Track 范围、指标、任务组合、阶段与退出条件 | 不可复现的临时结论 |
| `docs/decisions/` | 影响公平性、架构、兼容性或发布口径的 ADR | 日常 TODO、会议流水账 |
| `docs/progress/` | 按日期追加的里程碑快照、已完成项和 blocker | 会被持续覆盖的实时状态 |
| `benchmark/` | schema、task、adapter、evaluator、harness 和测试 | 大型数据 payload、一次性 notebook 输出 |
| `datasets/` | 数据来源、版本、许可证、划分、校验和与生成脚本 | 私有原文、下载缓存、凭据 |
| `systems/` | 参评系统、模型、工具与运行时版本清单 | 直接 vendor 的第三方完整仓库 |
| `runs/` | 不可变 run manifest、摘要、失败归因、哈希和工件链接 | 密钥、客户数据、可覆盖的 `latest` 结果 |
| `reports/` | 由冻结 run 生成的分析与发布版 | 无 run 引用的宣传结论 |

## 3. Benchmark 内部结构

代码到来时按职责逐步创建，不为“看起来完整”预建空目录：

```text
benchmark/
├── contracts/       # task/system/run/result 的版本化 schema
├── tasks/
│   ├── astra/       # 平台无关的 Agent 任务与 Oracle
│   └── nlp2sql/     # 问题、数据库、权限和 Oracle
├── adapters/        # 各参评系统的薄适配层
├── evaluators/      # 规则、执行式评分器与 Judge 配置
├── harness/         # 调度、隔离、采集和失败分类
├── fixtures/        # 小型、无敏感信息、可提交的测试夹具
└── tests/           # schema、adapter 与 evaluator 自测
```

任务表达用户目标、输入、权限、预算与 Oracle，不描述某个参评系统的内部 API。平台原生不支持的能力记录为 `Unsupported`，不得通过 adapter 补造能力。

## 4. 端到端留痕链

```text
研究证据
  → ADR / 计划变更
  → GitHub Issue + milestone
  → PR（任务、代码、配置或数据 manifest）
  → 冻结版本
  → run record + 外部工件哈希
  → 报告
  → tag / GitHub Release
  → 持续回归趋势
```

每个环节至少引用前一环节的稳定标识：Issue 号、PR 号、commit、task-set version、`run_id` 或 release tag。

## 5. 版本与变更规则

- Task、schema 和 evaluator 使用显式版本；语义变化必须升版本。
- 运行一旦进入报告不得覆盖；无效运行增加 `invalid_reason`，保留原记录。
- 报告必须列出包含和排除的 `run_id`，并解释定向重跑。
- 竞品 commit、模型标识、系统提示、工具 schema、数据库快照和资源限制均是实验版本的一部分。
- 大型或私有工件使用受控存储；Git 中保存 URI、大小、SHA-256、访问级别和保留期。

## 6. GitHub 映射

- Milestone：对应 `ROADMAP.md` 的 M0–M4。
- Issue：一个可验收的工作项或一次计划实验。
- PR：实现 Issue，并说明是否改变比较口径或历史兼容性。
- ADR：只记录长期有效的关键选择。
- Release：冻结报告、task set、system manifest 与纳入的 run records。

建议 labels：`track/astra`、`track/nlp2sql`、`area/task`、`area/adapter`、`area/evaluator`、`area/data`、`area/report`、`experiment`、`decision`、`blocked`、`security`。
