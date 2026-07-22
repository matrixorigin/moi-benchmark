# System registry

`manifest.yaml` 记录 MOI、竞品、模型、工具和关键运行时的来源与冻结状态。它是开发现场快照，不自动等于正式实验清单。

正式冻结时：

1. 使用 tag、commit digest 或不可变镜像 digest；
2. 记录许可证、入口、配置与赛道资格；
3. 工作树必须干净，或提交可审查 patch 并记录其 SHA-256；
4. 模型、系统提示、工具 schema 和适配器版本单独记录；
5. manifest 变更通过 PR，并说明是否导致结果不可横向比较。
