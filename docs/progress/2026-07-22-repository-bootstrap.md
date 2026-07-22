# 2026-07-22：仓库留痕骨架建立

## 起始状态

- 根 Git 仓库位于 `main`，尚无 commit。
- 已有五份 MOI/Astra/NLP2SQL 研究与计划文档。
- 本地存在 Astra、Hermes Agent、AgentOps 三个独立 Git checkout，总体积约 1.1 GB。
- 尚无统一 benchmark 实现、冻结任务、数据 manifest、run record 或报告。

## 本次完成

- 将当前计划、研究和非权威旧稿分层归档。
- 建立 README、路线图、实时状态、目录职责和 ADR 规则。
- 建立 benchmark、dataset、system、run 与 report 的最小入口。
- 增加 GitHub Issue/PR 模板，要求任务、实验和结论关联稳定证据。
- 忽略本地第三方源码、大型数据和原始运行工件，改用 manifest、校验和与外部工件链接留痕。

## 当前结论

项目处于 Phase 0。研究与方案已经足够支持下一步冻结，但没有任何正式 benchmark 数字可以发布。Astra 与 AgentOps 的本地 checkout 存在修改，当前只能作为开发现场，不能作为可复现实验版本。

## 下一退出条件

1. 建立远端 GitHub repo、M0/M1 milestones 和首批 Issues。
2. 确认维护人、评测设计审批人和结果发布人。
3. 选择 Astra 或 NLP2SQL 作为首个闭环 Track。
4. 冻结参评系统与 Smoke/Pilot 任务，产出第一个可复现 `run_id`。
