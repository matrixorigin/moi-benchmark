# Run records

每次有效尝试使用新的不可变 `run_id`。建议目录名：`<track>/<YYYY-MM-DD>-<short-id>/`。

一个可进入报告的 run record 至少包含：

```text
run.yaml             # run_id、commit、task set、system/model/config/evaluator 版本
environment.yaml     # OS、镜像、资源、依赖与数据库快照
summary.json         # 聚合指标、样本数、失败数和不确定项
failures.csv         # 失败分类与人工复核状态
checksums.sha256     # 外部原始工件校验和
README.md            # 目的、偏差、异常、工件 URI 和结论边界
```

原始 trace、日志、数据库快照和大输出不提交 Git。它们进入受控工件存储，并在 `run.yaml` 中记录稳定 URI、访问级别、大小、SHA-256 和保留期。

运行无效时保留记录并增加 `status: invalid` 与 `invalid_reason`；不得删除或覆盖后伪装成同一次运行。
