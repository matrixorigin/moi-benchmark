# ADR-0001：仓库分层与项目留痕

- 状态：Accepted
- 日期：2026-07-22

## 背景

初始工作区把研究文档和三个完整外部源码 checkout 放在根目录，根 Git 仓库尚无提交。项目需要同时承载当前方案建设、后续 benchmark 实现、长期回归和 GitHub 审计记录。

## 决定

1. 将调研、计划、决策、进度、可执行 benchmark、数据清单、系统版本、运行记录和报告分层保存。
2. 外部参评系统源码不直接提交到主仓库；只在 `systems/manifest.yaml` 冻结来源、commit 和本地改动状态。
3. Git 只提交轻量且可审查的实验证据。大数据、原始 trace 和大工件存放在受控存储，并以哈希和稳定链接引用。
4. GitHub Issue/PR 记录协作过程；ADR、progress、run record 和 report 记录长期可审计事实。

## 理由

- 防止超过 1 GB 的第三方源码污染 MOI benchmark 自身历史。
- 让计划变更、实验实现、运行证据和发布结论之间可以反向追溯。
- 避免覆盖旧结果或用新评分器静默重算历史排行榜。
- 允许私有业务数据参与实验，同时不进入公开 Git 历史。

## 后果

- 首次正式运行前必须补齐 system、dataset、task、config 和 evaluator 版本。
- 本地修改过的参评系统必须形成 patch/镜像摘要或干净 commit，才能被标记为可复现。
- 报告不能直接引用本地临时输出；必须先形成 run record。
