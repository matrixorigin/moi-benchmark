# Morpheus 测评方案

## 目录

- [1. 适用范围与目标](#1-适用范围与目标)
- [2. 指标分类](#2-指标分类)
- [3. 测评数据要求](#3-测评数据要求)
- [4. 主指标测评表](#4-主指标测评表)
- [5. Baseline 分组](#5-baseline-分组)
- [6. 通过标准](#6-通过标准)
- [7. 测评频率](#7-测评频率)
- [8. 执行前置条件](#8-执行前置条件)
- [9. 最小执行流程](#9-最小执行流程)
- [10. 当前阶段推荐结论口径](#10-当前阶段推荐结论口径)
- [11. 排期与资源要求](#11-排期与资源要求)
- [12. 后续增强项](#12-后续增强项)

## 1. 适用范围与目标

本文档用于定义当前阶段 Morpheus 的基础测评方案。测评重点是验证系统能否形成一条稳定、可复现、可审计的闭环：

```text
业务/trace 数据 -> 训练样本 -> LoRA adapter -> eval -> replay -> gate -> release/rollback
```

当前阶段的核心评估目标：

- 评估结构化任务效果是否相对 baseline 有稳定提升。
- 评估训练、评估、回放、门控、发布、回滚链路是否可稳定执行。
- 评估 replay/gate 是否能降低低质量 adapter 或退化 adapter 上线风险。
- 为后续多轮自进化、GRPO 增益、trace 决策能力评测提供基础口径。

当前阶段暂不将以下方向作为主 KPI，可作为后续扩展评测项：

- GRPO 带来显著单轮涨分。
- 多轮自进化连续稳定上涨。
- 完整 long-horizon agent success rate。
- 开放问答能力大幅提升。
- 类人决策能力或类人认知一致性。

## 2. 指标分类

当前基础测评采用 8 个主指标，分为功能指标和系统指标。

### 2.1 功能指标

功能指标用于评估 adapter 的任务效果、结构化能力、回归稳定性和上线可用性。

| 指标 | 说明 |
|---|---|
| `overall_score` | 综合任务效果分数，用于判断 candidate adapter 是否优于 baseline |
| `schema_compliance` | 结构化输出合规率，用于判断 JSON/schema 输出是否稳定 |
| `field_accuracy / action_quality` | 字段准确率或动作质量，用于判断字段抽取、动作选择是否有效 |
| `regression_pass_rate` | 回归集通过率，用于判断关键旧能力是否退化 |
| `replay_pass_rate` | 回放通过率，用于判断 candidate adapter 在线上前置回放中的风险 |
| `gate_verdict` | 门控结论，用于判断 candidate adapter 是否允许进入发布阶段 |

说明：

- `replay_pass_rate` 同时具备功能评估和治理评估属性。当前方案将其归入功能指标，因为它直接服务于 adapter 是否可上线的判断。
- `gate_verdict` 是 eval、replay、稳定性、风险策略共同作用后的发布前判定结果。

### 2.2 系统指标

系统指标用于评估 Morpheus 平台链路是否能稳定执行。

| 指标 | 说明 |
|---|---|
| `training_job_success_rate` | 训练任务成功率，用于判断训练链路是否能稳定产出 adapter |
| `rollback_success_rate` | 回滚成功率，用于判断发布退化或失败时是否具备安全恢复能力 |

以下指标建议作为辅助系统指标记录，但不作为当前主 KPI：

```text
adapter_load_success_rate
artifact_save_success_rate
release_route_success_rate
latency_p95
error_rate
fallback_rate
eval_success_rate
```

### 2.3 自进化过程指标

自进化过程指标用于评估 Morpheus 是否完成“数据 -> 样本 -> adapter -> eval/gate”的闭环。当前仅保留 1 个扩展指标，不替代第 2.1 和 2.2 节的 8 个主指标。

| 指标 | 说明 | 推荐口径 |
|---|---|---|
| `self_evolution_loop_completion_rate` | 自进化闭环完成率，用于评估一轮数据进入后是否完成“样本产出、adapter 产出、eval/gate”闭环 | `completed_evolution_rounds / planned_evolution_rounds` |

一轮 `completed_evolution_round` 至少应满足：

```text
产生可训练样本
成功产出 candidate adapter
完成 eval
完成 gate
```

该指标只能说明闭环是否跑通，不能单独证明模型具备长期、持续、自主的自进化能力。

## 3. 测评数据要求

当前基础测评建议固定四类数据集。若本轮只评估已有 adapter，可先准备 `holdout`、`regression`、`replay`；若本轮同时覆盖训练链路，则需要同时准备 `train`。

| 数据集 | 用途 | 建议规模 | 要求 |
|---|---|---:|---|
| `train` | 训练 adapter | 按任务决定 | 可来自业务数据、trace、公开数据或人工样本 |
| `holdout` | 测泛化和主效果 | 50-200 条 | 与 train 严格隔离，固定版本 |
| `regression` | 测关键旧能力是否退化 | 20-100 条 | 选择历史核心样本、关键业务样本、已知风险样本 |
| `replay` | 测上线前风险 | 20-100 条 | 优先包含真实链路、高风险样本、hardcase 样本 |

如需评估 trace 决策能力，应额外准备以下数据集。

| 数据集 | 用途 | 要求 |
|---|---|---|
| `trace_transition_eval` | 评估 `state -> action` 决策质量 | 按 action 类型分层 |
| `hardcase` | 收集失败、边界、工具错误样本 | 可随评测轮次持续补充 |
| `reward_preflight` | 评估 reward 是否具备区分度 | 用于 GRPO 前置检查 |

数据集应版本化管理，例如：

```text
holdout_v1
regression_2026_07
replay_v1
trace_transition_eval_v1
```

正式测评前应固定一份可复现的数据快照，并记录：

```text
manifest
样本数量
样本来源
时间窗口
hash
训练/评测隔离说明
```

注意：本节中的 `train`、`holdout`、`regression`、`replay` 是测评数据类型，不等价于当前仓库中已经提交的正式基准集。正式测评必须使用冻结的数据快照；demo 可以使用临时数据，但报告中必须标注数据集状态。

### 3.1 测评数据交付规范

为避免不同执行人使用不同数据导致结果不可比较，正式测评建议交付一套固定目录，例如：

```text
benchmarks/morpheus-eval-v1/
  manifest.json
  train.jsonl
  holdout.jsonl
  regression.jsonl
  replay.jsonl
  trace_transition_eval.jsonl      # 可选
  reward_preflight.jsonl           # 可选
```

其中 `manifest.json` 至少记录：

```json
{
  "dataset_id": "morpheus-eval-v1",
  "created_at": "YYYY-MM-DD",
  "task_type": "structured_domain_task",
  "splits": {
    "train": {"path": "train.jsonl", "count": 0},
    "holdout": {"path": "holdout.jsonl", "count": 0},
    "regression": {"path": "regression.jsonl", "count": 0},
    "replay": {"path": "replay.jsonl", "count": 0}
  },
  "source": "business/trace/public/manual",
  "split_policy": "train/holdout/regression/replay are de-duplicated and isolated",
  "hash": ""
}
```

如果当前阶段尚未确定正式数据，可先创建 `benchmarks/morpheus-eval-draft-v1/` 用于试跑；试跑通过后再冻结为正式版本。

### 3.2 样本格式规范

所有测评文件建议采用 JSONL 格式，即每行一个 JSON object。

#### 3.2.1 结构化任务样本

`holdout.jsonl`、`regression.jsonl`、`replay.jsonl` 中用于 `scripts/evaluate_adapter.py` 的结构化样本，至少包含：

| 字段 | 类型 | 是否必需 | 说明 |
|---|---|---|---|
| `input_text` | string | 是 | 原始输入文本，评测脚本会基于该字段构造 prompt |
| `expected_output` | object | 是 | 标准输出，用于计算字段准确率、schema 合规率、action 质量 |
| `prompt_text` | string | 否 | 如果提供，则直接作为模型输入 prompt；否则由 `input_text` 自动构造 |
| `metadata` | object | 否 | 样本来源、业务标签、trace_id、难度等辅助信息 |

最小样例：

```json
{"input_text":"用户反馈支付成功但订单仍显示未支付，需要判断问题类型和处理动作。","expected_output":{"issue_type":"payment_error","severity":"high","product":"payment","root_cause":"transaction_state_mismatch","user_impact":"order_blocked","action":["inspect payment transaction","repair order payment state"]},"metadata":{"source":"business_case","split":"holdout"}}
```

`expected_output` 建议包含以下字段：

| 字段 | 说明 |
|---|---|
| `issue_type` | 问题类型 |
| `severity` | 严重程度，建议使用 `low`、`medium`、`high`、`critical` |
| `product` | 产品、模块或业务域 |
| `root_cause` | 归因字段 |
| `user_impact` | 用户影响 |
| `action` | 推荐动作列表，至少 1 个动作 |

如需评估更细的决策质量，可在 `expected_output` 或顶层字段中补充：

```json
{
  "decision_rubric": {
    "preferred_actions": ["inspect payment transaction", "repair order payment state"],
    "must_not_actions": ["close ticket"],
    "action_keywords": ["payment", "transaction", "repair"],
    "risk_focus": ["order_blocked"]
  }
}
```

#### 3.2.2 trace transition 样本

`trace_transition_eval.jsonl` 用于评估 agent trace 中的 `state -> action` 决策质量。建议每行至少包含：

| 字段 | 类型 | 是否必需 | 说明 |
|---|---|---|---|
| `trace_id` | string | 是 | 原始 trace 标识 |
| `transition_id` | string | 是 | 当前决策点标识 |
| `state` | object | 是 | 当前状态，包括任务、历史、可用工具、上下文等 |
| `action` | object | 是 | 期望动作或参考动作 |
| `reward` | number | 否 | 当前 step 的奖励分 |
| `done` | boolean | 否 | 是否为终止动作 |
| `metadata` | object | 否 | action 类型、来源、质量标签等 |

最小样例：

```json
{"trace_id":"trace-001","transition_id":"trace-001:step-003","state":{"task":"查询订单支付状态","history":[{"role":"user","content":"订单已支付但状态未更新"}],"available_tools":["db_query","search","final_answer"]},"action":{"type":"tool_call","tool_name":"db_query","args":{"table":"orders","field":"payment_status"}},"reward":0.9,"done":false,"metadata":{"action_type":"tool_call","split":"trace_transition_eval"}}
```

#### 3.2.3 replay 样本

`replay.jsonl` 可以复用结构化任务样本格式；如果需要真实链路回放，建议额外保留：

| 字段 | 类型 | 是否必需 | 说明 |
|---|---|---|---|
| `request_id` | string | 否 | 原始请求标识 |
| `workspace_id` | string | 否 | workspace 标识 |
| `application_id` | string | 否 | 应用标识 |
| `baseline_output` | object/string | 否 | stable adapter 或 base 的历史输出 |
| `expected_output` | object | 是 | 回放评测目标 |
| `metadata` | object | 否 | 来源、风险标签、线上反馈等 |

replay 样本应优先包含真实链路、高风险样本、历史失败样本和 hardcase，不建议只使用随机简单样本。

## 4. 主指标测评表

| 目标 | 数据 | 指标 | 方法 | 工具 | Baseline | 阈值 | 频率 |
|---|---|---|---|---|---|---|---|
| 评估整体任务效果 | 固定 `holdout` | `overall_score` | candidate adapter 与 baseline 在同一 holdout 上跑分 | `scripts/evaluate_adapter.py` / `morpheus/eval.py` | Base 模型、上一版 stable adapter | candidate >= baseline + 3%-5%，且不低于上一版 stable | 每次训练后 |
| 评估结构化输出稳定性 | 固定 `holdout` | `schema_compliance` | 校验输出 JSON 是否可解析、字段是否完整、枚举是否合法 | `scripts/evaluate_adapter.py` | Base / stable adapter | >= 95% 为理想；最低不低于 90% | 每次训练后 |
| 评估字段或动作学习效果 | 结构化业务样本或 trace action 样本 | `field_accuracy / action_quality` | 对比 expected output，统计字段命中率和 action 覆盖度 | `scripts/evaluate_adapter.py` / trace evaluator | Base / LoRA-SFT | 比 baseline 提升，建议 +5% 以上 | 每次训练后 |
| 评估关键能力是否退化 | 固定 `regression` | `regression_pass_rate` | 新 adapter 跑核心旧样本，与 stable 对比 | eval 脚本 / regression runner | 上一版 stable adapter | 不低于 stable，或下降不超过 1%-2% | 每次训练后 |
| 评估上线前回放风险 | 固定 `replay` | `replay_pass_rate` | candidate 和 stable 跑同一 replay，比较通过率 | `replay_runs` / `sleep_replay` | stable adapter | >= stable，且达到 gate 阈值，例如 80%-90% | 每次发布前 |
| 评估发布前门控结论 | eval + replay 结果 | `gate_verdict` | 根据 score、latency、error、replay 结果生成 gate verdict | `gate_results` / `training_gate_service` | 无治理或人工判断 | 低分/退化 adapter 应 reject；通过的 adapter 不应明显退化 | 每次训练后 |
| 评估训练链路稳定性 | 训练 job 记录 | `training_job_success_rate` | 统计训练 job 是否成功完成并产出 adapter | DB/API/训练产物 | 历史成功 job 或当前批次均值 | demo 阶段 >= 90%；稳定阶段 >= 95% | 每周/每批 |
| 评估发布安全恢复能力 | release/rollback 记录 | `rollback_success_rate` | 检查 release 退化或失败时能否成功回滚 | `release_gateway` / `release_service` | stable release | 接近 100% | 每次发布或演练 |

## 5. Baseline 分组

建议采用以下最小对照分组。

| 分组 | 说明 |
|---|---|
| `Base` | 原始基础模型，不加载 adapter |
| `LoRA-SFT` | 只做 SFT 后的 adapter |
| `LoRA-SFT + Replay/Gate` | SFT adapter 加上 Morpheus replay/gate 治理 |
| `Self-Evolve v1` | 数据管线 + 训练 + eval + replay + gate + release/rollback |

GRPO 可作为增量观察项纳入实验，但当前不建议将整体测评结论完全依赖 GRPO 增益。

## 6. 通过标准

candidate adapter 进入发布或下一阶段前，建议至少满足以下条件：

```text
overall_score 高于 baseline 或不低于上一版 stable
schema_compliance 不低于 90%，理想达到 95%+
field_accuracy / action_quality 有明确提升或不退化
regression_pass_rate 不低于 stable，或下降不超过 1%-2%
replay_pass_rate 不低于 stable，并达到 gate 阈值
gate_verdict 不是 reject
训练过程中无 NaN/Inf 致命错误
训练 job 成功产出 adapter artifact
rollback 链路可用
```

如果 candidate adapter 的主效果指标略有提升，但 `regression_pass_rate` 或 `replay_pass_rate` 明显下降，应判定为不可发布或进入人工复核。

## 7. 测评频率

| 时机 | 必测内容 |
|---|---|
| 每次训练后 | `overall_score`、`schema_compliance`、`field_accuracy/action_quality`、`regression_pass_rate`、`training_job_success_rate` |
| 每次发布前 | `replay_pass_rate`、`gate_verdict` |
| canary 期间 | `latency_p95`、`error_rate`、`fallback_rate`、`release_route_success_rate` |
| 每周/每批 | `training_job_success_rate`、`rollback_success_rate`、adapter 产出成功率 |
| 每轮自进化后 | 对比本轮与上一轮 stable 的 score、replay、regression 变化，并统计 `self_evolution_loop_completion_rate` |

## 8. 执行前置条件

本文档可以作为当前阶段测评的执行依据，但正式执行前需要确认以下前置条件。

| 条件 | 是否必需 | 说明 |
|---|---|---|
| 已冻结测评数据快照 | 是 | 至少包含 `holdout.jsonl`、`regression.jsonl`、`replay.jsonl` 和 `manifest.json`；如果本轮覆盖训练链路，还需要 `train.jsonl` |
| 可访问 baseline 模型或 stable adapter | 是 | 用于计算对照结果 |
| 可访问 candidate adapter | 是 | 通常来自一次 Morpheus 训练 job 的产物 |
| vLLM 推理服务可用 | 是 | `scripts/evaluate_adapter.py` 会通过 vLLM API 调用模型 |
| `MORPHEUS_VLLM_BASE_URL` 配置正确 | 是 | 默认通常为 `http://localhost:8000` |
| 数据库/API 可访问 | 视测评范围而定 | 如果要统计 job、gate、release、rollback，需要访问 Morpheus DB 或 API |
| release gateway 可用 | 视测评范围而定 | 如果要测 release/canary/rollback，需要启动 gateway |

如果只做离线 adapter 效果测评，最小前置条件是：

```text
评测数据快照 + baseline/candidate adapter + vLLM 服务
```

如果要做完整闭环测评，还需要：

```text
Morpheus control-plane + training-worker + replay/gate + release-gateway + 数据库
```

## 9. 最小执行流程

### 9.1 数据准备

先准备或冻结一份评测数据：

```text
benchmarks/morpheus-eval-v1/
  manifest.json
  holdout.jsonl
  regression.jsonl
  replay.jsonl
```

确认 JSONL 格式满足第 3.2 节要求。

### 9.2 跑 baseline

使用同一份 `holdout.jsonl` 评估 baseline。

示例：

```bash
python3 scripts/evaluate_adapter.py \
  --adapter-uri <BASELINE_MODEL_OR_STABLE_ADAPTER> \
  --dataset-uri benchmarks/morpheus-eval-v1/holdout.jsonl \
  --output artifacts/evaluation/baseline_holdout_report.json
```

### 9.3 跑 candidate adapter

使用同一份 `holdout.jsonl` 评估 candidate adapter。

示例：

```bash
python3 scripts/evaluate_adapter.py \
  --adapter-uri <CANDIDATE_ADAPTER_URI> \
  --dataset-uri benchmarks/morpheus-eval-v1/holdout.jsonl \
  --output artifacts/evaluation/candidate_holdout_report.json
```

### 9.4 跑 regression

使用 `regression.jsonl` 检查关键能力是否退化。

示例：

```bash
python3 scripts/evaluate_adapter.py \
  --adapter-uri <CANDIDATE_ADAPTER_URI> \
  --dataset-uri benchmarks/morpheus-eval-v1/regression.jsonl \
  --output artifacts/evaluation/candidate_regression_report.json
```

### 9.5 跑 replay/gate/release

如果当前环境已启动 Morpheus 服务和 release 链路，应继续执行：

```text
replay -> gate -> release/canary -> rollback 演练或检查
```

这部分结果优先从以下来源汇总：

```text
replay_runs
gate_results
release_metrics
release_gateway diagnostics
training_jobs
training_evaluations
```

如果当前环境只具备离线能力，则本轮可以先完成 `holdout` 和 `regression`，并在报告中标注 replay/gate/release 未执行。

### 9.6 结果汇总

最终报告至少汇总 8 个主指标：

```text
overall_score
schema_compliance
field_accuracy / action_quality
regression_pass_rate
replay_pass_rate
gate_verdict
training_job_success_rate
rollback_success_rate
```

未执行的指标必须标注原因，例如：

```text
未执行 replay：当前环境未启动 release/replay 链路
未统计 rollback_success_rate：本轮未进行 release/rollback 演练
```

## 10. 当前阶段推荐结论口径

测评报告应按实际执行范围组织结论。若已完成离线测评和闭环测评，可采用以下口径：

```text
Morpheus 在结构化领域任务上具备可验证的提升潜力；
Morpheus 可形成训练、评估、回放、门控、发布、回滚的受控闭环；
相比单纯训练 adapter，Morpheus 的主要价值体现在效果提升、风险拦截和可追溯发布治理。
```

如果加入自进化过程指标，可以补充以下结论口径：

```text
Morpheus 已具备从反馈/trace 到训练样本、候选 adapter、评估门控的闭环过程；
当前阶段可验证的是“受控离线 adapter 级自优化闭环”，而不是成熟的长期自主自进化；
自进化过程应通过 `self_evolution_loop_completion_rate` 验证，即 planned round 中有多少轮完成了“样本产出、adapter 产出、eval/gate”的闭环。
```

当前阶段不建议直接将结论表述为：

```text
Morpheus 已经实现长期自进化；
Morpheus 能稳定提升完整 agent long-horizon 决策能力；
GRPO 是当前效果提升的主要来源。
```

## 11. 排期与资源要求

### 11.1 排期安排

| 阶段 | 目标 | 产出 |
|---|---|---|
| 测评准备 | 固定测评方案、数据格式、baseline、阈值 | `morpheus-eval-v1`、manifest、测评执行说明 |
| 离线测评 | 跑 Base / SFT / candidate adapter 对比 | holdout、regression、主指标结果 |
| 闭环测评 | 跑 eval -> replay -> gate -> release/rollback | replay/gate/release 报告、rollback 演练结果 |
| 受控试点 | 接入 1 个真实 workspace，小流量验证 | 数据接入、训练、发布、线上指标、问题清单 |
| 产品化补强 | 补自动报表、数据版本管理、监控告警 | `evaluation_report`、benchmark 管理、部署规范 |

### 11.2 算力资源

当前算力资源按单张 RTX 4090 24GB 规划，适合小规模验证，不适合大模型全参训练或大规模 GRPO。

| 资源 | 用途 | 约束 |
|---|---|---|
| RTX 4090 24GB | LoRA/QLoRA SFT、离线 eval、demo 推理 | 优先 1.5B/3B 模型 |
| 同卡错峰 vLLM | adapter 评估和 demo 推理 | 训练和推理不要同时运行，避免 OOM |
| GRPO 小规模试验 | 作为探索项 | 小 batch、小 rollout、小样本，不作为当前主测评依赖 |

当前不建议作为主线开展：

```text
13B+ 训练
full fine-tune
大规模 GRPO
训练和推理长期并发
```

### 11.3 存储和数据资源

| 资源 | 用途 |
|---|---|
| `morpheus-eval-v1` | 固定评测数据集，包含 `holdout / regression / replay` |
| `train.jsonl` | LoRA/QLoRA SFT 训练数据 |
| `trace_transition_eval.jsonl` | 可选，用于 trace 决策能力评测 |
| 原始业务/trace 数据 | 用于构造训练样本和后续自进化 |
| MatrixOne / SQLite | 存储任务、评测、release、adapter 元数据 |
| MinIO / 对象存储 | 存储 adapter、训练产物、评测报告 |
| 日志存储 | 保存训练日志、runtime metrics、release metrics |

当前资源条件下，建议先围绕 1.5B/3B 小模型 LoRA/QLoRA 完成 Morpheus 闭环验证，再评估是否扩大模型规模或引入更重的 GRPO 训练。

### 11.4 预算要求

当前阶段以小规模验证为主，不单独规划大规模算力预算。优先复用现有 RTX 4090 24GB、现有数据库/对象存储和已有业务/trace 数据。

若进入真实试点，再按训练频率、模型规模和数据量评估新增 GPU、存储和标注预算。

## 12. 后续增强项

本节内容不作为当前测评的必做前置条件。当前测评可以先基于已有 eval、replay、gate、release 工具半自动完成。

如果后续需要提升测评自动化、复现性，或支撑论文实验和产品化验收，建议逐步补充以下能力。

| 优先级 | 功能 | 目的 |
|---|---|---|
| P0 | `evaluation_report` 自动汇总 | 汇总 job、dataset、adapter、eval、replay、gate、release |
| P0 | 评测集版本管理自动化 | 在手工冻结数据快照的基础上，进一步自动管理 holdout/regression/replay 版本、hash 和泄漏检查 |
| P1 | trace action 分层指标 | 分别统计 plan、route、tool、retrieval、db、stop/retry/reflect |
| P1 | evolution round 报表 | 记录每轮数据、训练、adapter、发布和收益 |
| P1 | 基础成本指标 | 记录训练耗时、eval 耗时、GPU hours |
