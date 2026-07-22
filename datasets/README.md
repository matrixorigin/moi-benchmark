# Dataset registry

Git 只保存数据 manifest 和可公开的小型夹具，不保存下载缓存、私有原文或客户数据。

每个数据集 manifest 至少记录：

- `dataset_id`、版本与用途；
- 官方来源、许可证和引用方式；
- 下载/生成方法与 SHA-256；
- train/dev/test 或自定义 split；
- 抽样规则、去重、修订和污染检查；
- 公开/内部/受限访问级别；
- 脱敏方法、保留期和责任人；
- 对应 task-set version。

私有数据只能通过受控路径加载。manifest 可以进入 Git，但不得包含客户名称、原始内容、凭据或可反推敏感信息的样本。
