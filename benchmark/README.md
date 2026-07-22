# Benchmark implementation

本目录承载可执行测评资产。当前处于 Phase 0，仅冻结结构边界；代码和任务在对应 Issue 明确验收标准后进入。

计划结构：

- `contracts/`：task、system、run、result 的版本化 schema。
- `tasks/astra/`、`tasks/nlp2sql/`：平台无关任务、输入和 Oracle。
- `adapters/`：参评系统薄适配层，不补造产品原生能力。
- `evaluators/`：执行式评分、规则评分和必要的 Judge 配置。
- `harness/`：调度、隔离、采集、预算和失败分类。
- `fixtures/`：小型、无敏感信息且许可证允许提交的夹具。
- `tests/`：schema、adapter、evaluator 与可重复性测试。

第一份实现应先定义 schema 和一个端到端 smoke task，而不是一次性搭建全部目录。
