# 当前项目状态

> 快照日期：2026-07-22  
> 总体阶段：Phase 0 / 边界与可运行性  
> 状态维护规则：当 milestone、Track 状态或主要 blocker 变化时随 PR 更新。

## 总览

| 工作流 | 状态 | 已有证据 | 下一门槛 |
|---|---|---|---|
| 仓库治理 | 进行中 | 目录、路线图、ADR 与 GitHub 模板已建立 | 建立远端 repo、milestones、labels 与责任人 |
| Astra | 设计基线就绪 | v0.2 计划、博客优势证据、竞品与 adapter 可行性分析 | 冻结产品 commit、Pilot Case 与 Runner 合约 |
| NLP2SQL | 设计基线就绪 | v0.1 计划、竞品/数据集/指标调研 | 确认 MOI 接口与方言，冻结不超过 30 个 Smoke Case |
| 数据治理 | 未开始 | 已定义只提交 manifest 的仓库规则 | 建立公开/私有数据 manifest、许可证与脱敏规则 |
| Benchmark 实现 | 未开始 | 已定义目标目录边界 | 建立 schema、harness、adapters 与 evaluators |
| 实验运行 | 未开始 | 已定义 run record 规则 | 产出首个可复现 Smoke `run_id` |
| 报告发布 | 未开始 | 已定义 reports 分层 | 用冻结运行生成内部 Pilot 报告 |

## 当前已知风险

1. Astra 本地 checkout 有未提交修改，当前 commit 不能单独复现本地状态。
2. MOI NLP2SQL 的产品接口、方言、语义层和审计输出尚未冻结。
3. 当前没有任务 schema、统一输出 Envelope、evaluator 验证或正式 run record。
4. 私有数据、原始 trace、大模型凭据和客户信息的存储边界尚未形成正式政策。

## 最近进展

- [2026-07-22：仓库留痕骨架建立](progress/2026-07-22-repository-bootstrap.md)
