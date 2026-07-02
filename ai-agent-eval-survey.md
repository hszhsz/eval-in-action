# AI Agent 评估方法综合调研报告

> **系列**：eval-in-action — AI Agent 可观测性与评估工程实践
> **定位**：不限定单一项目、不限定参考代码的方法论综合调研
> **成文时间**：2026-07

---

## 摘要

大语言模型（LLM）驱动的 Agent 正在从"能对话"走向"能干活"——它会自主规划、调用工具、多步推理、与环境交互。这类系统的输出具有**非确定性**、**多步性**和**开放性**，传统软件测试的 `assert(input) == output` 范式几乎完全失效。如何科学地衡量一个 Agent "好不好、稳不稳、值不值"，已经成为 Agent 工程化落地的核心瓶颈。

本报告不局限于任何单一工具或框架，而是从方法论层面综合梳理 AI Agent 评估（Eval）的完整版图：**评估什么（对象与维度）**、**怎么评（方法与指标）**、**用什么基准（Benchmark 生态）**、**如何落地（工程闭环）**，以及**当前的核心挑战与未来趋势**。目标是为读者建立一张可迁移、跨项目的 Agent 评估心智地图。

---

## 目录

1. [为什么 Agent 评估如此困难](#1-为什么-agent-评估如此困难)
2. [评估对象：从最终答案到全链路轨迹](#2-评估对象从最终答案到全链路轨迹)
3. [评估维度分类学](#3-评估维度分类学)
4. [评估方法全景](#4-评估方法全景)
   - 4.1 [确定性/统计评估器](#41-确定性统计评估器)
   - 4.2 [LLM-as-a-Judge](#42-llm-as-a-judge)
   - 4.3 [模型化评估方法：G-Eval / QAG / DAG](#43-模型化评估方法g-eval--qag--dag)
   - 4.4 [人工标注](#44-人工标注)
5. [三个评估粒度：端到端 / 轨迹级 / 组件级](#5-三个评估粒度端到端--轨迹级--组件级)
6. [专项场景：RAG 评估三元组](#6-专项场景rag-评估三元组)
7. [Benchmark 生态全景](#7-benchmark-生态全景)
8. [Online × Offline：生产评估与离线实验的闭环](#8-online--offline生产评估与离线实验的闭环)
9. [工程落地：数据集、实验与 CI/CD 回归](#9-工程落地数据集实验与-cicd-回归)
10. [核心挑战](#10-核心挑战)
11. [趋势与建议](#11-趋势与建议)
12. [参考资料](#12-参考资料)

---

## 1. 为什么 Agent 评估如此困难

相较于传统 ML 模型评估（固定测试集 + 准确率）和单轮 LLM 评估（prompt → completion），Agent 评估的难度陡增，根源在于四个特性：

| 特性 | 含义 | 对评估的冲击 |
|---|---|---|
| **非确定性** | 相同输入，多次运行的推理路径、工具参数、最终答案都可能不同 | 单次运行的分数不可靠，必须引入重复性/方差度量 |
| **多步性** | 一次任务涉及规划、多次工具调用、反思、重试 | 只看终态无法定位失败根因；需要评估中间步骤 |
| **开放性** | 正确答案往往不唯一（多条路径都能成功） | 精确匹配失效，需要语义级/裁判式判断 |
| **交互性** | 与外部环境（网页、文件系统、数据库、用户）动态交互 | 评估需要可复现的环境与状态校验，而非静态数据 |

一个直观的例子：让 Agent "帮我预订明天飞北京的最便宜航班"。它可能先查日历、再搜航班、比价、最后下单。评估要回答的不只是"最终订到没有"，还包括：**它选对工具了吗？参数（日期、城市）填对了吗？走的路径高效吗？花了多少 token 和几步？如果重跑 5 次，成功率稳定吗？** 这些问题共同定义了 Agent 评估的复杂度。

---

## 2. 评估对象：从最终答案到全链路轨迹

现代 Agent 的一次执行会产生一条完整的**轨迹（Trajectory / Trace）**——由一系列有序的观测（Observation / Span）组成：

```
Trace（一次完整的 Agent 执行）
├── Span: 规划（Planner LLM）        → 计划是否合理、可执行？
├── Span: 工具调用（Search Tool）     → 选对工具了吗？参数对吗？
├── Span: 检索（Retriever）           → 检索到的上下文相关吗？
├── Span: 工具调用（Calculator）      → 结果被正确使用了吗？
├── Span: 反思/重试（Reflection）     → 错误恢复是否有效？
└── Span: 最终回答（Response LLM）     → 答案正确、忠实、有用吗？
```

评估的对象因此分为两大类：

- **结果（Outcome）**：最终交付物是否达成目标。这是最直观、最重要的头条指标，但**信息量有限**——它告诉你"错了"，却不告诉你"错在哪"。
- **过程（Trajectory）**：达成结果的路径质量。它能捕捉"用错工具、用错顺序、幻觉步骤、绕远路"等结果指标看不到的失败模式，是**定位根因、优化 Agent 的关键**。

**核心洞见**：Agent 评估正在经历从"只看终态"到"看轨迹与中间状态"的范式迁移。AgentBoard 提出的 **progress rate（细粒度进度率）**、BFCL v3 引入的 **state-based 校验**，都是这一趋势的代表——它们承认"部分完成"也是有价值的信号。

---

## 3. 评估维度分类学

无论用什么工具，Agent 的可衡量维度可归纳为以下几类。一套成熟的评估体系应覆盖多个维度而非单一指标：

| 维度 | 衡量内容 | 典型指标 |
|---|---|---|
| **任务成功率** | 最终目标是否达成 | Success Rate、% Resolved、Pass Rate |
| **轨迹/步骤正确性** | 动作序列是否正确合理 | Trajectory Match、Progress Rate、Plan Adherence |
| **工具调用质量** | 函数名、参数、结构是否正确 | Tool Correctness、Argument Correctness、AST Match |
| **答案质量** | 正确性、相关性、忠实度、有用性 | Correctness、Relevancy、Faithfulness、Helpfulness |
| **安全与合规** | 是否有害、越权、泄露、偏见 | Toxicity、Bias、PII Leakage、越权率 |
| **效率成本** | 资源消耗 | Token 数、API 费用、步数、延迟 |
| **可靠性/一致性** | 重复运行的稳定性 | pass^k、方差、置信区间 |

> **实践要点**：成本（cost）、步数（steps）、延迟（latency）已从"附属信息"上升为**一等评估指标**。HAL、vals.ai、OSWorld-Human 等排行榜都将它们与成功率并列——一个成功率 90% 但每次花费 \$2、耗时 10 分钟的 Agent，在多数生产场景下不如成功率 85% 但快且便宜的方案。

---

## 4. 评估方法全景

评估方法本质上是"如何给一次 Agent 输出打分"的函数。按是否依赖 LLM、是否需要真值，可分为四大类，它们各有取舍，实践中通常**组合使用**。

### 4.1 确定性/统计评估器

不依赖 LLM，**成本为零、速度极快、完全可复现**，适合有明确判据的场景。

- **执行式校验（Execution-based）**：跑单元测试、查数据库终态、执行生成的代码。这是**最可靠**的评估范式，SWE-bench（跑测试）、OSWorld（校验脚本）、τ-bench（比对数据库状态）均采用。
- **精确/模糊匹配**：Exact Match、正则、JSON Schema 校验，适合结构化输出与工具参数。
- **经典 NLP 指标**：
  - **F1**：token 级重叠，适合抽取式 QA。
  - **BLEU**：基于 n-gram 精确率，机器翻译经典指标。
  - **ROUGE**：基于 n-gram 召回，摘要常用。
  - **BERTScore**：用上下文 embedding 余弦相似度，捕捉语义，优于 BLEU/ROUGE 但依赖领域。

**取舍**：确定性指标快、便宜、可复现，是回归测试的骨架；但它们**无法判断答案是否真正"正确、忠实、有用"**——两个语义相同但措辞不同的答案，Exact Match 会判错。

```python
# 示例：确定性工具参数校验（成本为零）
def tool_arg_evaluator(actual_call, expected_call) -> dict:
    name_ok = actual_call["name"] == expected_call["name"]
    args_ok = actual_call["args"] == expected_call["args"]
    return {"score": 1.0 if (name_ok and args_ok) else 0.0,
            "reason": f"name={name_ok}, args={args_ok}"}
```

### 4.2 LLM-as-a-Judge

用一个强 LLM（裁判模型）按 Rubric 对另一个模型的开放式输出打分，是**灵活性最高、覆盖开放式任务的主力方法**。

**三种范式**：
- **Pointwise（单点打分）**：直接对单个答案给分（如 helpfulness: 1–5）。
- **Pairwise（成对比较）**：给定问题和两个答案，选更优者。稳定性通常优于单点打分。
- **Reference-guided（参考引导）**：提供参考答案再评判，适合数学/推理题。

**可靠性**：Zheng et al.（MT-Bench & Chatbot Arena, 2023）证明 GPT-4 级裁判与人类偏好一致率超过 **80%**，与人类评估者之间的一致率相当。这是 LLM-as-a-Judge 被广泛采用的理论基础。

**主要偏见与缓解**：

| 偏见 | 表现 | 缓解手段 |
|---|---|---|
| **位置偏见** | 偏好特定位置（如第一个）的答案 | 交换位置各评一次，两次都胜才判赢；或随机化位置 |
| **冗长偏见** | 偏好更长的回答，即便不够准确 | Rubric 明确长度无关；控制长度变量 |
| **自我偏好偏见** | 偏好自己/同族模型的输出 | 用**不同厂商或更强**的模型做裁判 |
| **推理局限** | 数学/逻辑题上判断易错 | 参考引导 + Chain-of-Thought |

> **警示**：近期偏见测试（JudgeBiasBench、FairJudge 等）显示，前沿模型在进阶偏见场景下错误率可超过 50%。裁判在生产环境往往**无法维持论文中的理想一致率**，因此必须先用 30–50 条人工标注数据**校准**裁判，并持续监测其与人类的一致率。

### 4.3 模型化评估方法：G-Eval / QAG / DAG

在朴素 LLM-as-a-Judge 之上，学术界与工程界发展出更结构化的评估方法，以降低噪音、提升与人类的相关性：

- **G-Eval**（Liu et al., EMNLP 2023）：用 GPT-4 + **思维链（CoT）+ 表单填写（form-filling）**。先让 LLM 根据评测标准自动生成评测步骤，再按步骤逐维度打分，还可用输出 token 概率加权得到连续分数。在摘要任务上与人工评分的 Spearman 相关达 **0.514**，显著超越传统指标。
- **QAG（Question-Answer Generation）**：用于忠实度/幻觉检测。流程：从答案中生成问题 → 分别用源文档和答案作答 → 比对两组答案（token 级 F1）→ 不一致即暗示幻觉。相较 n-gram 重叠，与人工忠实度判断的相关性显著更高。
- **DAG（Deep Acyclic Graph）**：把评估拆解成一棵**决策树**，每个节点是一个确定性或 LLM 判断，逐层导向最终评分。它把"模糊的整体打分"拆成"一串可控的小判断"，兼具 LLM 的灵活性与代码评估器的可控性，适合评分标准复杂、需要强解释性的场景。

### 4.4 人工标注

当自动评估不足或需要建立"黄金标准"时，将轨迹推入**人工标注队列（Annotation Queue）**：
- 构建 **Ground Truth** 数据集，作为离线实验的参照系。
- 对 LLM-as-Judge 的结果抽检、校准，测量裁判与人类的一致率。
- 团队协作标注，支持自定义维度。

人工标注是所有自动化评估的"锚点"，成本高但不可或缺——**没有人工校准的自动裁判，只是一个"自信但可能错误"的黑盒**。

---

## 5. 三个评估粒度：端到端 / 轨迹级 / 组件级

一套完整的 Agent 评估应同时在三个粒度展开，形成"由粗到细"的定位能力：

```
① 端到端（End-to-End）    评估整条 Trace：任务是否成功？总成本多少？
        │  "结果导向"——回答"好不好"
        ▼
② 轨迹级（Trajectory）    评估动作序列：路径是否高效合理？工具选择恰当吗？
        │  "过程导向"——回答"路径对不对"
        ▼
③ 组件级（Component）     评估单个 Span：这一步的工具/子模块出错了吗？
           "根因导向"——回答"错在哪一步"
```

**轨迹匹配的确定性方法**：
- **Exact Match**：轨迹与参考完全一致（最严格）。
- **In-order Match**：参考工具调用按顺序出现，允许额外步骤。
- **Any-order Match**：参考工具都被调用，顺序不限（最宽松）。

**组合评估策略（Compositional Evaluation）**：同一条 Trace 上并行运行多个 Evaluator，各自只打目标 Span——例如 Toxicity 只评最终回答，Retrieval Relevance 只评检索步骤，Tool Correctness 只评工具调用。这种"分而治之"的策略是定位复杂 Agent 失败根因的关键。

> **工程建议**：优先采用 **Observation-level（观测级）** 评估而非 **Trace-level（整条链路级）**。前者可精确过滤到目标 Span、秒级完成；后者需整条 Trace 结束才触发、分钟级完成，仅在需要跨步全局上下文时才必要。

---

## 6. 专项场景：RAG 评估三元组

检索增强生成（RAG）是 Agent 最常见的能力之一，其评估已形成成熟的"三元组/四元组"范式（以 Ragas 框架为代表），能**精确定位失败在检索还是生成阶段**：

| 指标 | 衡量 | 定位 | 计算思路 |
|---|---|---|---|
| **Faithfulness（忠实度）** | 答案是否忠于检索上下文（查幻觉） | 生成端 | 抽取答案中所有 claim，统计可由上下文推得的比例 |
| **Answer Relevancy（答案相关性）** | 答案与问题的相关度 | 生成端 | 由答案反向生成问题，与原问题算 embedding 余弦相似度 |
| **Context Precision（上下文精确率）** | 检索结果中相关内容的比例 | 检索端 | 相关上下文 / 检索总数 |
| **Context Recall（上下文召回率）** | 应检索到的内容被检索到的比例 | 检索端 | 参考答案 claim 中被上下文支持的比例 |

Ragas 的价值在于它**大部分指标 reference-free（无需人工真值）**，用 LLM 自身完成 claim 抽取与核验，大幅降低了 RAG 评估的标注成本。

---

## 7. Benchmark 生态全景

公开 Benchmark 是横向对比模型/Agent 能力的标尺。按能力域分类，当前主流基准如下（**注：具体 SOTA 分数随模型、scaffold、版本剧烈变动，引用时务必标注来源与日期**）：

| Benchmark | 能力域 | 规模 | 评分方法 |
|---|---|---|---|
| **SWE-bench / Verified** | 软件工程（改 bug） | 2294 / 500 题 | **执行式**：应用补丁跑单元测试，% Resolved |
| **GAIA** | 通用助手（推理+多模态+工具） | 466 题 | 准确匹配（quasi-exact match） |
| **AgentBench** | 多环境自主决策 | 8 个环境 | 各环境任务专属成功指标加权 |
| **τ-bench / τ²-bench** | 工具-用户交互（客服） | retail/airline/telecom | **状态式**：比对数据库终态；pass^k 衡量可靠性 |
| **WebArena / VisualWebArena** | 网页导航 | 812 / 910 题 | **功能正确性**：字符串匹配 + 后端状态校验 |
| **OSWorld** | 计算机使用（GUI） | 369 题 | **执行式**：初始化+校验脚本返回 reward |
| **BFCL**（Berkeley Function Calling） | 工具/函数调用 | v1–v4 演进 | **AST 匹配** + 可执行校验 + 相关性检测；v3 引入状态式 |
| **MLE-bench** | 机器学习工程 | 75 个 Kaggle 赛 | 按奖牌门槛评判；获奖牌比例 |
| **WebVoyager** | 端到端网页 agent | 643 任务/15 站点 | GPT-4V 自动评测（与人类约 85% 一致） |
| **AgentBoard** | 分析式多任务评测 | 1013 环境 | **Progress Rate（细粒度进度率）** + 可视化 |
| **ToolBench / ToolLLM** | 工具使用 | 16464 个 API | ToolEval：Pass Rate + Win Rate（pairwise） |

**从 Benchmark 生态可读出的三条趋势**：
1. **评测范式三分**：执行式（最可靠）> 结构化匹配 > LLM-as-Judge（最灵活但争议最大）。
2. **从终态到过程、从单次到可靠性**：progress rate 与 pass^k 的出现，说明业界不再满足于"一次成功率"。
3. **Benchmark 会"成熟/退役"**：SWE-bench Verified、OSWorld-Verified、WebArena Verified 都是对原基准脏样本/歧义的修订——**任何 Benchmark 都不是绝对真理，需理解其构造与局限**。

---

## 8. Online × Offline：生产评估与离线实验的闭环

成熟的 Agent 评估不是一次性动作，而是一个贯穿开发与生产的**持续闭环**：

```
      ┌──────────── 部署上线 ────────────┐
      ▼                                  │
  Online 评估（生产监控）                 │
   · 对实时 Trace 采样打分（5–10%）        │
   · 监控幻觉率/Toxicity/相关性趋势        │
   · 从生产流量捞出边缘 case ──────────┐  │
      │                              │  │
      ▼                              ▼  │
  发现回归/异常              补充进 Dataset │
      │                              │  │
      └──► Offline 评估（离线实验）◄──┘  │
            · 在 Dataset 上跑 Experiment  │
            · LLM-Judge / Code / Human 打分│
            · 对比多次 Run，决定是否上线 ──┘
```

- **Online（生产）**：对真实流量**采样**评估（控制成本），及时发现生产中的质量漂移。推荐观测级评估以获得秒级反馈。
- **Offline（离线）**：在固定 Dataset 上跑实验，支持 `ground_truth` 参照，用于版本对比与**阻断回归上线**。

两者相互喂养：生产中的失败 case 持续补充离线数据集，使测试集分布逼近真实分布——这是避免"实验分数好、生产质量差"的关键机制。

---

## 9. 工程落地：数据集、实验与 CI/CD 回归

**数据集（Dataset）**是离线评估的基石。构建原则：
- **从小开始**：10–30 个精心设计的用例，价值远超 1000 个随意用例。
- **持续补充**：从生产 Trace 中不断加入边缘 case，防止分布偏移。
- **提供真值**：尽量附 `expected_output`，让裁判有参照系。
- **分层组织**：按能力域（tool-calling / rag / multi-turn）组织版本化目录。

**实验（Experiment）**：在数据集上运行被测 Agent，收集输出并批量打分，产出可对比的 Score 分布。伪代码示意（与具体框架无关）：

```python
dataset = get_dataset("agent-tool-calling/v1")

def task(item):
    return my_agent(item.input["query"])   # 被测 Agent

result = dataset.run_experiment(
    name="claude-opus-4-5-baseline",
    task=task,
    evaluators=["tool-selection-accuracy", "helpfulness-judge"]
)
```

**CI/CD 回归门禁**：将评估接入 PR 流程，用阈值自动阻断质量回退：

```yaml
# 在 PR 阶段自动检查关键指标
- name: Check Score Regression
  run: |
    python check_regression.py \
      --metric "tool-selection-accuracy" \
      --threshold 0.85   # 低于 0.85 则阻断合并
```

这把"感觉不错"变成"数据说话"，是 Agent 从 Demo 走向生产的工程分水岭。

---

## 10. 核心挑战

| 挑战 | 本质 | 应对方向 |
|---|---|---|
| **非确定性** | 相同输入下输出与分数波动 | 多次重复取均值/方差；报告置信区间；引入 pass^k |
| **裁判偏见** | 位置/冗长/自我偏好偏见 | 换厂商裁判、位置交换、CoT、人工校准 |
| **真值稀缺** | 高质量人工标注昂贵 | reference-free 方法（Ragas/QAG）、合成数据、生产回捞 |
| **裁判校准漂移** | 未校准裁判"自信但错误" | 小规模标注对齐、持续测一致率、漂移时重校准 |
| **成本失控** | 前沿裁判 + 多维度 + 全量评估 | 生产采样 5–10%、确定性指标兜底、分级评估 |
| **可复现性危机** | 大量已发表评测统计上不可靠 | 固定裁判版本、报告方差、独立复现 |
| **只看终态** | 无法定位失败根因 | 轨迹级 + 组件级评估，对关键 Span 独立打分 |
| **分布偏移** | 测试集与生产分布脱节 | 持续从生产补充边缘 case |

其中，**非确定性**与**裁判可靠性**是最根本的两道坎：前者动摇了"单次分数"的意义，后者动摇了"自动裁判"的可信度。任何严肃的 Agent 评估都必须正面处理这两者，而非假装它们不存在。

---

## 11. 趋势与建议

**趋势**：
1. **评估粒度下沉**：从端到端成功率，走向轨迹级、组件级、状态级评估。
2. **可靠性成为一等公民**：pass^k、方差、置信区间取代单次成功率。
3. **成本/延迟并入核心指标**：能力不再是唯一维度，"性价比"成为共识。
4. **评估方法结构化**：从朴素打分走向 G-Eval / DAG / QAG 等可控、可解释的方法。
5. **在线离线闭环工程化**：生产监控与离线实验双向喂养，接入 CI/CD 门禁。

**给实践者的建议**：
1. **多维度而非单指标**：至少覆盖 成功率 + 轨迹质量 + 成本 + 可靠性。
2. **确定性优先，裁判兜底**：能用执行式/精确匹配的绝不用 LLM 裁判；开放式任务才上裁判。
3. **裁判必须校准**：上线前用人工标注验证一致率，用异厂模型防自我偏好。
4. **从小数据集起步，持续从生产补充**：质量 > 数量，分布对齐 > 规模。
5. **把评估接进 CI/CD**：设阈值门禁，让回归在合并前被拦住。
6. **理解 Benchmark 的构造与局限**：公开分数用于横向参考，落地要靠贴合自身场景的私有数据集。

> **一句话总结**：Agent 评估的成熟度，取决于你能否回答"它错在哪一步、重跑还稳不稳、值不值这个成本"——而不仅仅是"它这次对没对"。

---

## 12. 参考资料

**方法与理论**
- Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena (Zheng et al., NeurIPS 2023) — https://arxiv.org/abs/2306.05685
- G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment (Liu et al., EMNLP 2023) — https://arxiv.org/abs/2303.16634
- QAGS: Asking and Answering Questions to Evaluate Factual Consistency (Wang et al., ACL 2020) — https://aclanthology.org/2020.acl-main.454.pdf
- RAGAS: Automated Evaluation of Retrieval Augmented Generation — https://arxiv.org/html/2309.15217v1
- Ragas 官方文档 — https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/

**轨迹与组件级评估**
- LangSmith Trajectory Evals — https://docs.langchain.com/langsmith/trajectory-evals
- Arize AX Agent Trajectory Evaluations — https://arize.com/docs/ax/evaluate/evaluators/trace-and-session-evals/trace-level-evaluations/agent-trajectory-evaluations
- Confident AI: LLM Agent Evaluation Complete Guide — https://www.confident-ai.com/blog/llm-agent-evaluation-complete-guide

**Benchmark**
- SWE-bench — https://www.swebench.com/ ；arXiv 2310.06770 — https://arxiv.org/abs/2310.06770
- SWE-bench Verified (OpenAI) — https://openai.com/index/introducing-swe-bench-verified
- GAIA — https://arxiv.org/abs/2311.12983
- AgentBench — https://arxiv.org/abs/2308.03688 ；https://github.com/THUDM/AgentBench
- τ-bench / τ²-bench — https://arxiv.org/abs/2406.12045 ；https://github.com/sierra-research/tau2-bench
- WebArena — https://arxiv.org/abs/2307.13854 ；https://webarena.dev
- VisualWebArena — https://arxiv.org/abs/2401.13649
- OSWorld — https://arxiv.org/abs/2404.07972 ；https://os-world.github.io
- BFCL (Berkeley Function Calling Leaderboard) — https://gorilla.cs.berkeley.edu/leaderboard.html
- MLE-bench — https://arxiv.org/abs/2410.07095 ；https://github.com/openai/mle-bench
- WebVoyager — https://arxiv.org/abs/2401.13919
- AgentBoard — https://arxiv.org/abs/2401.13178 ；https://hkust-nlp.github.io/agentboard
- ToolLLM / ToolBench — https://arxiv.org/abs/2307.16789 ；https://github.com/OpenBMB/ToolBench

**挑战与可靠性**
- LLM-as-a-Judge Reliability & Bias — https://www.adaline.ai/blog/llm-as-a-judge-reliability-bias
- LLM Judge Calibration — https://deepchecks.com/llm-judge-calibration-automated-issues
- LangChain: LLM-as-a-Judge — https://www.langchain.com/resources/llm-as-a-judge

**本系列相关文档**
- Eval in Langfuse — https://github.com/hszhsz/eval-in-action/blob/main/eval-in-langfuse.md
- Eval in DeepEval — https://github.com/hszhsz/eval-in-action/blob/main/eval-in-DeepEval.md
