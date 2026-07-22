# Reports

报告分三类：

- `analyses/`：探索性分析，不代表正式排名；
- `milestones/`：M1/M2 等内部阶段报告；
- `releases/`：经过公平性、复现性、安全和数据审查的发布版。

每份报告必须列出：采用/排除的 `run_id`、task-set version、system manifest、evaluator version、置信区间或重复策略、失败归因、限制与已知偏差。

任何排行榜更新都生成新的报告版本和 release tag，不覆盖旧版本。
