# MOI 整体 + 功能级 Benchmark 方案

> 目标：证明 MOI 在 AI+DATA 一体化和各功能模块上，相对于开源/主流大厂产品的性能与效果优势。

---

## 一、总体架构 + 竞品选择依据

Benchmark 分两层执行（竞品选择均经过 2026-07 最新 web 搜索验证）：

```
平台层（端到端）
  ├─ AI+DATA 全链路：数据接入 → 治理 → 知识库 → AI 应用
  ├─ 对应产品：Databricks、Dify（开源自托管）、百度千帆、阿里百炼
  └─ 评测维度：端到端准确率 / 全链路时延 / TCO

功能层（各模块独立对表）
  ├─ Astra → LangGraph / CrewAI / Dify Agent
  ├─ Memoria → Mem0 / LangMem
  ├─ 文档解析 → MinerU / Marker / LlamaParse / Docling / PaddleOCR
  ├─ RAG → Dify RAG / LlamaIndex / FastGPT
  ├─ NL2SQL → Vanna.AI / Chat2DB-GLM / DAIL-SQL / Databricks AI/BI
  └─ Morpheus → RAGAS / DeepEval / EvalScope（ModelScope）
```

**竞品选择依据（2026-07 web search 验证）：**
- **移除 Snowflake Cortex**：Snowflake 是云数仓架构，非统一 AI+DATA 平台，与 MOI 架构差异过大，对标价值低
- **移除 AutoGen**：更偏向研究型多 Agent 对话，企业级 Agent 运行时不如 LangGraph + CrewAI 主流
- **移除 MemGPT(Letta)**：仍偏实验性，生产级选 Mem0（YC 支持、SOC2/HIPAA）+ LangMem（LangChain 生态）
- **移除 Zep**：专注时间线/图谱记忆，与 Memoria 版本控制差异化能力不直接对标
- **增加 Docling / PaddleOCR**：2026-06 CSDN 解析横向评测覆盖了 MinerU/Docling/Marker/Unstructured/PaddleOCR/LlamaParse 六款，Docling 和 PaddleOCR 是重要补充
- **增加 FastGPT**：国内知识库问答场景 RAG 最强开源选项，专注度高于 Dify RAG
- **增加 Chat2DB-GLM**：2026 国内企业 NL2SQL 落地主流选择之一
- **增加 EvalScope**：阿里 ModelScope 的评测框架，2026-06 已支持 MCP/GAIA，国内评测场景比 LangSmith 更接地气
- **增加阿里百炼**：与百度千帆并列，国内大厂 AI 平台双雄，Coze 是 SaaS 闭源不适合私有化对比

---

## 二、对表产品矩阵

### 2.1 平台层对表

| 维度 | MOI | Databricks | Dify | 百度千帆 | 阿里百炼 |
|------|:---:|:----------:|:----:|:--------:|:--------:|
| 数据底座 | **MatrixOne (HTAP+Vector)** | Delta Lake + Spark | 无（外接） | 无（外接） | 无（外接） |
| AI 模型服务 | **MatrixGenesis** | Mosaic AI/Model Serving | 外接 LLM API | ERNIE+外接 | 通义+外接 |
| 数据处理 | **MatrixPipeline** | Delta Live Tables | 无 | 无 | 无 |
| 知识库/RAG | **MatrixCopilot KB** | Databricks RAG Studio | 内置 RAG（v2.0） | 内置 RAG | 内置 RAG |
| Agent 编排 | **Astra + Copilot** | Agent Framework | Agent 工作流 | AppBuilder | 百炼 Agent |
| 私有化部署 | **✅ 原生支持** | ❌ 仅公有云 | ✅ 开源自托管 | ❌ 仅公有云 | ❌ 仅公有云 |
| MCP 协议 | **✅ 原生支持** | 部分支持 | ✅ 原生支持 | 部分支持 | 部分支持 |

> **说明**：MOI 是唯一同时自有数据底座 + AI 模型服务 + 数据处理管线 + 知识库 + Agent 编排 + 支持私有化部署的一体化平台。Databricks 是功能最接近的竞品但不支持私有化。Dify 是开源中最接近的但缺数据底座和数据处理。

### 2.2 功能层对表

| 模块 | MOI 产品 | 对标产品（开源/国产） | 对标产品（商业/国际） |
|------|---------|---------------------|---------------------|
| Agent 运行时 | **Astra** | LangGraph / CrewAI / Dify Agent | Coze（字节，但仅SaaS） |
| Agent 记忆 | **Memoria** | Mem0 / LangMem | — |
| 企业 Agent 评测 | **Morpheus** | RAGAS / DeepEval / EvalScope（阿里） | LangSmith |
| 文档解析 | **MatrixPipeline** | MinerU / Docling / Marker / PaddleOCR | LlamaParse / Azure Doc Intelligence |
| RAG | **MatrixCopilot KB** | Dify RAG（v2.0 知识管道+图引擎）/ FastGPT | — |
| NL2SQL | **MatrixCopilot ChatBI** | Vanna.AI / Chat2DB-GLM / DAIL-SQL | Databricks AI/BI |

**竞争情报（2026-07 web search 发现）：**

| 发现项 | 对 MOI 的影响 |
|--------|--------------|
| **Dify v2.0 beta 已发布**（2025-09），引入知识管道（Knowledge Pipeline）和基于队列的图引擎，RAG 能力大幅升级 | Dify 是 MOI 平台层最直接的开源竞品，RAG 对标时必须用 v2.0 以上版本 |
| **Coze v3.0**（2026-06）支持多人多 Agent 协作、行业技能包，已接入 Claude Code/Codex CLI/OpenClaw | Coze 是 Astra 在"低代码 Agent 平台"维度的主要竞品，但闭源 SaaS 无法私有化，与 Astra 的企业级私有化定位差异明显 |
| **Vanna.AI 存在严重安全漏洞 CVE-2024-5565**（CVSS 8.1），提示注入可导致 SQL 数据库远程代码执行 | MOI NL2SQL 的安全设计可作为差异化卖点 |
| **MinerU/Marker/Docling/PaddleOCR/LlamaParse 六款解析工具横向评测**（CSDN 2026-06-28）已有完整测试方法论和数据集 | 解析 Benchmark 可直接引用该评测的测试集，节省自建成本 |
| **Mem0 已获 YC 支持**，通过 SOC2/HIPAA 认证，提供 K8s/私有云/气隙环境部署 | Mem0 是 Memoria 最强的开源对标，其企业级认证和部署选项值得关注 |
| **EvalScope**（阿里 ModelScope 2026-06）已支持 MCP Server、GAIA Benchmark，是国内评测框架首选 | 比 LangSmith 更贴近国内客户需求，适合作为 Morpheus 评测方法论的对标参照 |
| **国内 Agent 平台分工明确**：Coze（字节/零代码/个人）、Dify（开源/最广生态）、千帆（百度云）、百炼（阿里云）、元器（腾讯/微信生态） | 平台层对标选择 Dify（开源可私有化）+ 千帆/百炼（国产大厂），Coze 仅适合功能级对标 |

---

## 三、各模块 Benchmark 方案

### 3.1 Astra（Agent 运行时）

**评测维度：**

> 核心原则：Astra vs LangGraph/CrewAI/Dify Agent 对比的是 **Agent 编排平台能力**（harness、loop engineering、state management、可观测性）。所有系统使用**同一个 LLM**（如 Qwen3.5），消除模型差异对结果的影响。

**🟦 平台层（核心对比，同模型下比编排能力）：**

| 维度 | 指标 | 测试方法 | 为什么是平台层 |
|------|------|---------|:------------:|
| 任务完成率 | 端到端成功率 % | 50 个预设企业 Agent 任务，逐任务判定 | 编排逻辑决定任务能否走完 |
| 循环/重试工程 | 失败自动重试 + 退避策略有效率 | 注入工具调用失败，检查恢复路径 | Loop engineering 是 Astra 核心 |
| 状态管理 | 多步对话状态一致性 | 5 轮连续对话后检查状态未丢失/混淆 | Agent 运行时与模型无关 |
| 可审计性 | 全链路追踪日志完整性 | 检查组审计日志是否覆盖每一步 | 企业必需，非模型能力 |
| 可回滚性 | Agent 会话回滚后的状态一致性 | 执行→回滚→比对状态 | 平台基础设施能力 |
| MCP 协议兼容 | MCP 标准工具集 20 个工具调用成功率 | 同一套 MCP Server 挂载到各系统 | 协议层兼容性 |
| 权限隔离 | 多租户隔离正确性 | 测试跨租户数据泄露（RBAC 越权） | 企业平台能力 |
| 工具编排灵活度 | 条件分支/循环/并行/子任务的覆盖率 | 30 个合成场景的编排能力 | 平台功能完整度 |

**🟥 模型层（不对比 — 各系统使用同一 LLM，无差异）：**
- ❌ 回答质量/准确性 — 同模型下结果一致
- ❌ 意图识别准确率 — 取决于 LLM 不是平台
- ❌ 工具选择的正确性 — LLM 决定，平台只负责执行

**测试集准备：**
- 基于**中芯国际文档解析**和**华虹 MOVE 预测**两个真实场景，提取 20 个典型 Agent 任务
- 增加 30 个合成任务覆盖：多工具链式调用（5）、条件分支（5）、循环迭代（5）、异常恢复（5）、长上下文（5）、并行子任务（5）
- 每个任务定义明确的成功标准（结构化 JSON）

**执行方案：**
1. 所有系统挂载**同一个 LLM**（Qwen3.5-397B-A17B 或同等档次）
2. Astra 部署一套 + LangGraph 部署一套 + CrewAI 部署一套（同等资源配置）
3. 同一测试管线对三个系统发送相同的 50 个任务
4. 自动化判定结果并记录时延

---

### 3.2 Memoria（Agent 记忆系统）

**评测维度：**

> 核心原则：Memoria vs Mem0/LangMem 对比的是**记忆系统架构能力**（版本控制、检索架构、冲突管理）。所有系统使用**同一 Embedding 模型**。

**🟦 平台层（核心对比，同 Embedding 下比架构能力）：**

| 维度 | 指标 | 测试方法 | 为什么是平台层 |
|------|------|---------|:------------:|
| 版本控制 | 快照/回滚正确率 | 100 次快照创建+回滚操作 | **Memoria 核心差异化**，竞品基本不支持 |
| 冲突检测 | 矛盾记忆识别率 | 注入 20 对矛盾记忆 | 记忆治理能力，与 Embedding 无关 |
| 跨 Agent 共享 | 记忆共享后的一致性 | 2 Agent 读写同一记忆空间 | 分布式记忆架构 |
| 写入吞吐 | 记忆写入 QPS | 批量写入 10000 条记忆 | 存储引擎性能 |
| 检索时延 | P50/P95 检索耗时 | 不同记忆容量下（100/1K/10K/100K） | 索引架构 |
| 全文检索+向量混合 | 混合查询的 recall@5 | 同时使用 SQL 条件+向量检索 | MatrixOne 融合能力 |
| 记忆类型系统 | 类型种类/粒度/可扩展性 | 对比支持的记忆类型数量和语义 | 架构设计的完备性 |

**🟥 模型层（不对比 — 同 Embedding 下无差异）：**
- ❌ 语义搜索质量 NDCG@10 — 取决于 Embedding 模型，不是记忆系统
- ❌ 记忆检索准确率 Recall@5 — 同 Embedding 下结果一致

**测试集准备：**
- 从公司内部 wiki/文档中提取 500 条真实记忆（脱敏后）
- 合成 9500 条填充记忆（达到 10K 规模）
- 准备 100 个检索查询，标注标准答案和相关性排序
- 准备 20 对矛盾记忆用于冲突检测

**执行方案：**
1. Memoria 部署（MatrixOne 后端）
2. Mem0 / Zep 各部署一套（同为 MySQL 后端，公平对比）
3. 同一测试管线写入相同 10K 记忆
4. 执行 100 个查询，记录 recall/latency
5. 测试版本控制功能（快照/回滚）

---

### 3.3 Morpheus（企业 Agent 评测框架）

Morpheus 是方法论层产品，Benchmark 侧重评测方法论本身的效果。

**评测维度：**

| 维度 | 指标 | 测试方法 |
|------|------|---------|
| 评测覆盖度 | 评测维度数量 | 统计 Morpheus gate 覆盖的评测维度 |
| 评测准确率 | 人工 vs Morpheus 判定一致性 | 200 个 Agent 输出结果的比对 |
| 门控拦截率 | 坏输出被门控拦截的比例 | 注入 50 个有意引入错误的 Agent 输出 |
| 误报率 | 好输出被门控误拦截的比例 | 50 个正确的 Agent 输出 |
| 回归效率 | 版本回归时间 | N 个测试用例的回归执行时间 |

**对表产品：** LangSmith、RAGAS、DeepEval

**测试集准备：**
- 复用中芯国际文档解析的 10 维度/300 页评测数据集
- 准备 200 个 Agent 输出样本（100 好/100 坏）
- 标注标准答案（人工标注作为 ground truth）

---

### 3.4 文档解析

> **解析是纯算法对比**，无需分平台/模型层。MOI 的解析管线（MinerU+PaddleOCR+VLM 多引擎并行 + 100+ 参数开关）本身就是**差异化算法优势**（中芯国际 85 vs 58 分），所有维度都可以比。

**评测维度（全部可对比，纯算法能力）：**

| 维度 | 指标 | 测试方法 | ⭐ 优先级 |
|------|------|---------|:---------:|
| 文字准确率 | CER/WER | 比对解析结果 vs 原文 | 🔴 |
| 版面还原度 | 层级结构 F1 | 标题/正文/表格/图片的层级还原 | 🔴 |
| 表格解析 | 表格结构准确率 | 行列合并、跨页表格正确率 | 🔴 |
| 图片提取 | 图片位置/标题匹配率 | 图片位置标注与原文一致性 | 🔴 |
| 公式识别 | 公式 Markdown 转换准确率 | Latex/AsciiMath 转换正确率 | 🟡 |
| 流程图识别 | 流程图结构还原度 | 节点/连线/文本的还原 | 🔴（制造业刚需） |
| 处理速度 | P50/P95 页/秒 | 1000 页文档批量处理 | 🟡 |
| 大文件支持 | 最大支持页数 / 内存峰值 | 逐步增大文件至 OOM | 🟡 |

**对表产品：** MinerU、Docling、Marker、PaddleOCR、LlamaParse、Azure Document Intelligence

> 📌 **可引用现成评测**：CSDN 2026-06-28 已有 MinerU/Docling/Marker/Unstructured/PaddleOCR/LlamaParse 六款工具的横向评测（含测试方法论和数据集），可直接复用其测试集和指标定义，节省自建成本。

**测试集准备：**
- 复用中芯国际 15 份/300 页真实业务文档评测数据集
- 补充：带流程图的文档（5 份）、带复杂表格的文档（5 份）、手写扫描件（5 份）、PDF 扫描件（5 份）、PPT（3 份）
- 标注标准答案（文字 + 版面 + 表格结构 + 图片位置）

**执行方案：**
1. 所有解析系统部署在同等 GPU 资源上（1×H20）
2. 同一测试集输入所有系统
3. 自动化比对：用 MOI 的 parsing benchmark suite（micro-v1）统一打分

---

### 3.5 RAG

**评测维度：**

> 核心原则：RAG 对比的是**检索架构能力**（分块策略、混合召回、重排序），不是 LLM 生成能力。所有系统使用**同一个 LLM** 做生成，使用**同一 Embedding** 做向量化。

**🟦 平台层（核心对比，同模型/同 Embedding 下比检索架构）：**

| 维度 | 指标 | 测试方法 | 为什么是平台层 |
|------|------|---------|:------------:|
| 混合检索效果 | 向量+关键词+元数据 vs 纯向量 | 在有元数据筛选条件下的 Recall@5 提升率 | 三路召回是 MOI 独有架构 |
| 分块策略效果 | 不同分块策略下的检索准确率 | 同一文档用不同分块粒度 + 重叠策略对比 | RAG 管线设计 |
| 重排序效果 | Rerank 后 Top5 准确率提升 | 对比 rerank 前 / rerank 后 | 检索管线能力 |
| 检索时延 | P50/P95 检索耗时（不含 LLM 生成） | 不同知识库规模下（1K/10K/100K） | 索引性能 |
| 引用可追溯性 | 回答中引用是否可点开原文段落 | 检查引用链完整率 | 企业可信度要求 |
| 元数据过滤 | 带元数据条件检索的准确率 | 按日期/分类/部门筛选后检索 | 企业级检索灵活性 |

**🟥 模型层（不对比 — 同一 LLM 下生成质量一致）：**
- ❌ 生成质量 Faithfulness / Answer Relevance — 同 LLM 下结果一致
- ❌ 拒绝回答质量 — LLM 决定，不是 RAG 架构

**对表产品：** Dify RAG（v2.0+）、FastGPT、LlamaIndex

**测试集准备：**
- 从金盘知识管理案例中提取 20 个业务查询
- 从华虹/中鼎等制造业场景提取 20 个技术查询
- 合成 10 个跨文档推理查询（需多文档交叉引用才能回答）
- 构建 1000 份文档的测试知识库（每份文档标注：分类/部门/密级元数据）

---

### 3.6 NL2SQL

**评测维度：**

> 核心原则：NL2SQL 对比的是**架构能力**（Schema Linking、上下文管理、SQL 策略），不是 LLM 的 SQL 生成能力。所有系统使用**同一个 LLM**。

**🟦 平台层（核心对比，同模型下比策略架构）：**

| 维度 | 指标 | 测试方法 | 为什么是平台层 |
|------|------|---------|:------------:|
| Schema Linking 准确率 | 正确识别所需表和字段的比例 | 50 个查询，检查各系统选中的表和字段 | 元数据管理架构 |
| 上下文管理 | 多轮对话中 Schema 和 SQL 上下文保持 | 5 轮连续对话，检查每轮依赖关系 | 对话状态管理 |
| 复杂 SQL 策略 | JOIN/NESTED/WINDOW 的策略选择合理性 | 20 个复杂查询，对比各系统选择的 SQL 策略 | SQL 生成策略 |
| 错误处理 | 错误 SQL 的检测和重试机制 | 注入语法错误/表不存在等异常 | 工程健壮性 |
| 安全管控 | SQL 注入防护/权限绕过检测 | 测试注入攻击场景 | 🟢 MOI 优势方向 |
| 时延排布 | P50/P95 预处理耗时（不含 LLM 推理） | 测量 Schema 读取+prompt 构建耗时 | 管线效率 |

**🟥 模型层（不对比 — 同 LLM 下 SQL 生成质量差异小）：**
- ❌ SQL 正确率 Execution Accuracy — 同 LLM 下差异主要在 prompt 策略而非平台
- ❌ 逻辑等价率 ESM — 同上

> ⚠️ 如果 MOI ChatBI 有**自研的 Schema Linking 算法或 SQL 策略优化**（算法优势），则可以将 SQL 正确率加入平台层对比。否则只比平台层。

**对表产品：** Vanna.AI、Chat2DB-GLM、DAIL-SQL（方法论）、Databricks AI/BI

> Vanna.AI 有已知 CVE-2024-5565（CVSS 8.1）安全漏洞，Benchmark 时需隔离环境执行，可作为 MOI 安全设计论据。
> Chat2DB-GLM 适合中文场景 NL2SQL。DAIL-SQL 是 prompt 方法论，适合策略层对比。

**测试集准备：**
- 使用开源 Spider/Bird 数据集（英文）
- 构建制造业场景的 50 个中文查询（适配 MOI 的 MatrixOne 方言）
- 构建含复杂业务的查询（库存/采购/生产计划相关）

---

## 四、对标品测试可行性分析

### 4.1 可自建测试环境的（开源/可私有化部署）

| 对标品 | 部署方式 | 资源需求 | 部署复杂度 | 可行性 |
|--------|---------|---------|:----------:|:------:|
| LangGraph | pip install | 4C8G，无 GPU | ⭐ 低 | ✅ |
| CrewAI | pip install | 4C8G，无 GPU | ⭐ 低 | ✅ |
| Dify（自托管） | Docker Compose | 4C8G+50G 磁盘 | ⭐⭐ 中 | ✅ |
| FastGPT | Docker Compose | 4C8G，需 Embedding API | ⭐⭐ 中 | ✅ |
| Mem0 | pip + Vector DB | 4C8G+Vector DB（可租） | ⭐⭐ 中 | ✅ |
| LangMem | pip install | 4C8G | ⭐ 低 | ✅ |
| MinerU | pip/Docker + GPU | 8C32G+1×H20 | ⭐⭐⭐ 高（GPU 驱动） | ✅ 但 GPU 占用高 |
| Docling | pip install | 8C16G | ⭐ 低 | ✅ |
| Marker | pip + GPU | 8C16G+GPU | ⭐⭐ 中 | ✅ |
| PaddleOCR | pip + GPU | 8C16G+GPU | ⭐⭐ 中 | ✅ |
| Vanna.AI | pip + Vector DB | 4C8G+LLM API | ⭐⭐ 中 | ⚠️ 需注意 CVE-2024-5565 |
| Chat2DB-GLM | Docker | 8C16G+GPU | ⭐⭐ 中 | ⚠️ 需要 GPU |
| RAGAS | pip install | 4C8G+LLM API | ⭐ 低 | ✅ |
| DeepEval | pip install | 4C8G+LLM API | ⭐ 低 | ✅ |
| EvalScope | pip install | 4C8G+LLM API | ⭐ 低 | ✅ |

### 4.2 仅支持 API 调用的（商业化/SaaS 对标品）

这些产品无法自建，必须通过 API/SDK 调用，需要设计统一的测试管线。

| 对标品 | 测试方式 | 计费模式 | 成本估算 | 数据隐私 | 可行性 |
|--------|---------|---------|:--------:|:--------:|:------:|
| **LlamaParse** | HTTP API 上传文档 → 返回 Markdown/JSON | $0.003/页 | 300页≈$1，共 ~$10 | 文档会上传至第三方 | ✅ 成本极低 |
| **Azure Doc Intelligence** | SDK/API 上传文档 → 返回结构化结果 | $0.0015/页（S0 层） | 300页≈$0.5，共 ~$5 | Azure 中国区合规 | ✅ 成本极低 |
| **Databricks AI/BI** | SQL 仓库 + Genie API | 按计算资源 $/h + 存储 | ~$500-1500（2周） | 数据需上传至 Databricks | ⚠️ 设置复杂，成本高 |
| **LangSmith** | SDK 追踪 + 数据集管理 | Free 层 1000 项/月，之后 $0.01/项 | ~$100-300 | 仅元数据，可脱敏 | ✅ 有免费层 |
| **Dify Cloud** | API key + HTTP API | Free 层 200 条/月，Pro $59/月 | ~$59-200 | 数据经 Dify 服务器 | ✅ 成本可控 |
| **Coze** | API + Bot SDK | Free 层额度 | ~$0-100 | 数据经字节服务器 | ✅ 免费层够用 |
| **百度千帆** | API 调用 | 按 token 计费 | ~$50-200 | 数据经百度服务器 | ✅ 成本可控 |
| **阿里百炼** | API 调用 | 按 token 计费 | ~$50-200 | 数据经阿里服务器 | ✅ 成本可控 |

### 4.3 统一测试管线设计

所有测试（无论自建还是 API）共享同一套测试数据格式和评测脚本：

```
测试集（标准格式 JSON）
  ├─ 输入：query / document / task
  ├─ 标准答案：expected_output / golden_answer
  └─ 元数据：category / difficulty / domain

         ↓ 分发（Test Harness）
         ↓
  ┌──────────────┬──────────────┬──────────────┐
  │  MOI（内部API） │  开源（自建）   │  商业（外部API） │
  │  HTTP调用      │  SDK调用      │  REST API调用  │
  └──────┬───────┴──────┬───────┴──────┬───────┘
         ↓              ↓              ↓
  ┌─────────────────────────────────────────────┐
  │         统一结果收集器（Collector）             │
  │  输出格式统一为：{ output, latency, cost,      │
  │    confidence, trace_log }                  │
  └──────────────────┬──────────────────────────┘
                     ↓
  ┌─────────────────────────────────────────────┐
  │         统一评测引擎（Evaluator）               │
  │  按模块加载评测指标 → 逐项打分 → 对比报告        │
  └─────────────────────────────────────────────┘
```

**测试管线开发工作量：约 2 人周**，含测试集加载器（JSON/YAML）、API 适配层（统一请求/响应格式）、结果收集器和评测引擎。

### 4.4 数据隐私处理

| 数据类型 | 开源自建 | 商业 API | 处理方式 |
|---------|:--------:|:--------:|---------|
| 内部文档（中芯/华虹等） | ✅ 安全 | ❌ 不可用 | 商业 API 测试需用**合成数据**替代 |
| 公开文档/开源数据 | ✅ 安全 | ✅ 可用 | 直接使用同一套公开数据 |
| 脱敏后的合成制造业数据 | ✅ 安全 | ✅ 可用 | 去掉客户标识后可用于商业 API |

**建议**：解析/RAG 测试分两组——"内部敏感数据"只测 MOI（不开源不调 API）、"公开/合成数据"测全量对标品。

---

## 五、成本估算

### 5.1 算力成本

| 项目 | 规格 | 周期 | 估算成本 |
|------|------|:----:|:--------:|
| MOI 现有测试环境 | 已有资源 | — | ¥0（复用） |
| 云 GPU 实例（解析用） | 1×H20（如阿里云 GN7i） | 2 周 | ~¥8,000 |
| 云 GPU 实例（MinerU/Marker 用） | 1×H20 | 1 周 | ~¥4,000 |
| 云服务器（开源系统部署） | 4C16G × 3 台 | 3 周 | ~¥3,000 |
| **算力小计** | | | **~¥15,000** |

### 5.2 API 调用成本

| 服务 | 测试量 | 成本估算 |
|------|:------:|:--------:|
| LlamaParse | 300 页 × 3 轮 | ~$3（¥20） |
| Azure Doc Intelligence | 300 页 × 3 轮 | ~$1.5（¥10） |
| Dify Cloud（RAG 测试） | ~5000 API 调用 | ~$100（¥700） |
| 百度千帆（NL2SQL） | ~500 次 LLM 调用 | ~$100（¥700） |
| 阿里百炼（NL2SQL） | ~500 次 LLM 调用 | ~$100（¥700） |
| LangSmith（评测追踪） | ~10000 项 | ~$100（¥700） |
| Databricks AI/BI（可选） | 2 周计算 | ~$1000（¥7,000） |
| **API 小计** | | **~¥9,000（含 Databricks）** |
| **API 小计（不含 Databricks）** | | **~¥2,800** |

### 5.3 人力成本

| 角色 | 投入 | 估算成本 |
|------|:----:|:--------:|
| 测试负责人 | 4 人周 | —（内部团队） |
| 部署工程师 | 2 人周 | — |
| 评测算法工程师 | 3 人周 | — |
| 领域专家（兼职） | 1 人周 | — |
| **人力小计** | **10 人周** | **内部承担** |

### 5.4 总成本汇总

| 方案 | 算力 | API 调用 | 人力 | 合计 |
|------|:---:|:--------:|:---:|:----:|
| **基准方案（不含 Databricks）** | ¥15,000 | ¥2,800 | 内部 | **~¥18,000** |
| **含 Databricks AI/BI** | ¥15,000 | ¥9,800 | 内部 | **~¥25,000** |
| **节约方案（仅开源自建）** | ¥15,000 | ¥0 | 内部 | **~¥15,000** |

> 如果 MOI 现有测试环境已有 GPU 资源可复用，算力成本可降至 ~¥5,000（仅补充云服务器）。

---

## 六、测试排期与资源分配

### 6.1 排期

| 阶段 | 内容 | 周期 | 并行资源 |
|------|------|:----:|:--------:|
| **W1** | 环境搭建 | 1 周 | 3 人 |
| | MOI + 所有开源对标品部署 | | 部署工程师 1 人 |
| | 测试管线开发（Harness + Collector + Evaluator） | | 算法工程师 1 人 |
| | 测试集构建 + 标准答案标注 | | 测试负责人 + 领域专家 |
| **W2** | 文档解析 Benchmark + RAG Benchmark | 1 周 | 2 人 |
| | 解析：MOI vs MinerU/Docling/Marker/PaddleOCR/LlamaParse/Azure | | GPU 独占，算法工程师 |
| | RAG：MOI vs Dify/FastGPT（并行使用同知识库） | | 测试负责人 |
| **W3** | NL2SQL Benchmark + Astra Benchmark | 1 周 | 2 人 |
| | NL2SQL：MOI vs Vanna/Chat2DB-GLM/百度千帆/阿里百炼/Databricks AI/BI | | 算法工程师 |
| | Astra：MOI vs LangGraph/CrewAI/Dify Agent | | 测试负责人 |
| **W4** | Memoria Benchmark + 平台层端到端 | 1 周 | 2 人 |
| | Memoria：MOI vs Mem0/LangMem（记忆检索+版本控制） | | 算法工程师 |
| | 平台层：选 2 个典型场景跑全链路 | | 测试负责人 |
| **W5** | Morpheus + 报告 | 1 周 | 2 人 |
| | Morpheus vs RAGAS/DeepEval/EvalScope（评测方法论对比） | | 算法工程师 |
| | 报告撰写、评审、定稿 | | 测试负责人 |

### 6.2 资源冲突矩阵

```
         W1    W2    W3    W4    W5
GPU      ─     ─     ─
解析GPU        ████
NL2SQL GPU          ████
Astra                          （无 GPU）
Memoria                 ████   （无 GPU）
RAG              ████          （无 GPU）
Morpheus                       ████（无 GPU）
```

> GPU 仅 W2-W3 有冲突（解析 vs NL2SQL），但两者可错开使用（白天解析，晚上 NL2SQL API 调用），**无需额外 GPU**。

---

## 七、报告输出物

| 输出 | 内容 |
|------|------|
| 平台层对比报告 | AI+DATA 全链路评分卡 + TCO 对比 |
| 6 份功能模块报告 | 每模块：对比表 + 雷达图 + 详细数据 + 关键发现 |
| Benchmark 测试集（脱敏） | 可复用的测试数据集，标注标准和评测脚本 |
| 综合结论 | MOI 优势点清单 + 差距点清单 + 改进建议 |
| 行业对标白皮书（可选） | 向客户展示的 Benchmark 方法论和结论摘要 |

---

## 八、核心建议

### 8.1 优先级排序

**P0（必须做，是 MOI 核心差异化能力）：**
1. **文档解析** — 中芯国际已验证 85 vs 58 分优势，扩大测试集巩固。成本极低（LlamaParse/Azure API < $5）
2. **RAG（混合检索）** — 向量+关键词+元数据三路召回是 MOI 独有。自建 Dify 对比即可，无需 API 成本
3. **Memoria（版本控制）** — Git for Memory 是差异化杀手锏。开源自建，零 API 成本

**P1（重要，增强平台说服力）：**
4. **NL2SQL** — MatrixOne 统一 SQL+向量 是技术壁垒。API 成本 ~¥1,400
5. **Astra（可审计性）** — 企业级可控是制造业客户刚需。开源自建，零成本

**P2（品牌/战略层面）：**
6. **Morpheus** — 学术论文层面验证方法论。零额外成本
7. **平台端到端 TCO** — 可选 Databricks 对比（~¥7,000）

### 8.2 不建议测试的项

| 不建议做的 | 原因 |
|-----------|------|
| 与 Databricks/Snowflake 比纯 SQL 性能 | 这不是 MOI 的差异化 |
| 与大模型厂商比模型效果 | MOI 是平台，不是基础模型公司 |
| 一次性全部模块 | 建议分两批：首批 3 个 P0 + 1 个 P1 |
| Coze 全面对标 | 闭源 SaaS，定位差异大，功能层部分参考即可 |
| 所有对标品都用商业版 | 没必要——能用开源自建就用开源，开源不覆盖的才用 API |

### 8.3 建议的首批范围（4 周，~¥15,000）

```
W1 部署：MOI + MinerU + Docling + Marker(解析) + Dify(RAG) + Mem0(Memoria)
W2 解析：MOI vs MinerU/Docling/Marker/PaddleOCR/LlamaParse/Azure API
W3 RAG：MOI vs Dify/FastGPT + Memoria vs Mem0/LangMem
W4 NL2SQL：MOI vs Vanna/Chat2DB-GLM + 报告撰写
```

**首批总成本约 ¥15,000（算力）+ ¥1,000（API 调用）+ 4 人周（内部）= ~¥16,000。**

---

## 附：Benchmark 哲学总结

本方案的核心理念是**同模型比平台，有算法优势才比模型效果**：

```
所有 Benchmark 的第一原则：
"如果所有系统用同一个 LLM，结果会不会一样？"
  → 会一样 → 这是模型层差异，不比
  → 不一样 → 这是平台层/算法层差异，核心对比项

文档解析例外：因为 MOI 的解析管线本身就是算法优势，
所有维度都是有效的对比项。
```

这样做的好处：
1. **结论不依赖特定 LLM** — 换模型不影响平台能力结论
2. **成本可控** — 不需要为每个系统配不同模型，同一 API 足矣
3. **客户易理解** — "同样用 Qwen，MOI 的 RAG 检索准确率高 20%" 比 "MOI 用 A 模型比 B 模型好" 更有说服力
