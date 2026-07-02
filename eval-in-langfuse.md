# Eval in Langfuse：AI Agent 评估体系实战

> **系列**：eval-in-action — AI Agent 可观测性与评估工程实践  
> **适用版本**：Langfuse v3+ (Python SDK v3+ / JS/TS SDK v4+)

---

## 目录

1. [为什么需要系统性评估](#1-为什么需要系统性评估)
2. [Langfuse 评估体系总览](#2-langfuse-评估体系总览)
3. [核心概念](#3-核心概念)
4. [评估闭环：Online × Offline](#4-评估闭环online--offline)
5. [评估方法详解](#5-评估方法详解)
   - 5.1 [LLM-as-a-Judge](#51-llm-as-a-judge)
   - 5.2 [Code Evaluators（代码评估器）](#52-code-evaluators代码评估器)
   - 5.3 [人工标注（Annotation Queues）](#53-人工标注annotation-queues)
   - 5.4 [Score via API/SDK](#54-score-via-apisdk)
6. [Dataset 与 Experiment 工作流](#6-dataset-与-experiment-工作流)
   - 6.1 [构建 Dataset](#61-构建-dataset)
   - 6.2 [运行 Experiment（SDK）](#62-运行-experimentsdk)
   - 6.3 [CI/CD 回归测试](#63-cicd-回归测试)
7. [Agent 场景的 Trace-level 与 Observation-level 评估](#7-agent-场景的-trace-level-与-observation-level-评估)
8. [Score 类型与分析](#8-score-类型与分析)
9. [完整实战示例：客服 Agent Eval Pipeline](#9-完整实战示例客服-agent-eval-pipeline)
10. [最佳实践与常见陷阱](#10-最佳实践与常见陷阱)
11. [参考资料](#11-参考资料)

---

## 1. 为什么需要系统性评估

AI Agent 的输出具有**非确定性**，传统单测（assert input → output）无法覆盖：

- 工具调用是否选择正确（Tool Selection Quality）
- 多步推理的中间步骤是否合理（Trajectory Correctness）
- 最终答案是否忠实于检索内容（Faithfulness）
- 用户满意度是否随版本迭代提升

没有系统性评估，Agent 的"感觉不错"只是幻觉。Langfuse 提供了一套**从生产 Trace 到离线实验再到 CI/CD 回归**的完整评估闭环。

---

## 2. Langfuse 评估体系总览

```
┌─────────────────────────────────────────────────────────────┐
│                     Langfuse Eval Loop                      │
│                                                             │
│  Deploy ──► Online Trace ──► Online Monitor                 │
│                │                    │                       │
│                │          LLM-as-Judge / Feedback           │
│                │                    │                       │
│                ▼                    ▼                       │
│         Build Dataset ◄─── Add edge cases from prod        │
│                │                                            │
│                ▼                                            │
│          Run Experiment ──► Offline Evaluate                │
│                │          (Judge / Code / Human)            │
│                ▼                                            │
│         Compare & Decide ──► Deploy or Iterate              │
└─────────────────────────────────────────────────────────────┘
```

核心理念：**评估是一个持续循环，而非一次性步骤。**

- **Online（生产）**：对实时 Trace 自动打分，发现生产中的异常
- **Offline（实验）**：在 Dataset 上跑 Experiment，阻断回归上线

---

## 3. 核心概念

| 概念 | 说明 |
|---|---|
| **Trace** | 一次完整的 Agent 执行记录，包含所有 Span |
| **Observation（Span）** | Trace 内的单个操作：LLM 调用、工具调用、检索等 |
| **Score** | 评估结果的通用数据对象，可附加到 Trace / Observation / Session / Dataset Run |
| **Dataset** | 测试用例集合，每个 Item 包含 `input` 和可选的 `expected_output` |
| **Dataset Item** | 单个测试用例 |
| **Experiment Run** | 在 Dataset 上执行一次 Task 并收集所有 Output 的过程 |
| **Evaluator** | 对 Experiment 结果或生产 Trace 打分的函数 |
| **Evaluation Rule** | 定义"对哪些数据、以什么频率、用哪个 Evaluator 打分"的规则 |

---

## 4. 评估闭环：Online × Offline

### Online Evaluation（生产监控）

```
生产流量 → Trace 入库 → 按 Filter 匹配 Observation → 异步评估队列 → Score 写回
```

- 支持**采样**（如 5% 抽评），控制成本
- 推荐精度：**Observation-level**（比 Trace-level 快数十倍，秒级完成）
- 典型用途：监控幻觉率、Toxicity、Response Relevance 的趋势变化

### Offline Evaluation（实验对比）

```
Dataset ──► run_experiment() ──► Traces ──► LLM-as-Judge / Code Eval ──► Scores
                                  │
                               比较多次 Experiment Run，决定是否上线
```

- 支持 `ground_truth`（expected_output）参与评估
- 与 CI/CD 集成，自动阻断分数回退的 PR

---

## 5. 评估方法详解

### 5.1 LLM-as-a-Judge

**核心原理**：用一个强 LLM（Judge 模型）对另一个 LLM 应用的输出按 Rubric 打分。

> 研究表明，强力 Judge 模型（GPT-5 级别）与人类评估者的一致率可达 80–90%，与评估者间人工一致率相当。

**Score 类型选择**：

| 类型 | 适用场景 | 示例 |
|---|---|---|
| `NUMERIC` | 连续性质量判断 | `helpfulness: 0.0–1.0` |
| `CATEGORICAL` | 离散标签 | `correct / partially_correct / incorrect` |
| `BOOLEAN` | 二元决策 | `is_out_of_scope: true/false` |

**完整配置流程**（UI 操作）：

1. 进入 **Evaluators** 页面 → `+ Set up Evaluator`
2. 设置默认 Judge 模型（需配置 LLM Connection，要求支持 Structured Output）
3. 选择 **Managed Evaluator**（内置 Hallucination / Toxicity / Helpfulness 等）或编写 **Custom Evaluator**（支持 `{{input}}` `{{output}}` `{{ground_truth}}` 变量）
4. 配置评估目标：Live Observations / Live Traces / Experiment Data
5. 设置 Filter 和采样率
6. 映射变量并 Preview

**Observation-level vs Trace-level**：

```
推荐：Observation-level（Python v3+ / JS/TS v4+）
- 评估完成时间：秒级
- 精度：可精确过滤 LLM Call / Tool Call / Retrieval

Legacy：Trace-level
- 评估完成时间：分钟级（整条 Trace 需完成后才触发）
- 适合需要跨多步全局上下文的评估
```

**编程方式配置（API）**：

```python
# 1. 创建 Evaluator
POST /api/public/evaluators
{
  "name": "helpfulness-judge",
  "prompt": "Rate helpfulness from 1-5...\nInput: {{input}}\nOutput: {{output}}",
  "outputDefinition": {"type": "NUMERIC", "min": 1, "max": 5}
}

# 2. 创建 Evaluation Rule（绑定到 Live Observations）
POST /api/public/evaluation-rules
{
  "evaluatorName": "helpfulness-judge",
  "target": "OBSERVATION",
  "samplingRate": 0.1,
  "filter": {"observationType": "GENERATION", "traceTags": ["customer-support"]}
}
```

### 5.2 Code Evaluators（代码评估器）

用于**确定性逻辑**检查，无需 LLM，成本为零，速度极快。

```python
# 示例：检查 JSON 格式合规性
def json_format_evaluator(output: str) -> dict:
    try:
        parsed = json.loads(output)
        has_required_fields = all(k in parsed for k in ["action", "params"])
        return {"score": 1.0 if has_required_fields else 0.0, "reason": "format check"}
    except json.JSONDecodeError:
        return {"score": 0.0, "reason": "invalid JSON"}
```

典型用途：
- 输出格式校验（JSON Schema、正则）
- 业务规则校验（价格区间、合规词汇）
- 字符串精确匹配（exact match for structured tasks）

### 5.3 人工标注（Annotation Queues）

当自动评估不够时，将 Trace 推入人工标注队列：
- 构建 **Ground Truth** 数据集
- 对 LLM-as-Judge 结果抽检校准
- 团队协作标注，支持自定义标注维度

### 5.4 Score via API/SDK

```python
from langfuse import get_client

langfuse = get_client()

# 对 Trace 打分
langfuse.score(
    trace_id="<trace_id>",
    name="user_satisfaction",
    value=4.5,
    comment="User gave thumbs up"
)

# 对 Observation 打分
langfuse.score(
    trace_id="<trace_id>",
    observation_id="<observation_id>",
    name="retrieval_relevance",
    value=0.87
)
```

---

## 6. Dataset 与 Experiment 工作流

### 6.1 构建 Dataset

**方式一：从生产 Trace 添加**

```python
langfuse.create_dataset_item(
    dataset_name="customer-support-eval",
    input={"messages": [{"role": "user", "content": "退款流程是什么？"}]},
    expected_output={"answer": "登录账户 → 订单详情 → 申请退款，3-5 个工作日到账"},
    source_trace_id="<production_trace_id>",  # 关联生产 Trace
)
```

**方式二：程序化生成合成数据**

```python
langfuse.create_dataset(
    name="agent-tool-calling/v1",
    description="工具调用准确性测试集",
    metadata={"version": "v1", "created": "2026-07"},
    # 可选 JSON Schema 校验
    input_schema={
        "type": "object",
        "properties": {"query": {"type": "string"}},
        "required": ["query"]
    }
)
```

**Dataset Folders**（用 `/` 命名组织层级）：

```
agent-evals/
├── tool-calling/v1
├── rag-faithfulness/v2
└── multi-turn/baseline
```

### 6.2 运行 Experiment（SDK）

```python
from langfuse import get_client
import anthropic

langfuse = get_client()
client = anthropic.Anthropic()

def my_agent(*, item, **kwargs):
    """被测 Agent 逻辑"""
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        messages=[{"role": "user", "content": item.input["query"]}]
    )
    return response.content[0].text

# 获取 Dataset（支持版本锁定）
dataset = langfuse.get_dataset("agent-tool-calling/v1")

# 运行 Experiment
result = dataset.run_experiment(
    name="claude-opus-4-5-baseline",
    description="Baseline with Claude Opus 4.5",
    task=my_agent,
    # 关联 LLM-as-Judge Evaluator（在 UI 中已配置）
    evaluators=["tool-selection-accuracy", "helpfulness-judge"]
)

print(f"Experiment URL: {result.url}")
```

### 6.3 CI/CD 回归测试

```yaml
# .github/workflows/eval.yml
name: Agent Eval Regression

on:
  pull_request:
    branches: [main]

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Experiment
        env:
          LANGFUSE_SECRET_KEY: ${{ secrets.LANGFUSE_SECRET_KEY }}
          LANGFUSE_PUBLIC_KEY: ${{ secrets.LANGFUSE_PUBLIC_KEY }}
        run: python scripts/run_eval.py
      
      - name: Check Score Regression
        run: |
          python scripts/check_regression.py \
            --baseline "main-latest" \
            --current "${{ github.sha }}" \
            --metric "tool-selection-accuracy" \
            --threshold 0.85
```

---

## 7. Agent 场景的 Trace-level 与 Observation-level 评估

多步 Agent（Planner → Tool Call → Summarizer）的评估粒度选择：

```
Trace（整条 Agent 执行）
├── Span: planner-llm          ← 评估"计划质量"（Observation-level）
├── Span: tool-call-search     ← 评估"工具选择准确性"（Observation-level）
├── Span: tool-call-calculator ← 评估"参数正确性"（Observation-level）
└── Span: final-response-llm   ← 评估"回答质量 + 忠实度"（Observation-level）
```

**组合评估策略**（Compositional Evaluation）：

```
同一 Trace → 多个 Evaluator 并行运行
- Toxicity → 只评估 final-response-llm
- Retrieval Relevance → 只评估 tool-call-search
- Helpfulness → 只评估 final-response-llm
```

**设置 Observation 过滤器**（确保 Evaluator 只打目标 Span）：

```
Filter:
  observationType = GENERATION
  observationName = "final-response-llm"
  traceTags contains "customer-support"
```

---

## 8. Score 类型与分析

### Score 数据模型

```python
Score(
    trace_id: str,
    observation_id: Optional[str],   # 精确到 Span
    name: str,                         # e.g. "helpfulness"
    value: float | str | bool,
    data_type: "NUMERIC" | "CATEGORICAL" | "BOOLEAN" | "TEXT",
    comment: Optional[str],            # Judge 的 reasoning
    source: "API" | "LLM_EVAL" | "ANNOTATION" | "CODE_EVAL"
)
```

### Score Analytics

在 Langfuse 仪表板中可以：
- 按时间趋势查看各指标均值/分布
- 对比不同 Experiment Run 的 Score 分布
- 设置告警：当某指标低于阈值时触发通知

---

## 9. 完整实战示例：客服 Agent Eval Pipeline

以下是一个端到端的评估工程实践：

```python
"""
eval_pipeline.py
客服 Agent 评估 Pipeline：
1. 从生产 Trace 中抽取边缘用例 → 构建 Dataset
2. 运行 Experiment（对比 v1 / v2 Prompt）
3. LLM-as-Judge 评估 Helpfulness + Faithfulness
4. 输出回归报告
"""

import os
from langfuse import get_client, Langfuse
from langfuse.decorators import observe
import anthropic

langfuse = get_client()
client = anthropic.Anthropic()

DATASET_NAME = "customer-support-eval/v2"
PROMPT_V1 = "你是一个客服助手，请简洁回答用户问题。"
PROMPT_V2 = "你是一个专业客服助手，请以友好、准确的方式回答用户问题，必要时引用具体政策。"


# Step 1: 构建 Dataset（首次运行）
def build_dataset():
    langfuse.create_dataset(name=DATASET_NAME)
    
    test_cases = [
        {"query": "退款需要多少天？", "expected": "3-5个工作日"},
        {"query": "如何修改收货地址？", "expected": "在订单页面点击修改"},
        {"query": "优惠券可以叠加使用吗？", "expected": "不可叠加，每单限用一张"},
    ]
    
    for case in test_cases:
        langfuse.create_dataset_item(
            dataset_name=DATASET_NAME,
            input={"query": case["query"]},
            expected_output={"answer": case["expected"]}
        )


# Step 2: 定义被测 Agent
@observe(name="customer-service-agent")
def run_agent(query: str, system_prompt: str) -> str:
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=512,
        system=system_prompt,
        messages=[{"role": "user", "content": query}]
    )
    return response.content[0].text


# Step 3: 运行 Experiment
def run_experiment(prompt_version: str, system_prompt: str):
    dataset = langfuse.get_dataset(DATASET_NAME)
    
    def task(*, item, **kwargs):
        return run_agent(item.input["query"], system_prompt)
    
    result = dataset.run_experiment(
        name=f"cs-agent-{prompt_version}",
        task=task,
        # 在 Langfuse UI 中预先配置好 evaluator
        evaluators=["helpfulness-judge", "faithfulness-judge"]
    )
    return result


# Step 4: 执行对比
if __name__ == "__main__":
    # build_dataset()  # 首次运行时取消注释
    
    result_v1 = run_experiment("v1", PROMPT_V1)
    result_v2 = run_experiment("v2", PROMPT_V2)
    
    print(f"V1 Experiment: {result_v1.url}")
    print(f"V2 Experiment: {result_v2.url}")
    print("在 Langfuse UI 中对比两次 Experiment Run 的 Score 分布")
```

---

## 10. 最佳实践与常见陷阱

### ✅ 最佳实践

1. **从小 Dataset 开始**：10–30 个精心设计的测试用例比 1000 个随意用例更有价值
2. **优先 Observation-level**：生产场景用 Observation-level Evaluator，比 Trace-level 快数十倍
3. **采样控制成本**：生产流量只抽 5–10% 评估，节省 LLM Judge 成本（每次评估约 $0.01–$0.10）
4. **Ground Truth 驱动**：离线评估时尽量提供 `expected_output`，让 Judge 有参照系
5. **Rubric 要具体**：避免"评估质量1-5分"这类模糊 Rubric，应描述每分对应的具体行为
6. **校准 LLM Judge**：先用 30–50 条人工标注数据验证 Judge 与人类的一致率
7. **CI/CD 设置阈值**：在 PR 阶段自动检查关键指标，如 `tool_accuracy >= 0.85` 才允许合并

### ⚠️ 常见陷阱

| 陷阱 | 后果 | 解决方案 |
|---|---|---|
| 只评估最终输出，不评估中间步骤 | 无法定位 Agent 失败根因 | 对每个关键 Span 设置独立 Evaluator |
| Dataset 与生产分布偏差过大 | 实验分数好但生产质量差 | 持续从生产 Trace 中补充边缘用例 |
| 所有 Trace 都打分（无采样） | LLM Judge 成本失控 | 设置 5–10% 采样率 |
| LLM Judge 使用被测同款模型 | 自我评估偏差（self-serving bias） | 使用不同厂商或更强的模型作为 Judge |
| Rubric 模糊 | Score 噪音大，趋势不可信 | 用 Few-shot 示例校准 Rubric |

---

## 11. 参考资料

- [Langfuse Evaluation Overview](https://langfuse.com/docs/evaluation/overview)
- [Langfuse Core Concepts](https://langfuse.com/docs/evaluation/concepts)
- [LLM-as-a-Judge Documentation](https://langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge)
- [Datasets & Experiments](https://langfuse.com/docs/datasets/overview)
- [LLM Evaluation 101: Best Practices](https://langfuse.com/blog/2025-03-04-llm-evaluation-101-best-practices-and-challenges)
- [AI Agent Observability with Langfuse](https://langfuse.com/blog/2024-07-ai-agent-observability-with-langfuse)
