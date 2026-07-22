# ADR-0002：根级五 Track 自包含目录

- 状态：Accepted
- 日期：2026-07-22
- 替代：[ADR-0001](0001-repository-layout-and-traceability.md)

## 背景

原结构把计划、数据、系统、运行和报告按资产类型放在仓库公共目录，导致一个 Track 的证据分散在多处，也使历史方案与当前执行边界难以区分。

## 决定

1. Astra、Memoria、文档解析、RAG、NLP2SQL 作为五个根级目录。
2. 每个 Track 自己保存 plan、research、benchmark、datasets、systems、runs、reports、decisions 和 progress。
3. `docs/` 只保存跨 Track 治理和旧混合材料。
4. 第三方源码不进入 Git；与单一 Track 直接相关的工具可保存在该 Track 内的 Git 忽略目录。文档解析使用的 OmniDocBench 位于 `document-parsing/OmniDocBench/`。

## 后果

- 不再使用顶层 `benchmark/`、`datasets/`、`systems/`、`runs/` 和 `reports/`。
- Track 负责人可以在一个目录内完成方案到报告的完整追溯。
- 跨 Track 复用通过明确协议实现，但不把 Track 资产重新移回公共目录。
