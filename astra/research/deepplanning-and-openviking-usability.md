# DeepPlanning 数据与 OpenViking 的 Astra 测评适用性

> 日期：2026-07-22  
> 范围：Astra、Hermes、Goose  
> 状态：候选数据与参考组件审查；尚未完成本地复现  
> 证据口径：官方论文、官方数据仓库和实现代码优先；项目方自报结果只作为待复现假设

## 1. 研究问题与结论

本审查回答两个问题：

1. DeepPlanning v1.1 能否作为 Astra、Hermes、Goose 的可复现产品测评任务？
2. OpenViking 的 README、集成和 Benchmark 资产应当作为数据集、竞品、共同组件还是方法参考？

结论：

- DeepPlanning 的数据、离线环境和规则检查器具备较高可获得性，可作为 P1 长程规划专项；官方代码没有三个产品的 Runner，Travel 评分前还依赖 Qwen-Plus 做格式转换，因此不能原样视为产品 Benchmark。
- DeepPlanning 主要测主动信息获取、约束推理和全局优化，不直接测 Crash 恢复、持久 checkpoint、权限、审计或重复副作用。它不能单独验证 Astra 的“状态可维护、可控制”主张。
- OpenViking 是上下文数据库，不是任务数据集，也不是本轮竞品。其 Benchmark 目录可作为长期记忆、经验记忆和检索成本测评的方法来源，但官方结果属于项目方自报证据。
- Hermes 已有 OpenViking 原生 memory provider，而 Astra、Goose 目前只能按通用 MCP 路径设计接入。未经控制的直接比较会把接入成熟度、自动预取和会话 hook 混入产品效果。

## 2. 三类“状态”必须分开

| 状态层 | 代表对象 | DeepPlanning | OpenViking | Astra 当前主张 |
|---|---|---|---|---|
| 任务与环境状态 | 库存、票务、营业时间、购物车、预算 | 主要覆盖 | 非重点 | 通过工具使用间接覆盖 |
| 上下文与记忆状态 | 历史会话、用户偏好、Agent 经验、知识资源 | 基本不覆盖 | 主要覆盖 | memory/context 属于运行状态的一部分 |
| 持久执行与控制状态 | run、child、checkpoint、owner、权限、outbox、恢复语义 | 不覆盖 | 不覆盖 | Git4Data、Introspect/Reflect、Server–CLI 边界的核心 |

因此，“在单次长任务中记住约束”不能推出“进程重启后状态仍一致”；“检索到历史偏好”也不能推出“外部副作用只执行一次”。三类结果必须分榜报告。

## 3. DeepPlanning v1.1 的事实基线

### 3.1 数据与任务

官方页面和论文给出的任务结构是：

- Travel Planning：120 个独立任务，每个任务有中文和英文版本；9 个 API；每题平均 7,708 条记录；输出分钟级行程和逐项成本。
- Shopping Planning：120 个英文任务；15 个 API；每题平均 171 条记录；输出结构化购物车和优惠券组合。
- 两个领域都运行在按 Case 隔离的离线 Python 环境中。
- 官方实验允许每题最多 400 次工具调用，并对每题重复 4 次。

Travel 的中英文是同一组 120 个问题的语言版本，不能作为 240 个相互独立的规划问题。若两种语言都运行，应按相同 task id 做配对语言分析。

### 3.2 版本与许可证

- 官方页面标记 v1.1 发布于 2026-03-03，修正了部分 Shopping 题目的错误答案标注。
- Hugging Face 数据仓库声明 Apache-2.0，总文件约 104 MB。
- 当前 v1.1 数据仓库提交为 `213876cce679f993a476d01042e13d111c0e3648`。
- Travel 数据来自飞猪、高德和 Web Search 等真实世界来源，Shopping 数据为合成数据。Apache-2.0 声明不自动消除真实来源数据的再分发与来源追踪问题，正式发布数据镜像前仍需做数据治理审查。

### 3.3 评分链

Shopping 通过最终 cart 与 ground truth 对比，主要输出 Match Score 和严格 Case Accuracy。该路径以规则检查为主，适合作为首个适配 Smoke。

Travel 的链路是：

```text
Agent Markdown 行程
  -> 固定转换模型（代码硬编码 qwen-plus）
  -> 标准 JSON
  -> Commonsense / Personalized 规则检查
  -> Composite Score / Case Accuracy
```

因此“规则评分”只适用于转换后的结构化数据；格式转换仍可能带来模型误差、版本漂移、额外成本和外部 API 故障。正式运行至少需要：

- 冻结转换模型的完整版本和 endpoint；
- 保存原始报告、转换 prompt、转换响应和最终 JSON；
- 对不少于 10% 的样本做人工格式一致性复核；
- 把转换失败与任务失败分开计数；
- 可选地增加一个直接要求结构化 JSON 的 MOI 变体，但不得与官方成绩混合。

### 3.4 产品适配边界

官方代码使用项目自带的 OpenAI-compatible function-calling 循环，没有 Astra、Hermes、Goose 的产品级运行接口。MOI 需要新增薄 Runner，并保持以下不变量：

1. 三个平台看到的 task query、工具说明、数据库内容和初始状态一致。
2. 每个 Case 使用独立数据库副本，任务间不共享购物车、库存或历史消息。
3. 工具返回、异常类型和 side effect 语义一致；MCP 包装不能改变搜索排序或参数默认值。
4. 模型、endpoint、生成参数、最大工具调用数、wall-clock timeout 和重试预算在 Track B 一致。
5. 产品无法导出的 trace 字段记为空，不用额外 Agent 替它补造规划、重试或审计。

通过共同 MCP server 暴露工具是首选受控方案，但这会形成 `DeepPlanning-MOI` 派生实现。官方原始 harness 结果和产品 Runner 结果必须分别命名。

## 4. DeepPlanning 对 Astra 主张的效度

| Astra 维度 | 原始 DeepPlanning 覆盖 | 说明 |
|---|---|---|
| 动态规划与编排 | 高 | 长程信息获取、局部约束和全局优化是其主目标 |
| 工具使用 | 中高 | API 数量多、调用链长，但协议可靠性和连接恢复不是主变量 |
| 执行监督 | 中 | 可观察遗漏查询、冗余调用和无效回退，但无显式 supervisor 注入 |
| 持久化与故障恢复 | 低 | 单轮完整运行，没有进程 Crash 或竞争恢复 |
| 可观测与审计 | 低 | 保存轨迹不等于验证因果链、状态转换和敏感字段 |
| 权限与隔离 | 低 | Case 数据隔离存在，但不测身份、授权、路径或租户边界 |
| Git4Data 数据恢复 | 无 | 数据库只是题目环境，不提供 snapshot/branch/merge 对照 |

原始数据只进入 P1。若要把它用作状态控制载体，应另建 `DeepPlanning-State` 派生集：

- 第 N 次工具调用后终止进程并从 durable state 恢复；
- cart 写入成功但响应丢失，验证 Outcome Unknown 与重复副作用；
- 已查询的库存、航班或营业时间在后续步骤改变，验证过期状态发现与重规划；
- 中途修改预算、日期或商品要求，验证旧计划失效和状态迁移；
- 两个实例竞争恢复同一 Case，验证 single owner 和最终状态一致性。

派生集报告恢复率、重复副作用、过期假设发现率、重规划成功率、恢复时延和额外成本，不复用官方 Avg Acc. 名称。

## 5. 建议的 DeepPlanning 纳入方式

### 5.1 适配门槛

- Shopping：Level 1/2/3 各 2 题，共 6 题。
- Travel：选择 6 个不同天数和约束密度的独立 task id，主语言只选一种。
- Astra、Hermes、Goose 各运行 1 次，共 36 次。
- 目的只检查 Runner、工具语义、Case 隔离、预算和评分链，不发布排名。

### 5.2 扩大条件

只有以下条件全部满足才扩大样本：

- 三个平台均能调用完整工具集，或对缺失能力留下 `Unsupported` 证据；
- 相同工具输入得到相同规范化结果；
- Case 初始化和清理可重复；
- Shopping 规则评分可在固定输入上重放；
- Travel 格式转换的人工抽检错误率处于预注册阈值以内；
- infra failure 与 product failure 可以可靠区分。

主分析分别报告 Travel 与 Shopping 的严格 Case Accuracy；partial score 用于失败诊断。不同领域的分数不合成 Astra P0 总分。

## 6. OpenViking 的角色与偏差

### 6.1 可复用资产

OpenViking 将 memory、resource、skill 放入 `viking://` 虚拟文件系统，提供 L0/L1/L2 分层内容、递归检索、检索轨迹和会话提交后的记忆抽取。其 `benchmark/` 当前包含 LoCoMo、LongMemEval、tau2、RAG 和 SkillsBench 等脚本。

这些资产可提供：

- 长期对话记忆的问答准确率、时延与 token 统计；
- 原生记忆、OpenViking E2E 和记忆预导入的实验臂设计；
- 检索 trace、导入成本和回答成本的拆分方法；
- tau2 经验记忆前后任务成功率的消融思路。

### 6.2 不能直接接受的结论

README 和官方博客中的 Hermes、OpenClaw、Claude Code、tau2 提升均由 OpenViking 团队报告，不是独立复现。LoCoMo 脚本还默认使用 LLM Judge。它们只能支持“值得复现”的假设，不能证明 OpenViking 或 Hermes 的相对优劣。

Hermes 官方集成可自动使用 OpenViking memory provider；通用 MCP 文档只保证工具端点可连接，不保证自动预取、上下文注入、会话捕获和提交 hook 等价。因此建议保留两个独立实验臂：

1. `Native Memory`：各产品使用自己的推荐记忆架构，衡量原生产品栈效果。
2. `Common Context Backend`：三个产品只通过相同 OpenViking MCP 工具、相同语料、相同检索预算运行；原生自动预取和生命周期 hook 要么全部关闭，要么单列为生态集成能力。

OpenViking 主项目使用 AGPLv3。内部隔离评测与把服务嵌入、修改后对外提供或随复现包分发不是同一许可证场景，发布前需单独审查。

## 7. 当前限制与下一步证据门槛

- 本次核验了官方页面、论文、代码目录、评分实现、数据文件清单和版本信息；由于当前环境对 Hugging Face 二进制下载超时，尚未在本地解压 v1.1 数据并重放评分器。
- Qwen-Agent 代码 commit、DeepPlanning Runner commit、Qwen-Plus 转换模型版本尚未冻结。
- Goose 尚未加入本地源码快照，也没有完成 DeepPlanning/OpenViking 接口 Smoke。
- OpenViking 的官方数值尚无独立复现，不能进入 MOI 结论。

进入试运行前必须完成：数据文件 SHA256、代码 commit、容器 digest、工具 schema、Runner 版本、模型与转换模型版本、抽样 task id、随机种子、资源预算和 evaluator 重放记录的统一冻结。

## 8. 主要来源

- [DeepPlanning 官方页面](https://qwenlm.github.io/Qwen-Agent/en/benchmarks/deepplanning/)
- [DeepPlanning 论文](https://arxiv.org/abs/2601.18137)
- [DeepPlanning 数据仓库](https://huggingface.co/datasets/Qwen/DeepPlanning)
- [DeepPlanning 官方代码](https://github.com/QwenLM/Qwen-Agent/tree/main/benchmark/deepplanning)
- [Travel 格式转换实现](https://github.com/QwenLM/Qwen-Agent/blob/main/benchmark/deepplanning/travelplanning/evaluation/convert_report.py)
- [OpenViking 中文 README](https://github.com/volcengine/OpenViking/blob/main/README_CN.md)
- [OpenViking Benchmark 目录](https://github.com/volcengine/OpenViking/tree/main/benchmark)
- [OpenViking Hermes 集成](https://docs.openviking.ai/zh/agent-integrations/05-hermes)
- [OpenViking MCP 客户端集成](https://docs.openviking.ai/zh/agent-integrations/06-mcp-clients)
- [OpenViking 自报 Benchmark 结果](https://blog.openviking.ai/post/openviking-benchmark-results/)

## AI 辅助披露

本审查使用 AI 辅助检索、代码定位和证据综合。关键事实以链接的官方页面、仓库、论文和本地文件为准；尚未本地复现的项目均明确标注为限制或待验证项。
