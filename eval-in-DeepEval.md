# Eval in DeepEval：AI Agent 评估体系实战

> **系列**：eval-in-action — AI Agent 可观测性与评估工程实践  
> **适用版本**：DeepEval v4.0+（Apache 2.0 开源，Python 3.9+）

---

## 目录

1. [DeepEval 是什么](#1-deepeval-是什么)
2. [核心设计理念](#2-核心设计理念)
3. [基础概念：TestCase / Golden / Dataset](#3-基础概念testcase--golden--dataset)
4. [Tracing：让 Agent 可被评估](#4-tracing让-agent-可被评估)
5. [三层评估体系](#5-三层评估体系)
   - 5.1 [推理层 Reasoning Layer](#51-推理层-reasoning-layer)
   - 5.2 [行动层 Action Layer](#52-行动层-action-layer)
   - 5.3 [执行层 Execution Layer](#53-执行层-execution-layer)
6. [通用 LLM 评估指标（非 Agent 专项）](#6-通用-llm-评估指标非-agent-专项)
7. [自定义指标：GEval 与 DAGMetric](#7-自定义指标geval-与-dagmetric)
8. [端到端 vs 组件级评估](#8-端到端-vs-组件级评估)
9. [与 Pytest 集成：`deepeval test run`](#9-与-pytest-集成deepeval-test-run)
10. [CI/CD 回归测试](#10-cicd-回归测试)
11. [生产监控：Confident AI](#11-生产监控confident-ai)
12. [完整实战示例：客服 Agent Eval Pipeline](#12-完整实战示例客服-agent-eval-pipeline)
13. [最佳实践与常见陷阱](#13-最佳实践与常见陷阱)
14. [与 Langfuse 方案对比](#14-与-langfuse-方案对比)
15. [参考资料](#15-参考资料)

---

## 1. DeepEval 是什么

**DeepEval** 是一个开源的 LLM 评估框架（Apache 2.0），定位为"AI Agent 的测试套件"。其核心思路是将 LLM 评估范式与 `pytest` 对齐——**像写单测一样写 Eval**。

> "DeepEval traces every step of your agent into something you can grade, and improve — visible in your terminal, testable in your runner, shippable in your next commit."

**关键数字**：
- 50+ 内置评估指标，覆盖 Agent / RAG / Chatbot / Safety
- 20M+ 日评估量（全球用户规模）
- 支持 10+ 主流 Agent 框架：LangChain、LangGraph、OpenAI Agents SDK、Pydantic AI、CrewAI、Google ADK、Strands、LlamaIndex 等

---

## 2. 核心设计理念

DeepEval 的设计围绕三个原则：

**① Trace-Driven（追踪驱动）**  
Agent 评估不能只看最终输出，必须分析完整执行轨迹（推理步骤 + 工具调用 + 中间决策）。DeepEval 通过 `@observe` 装饰器自动构建 Trace，零延迟。

**② Layer-Specific（按层分评）**  
Agent 在推理层、行动层、执行层各有独特的失败模式，需针对性指标覆盖，而非一个 Score 打天下。

**③ pytest 兼容（开发者优先）**  
通过 `assert_test()` + `deepeval test run` 命令，Eval 代码与普通测试代码完全同构，可直接集成 CI/CD。

---

## 3. 基础概念：TestCase / Golden / Dataset

### LLMTestCase

单次 LLM 交互的评估载体：

```python
from deepeval.test_case import LLMTestCase, ToolCall

test_case = LLMTestCase(
    input="退款需要多久？",
    actual_output="3-5个工作日",
    expected_output="3-5个工作日内到账",   # ground truth（可选）
    context=["退款政策：标准退款3-5工作日"],  # 检索上下文（RAG 场景）
    tools_called=[                          # 实际工具调用
        ToolCall(name="search_policy", input={"query": "退款流程"})
    ],
    expected_tools=[                        # 期望工具调用
        ToolCall(name="search_policy")
    ]
)
```

### Golden

Dataset 中的原始测试用例（只含 input 和可选的 expected_output，**无** actual_output）：

```python
from deepeval.dataset import Golden

golden = Golden(
    input="Book a flight from NYC to London for next Monday",
    expected_output="Flight booked successfully with confirmation number"
)
```

### EvaluationDataset

管理多个 Golden / TestCase 的容器，支持本地定义或从 Confident AI 云端拉取：

```python
from deepeval.dataset import EvaluationDataset, Golden

# 本地定义
dataset = EvaluationDataset(goldens=[
    Golden(input="订单状态查询"),
    Golden(input="申请退货退款"),
    Golden(input="发票开具"),
])

# 从 Confident AI 拉取（需登录）
# dataset = EvaluationDataset()
# dataset.pull(alias="customer-support-eval-v2")
```

---

## 4. Tracing：让 Agent 可被评估

Agent 评估的前提是**让 DeepEval 理解 Agent 的内部结构**。通过 `@observe` 装饰器标注每个组件：

```python
from deepeval.tracing import observe, update_current_span
from openai import OpenAI

client = OpenAI()

@observe(type="tool")
def search_flights(origin: str, destination: str, date: str):
    """Action Layer：与外部系统交互"""
    return [{"id": "FL123", "price": 450}, {"id": "FL456", "price": 380}]

@observe(type="tool")
def book_flight(flight_id: str):
    return {"confirmation": "CONF-789", "flight_id": flight_id}

@observe(type="llm")  # 推理层：在此附加 Action Layer 指标
def call_llm(messages):
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        tools=tools_schema
    )
    return response

@observe(type="agent")  # 顶层 Agent Span
def travel_agent(user_input: str):
    messages = [{"role": "user", "content": user_input}]
    response = call_llm(messages)
    # ... 工具调用逻辑
    return "Booking confirmed"
```

**Span 类型说明**：

| `type` | 对应层 | 说明 |
|---|---|---|
| `"agent"` | 顶层 | 整个 Agent 执行的根 Span |
| `"llm"` | 推理层 | LLM 推理调用，工具选择决策在此发生 |
| `"tool"` | 行动层 | 工具调用执行 |
| `"retriever"` | 行动层 | RAG 检索操作 |
| `"custom"` | 任意 | 自定义业务逻辑 |

> **零延迟**：`@observe` 装饰器仅在评估时激活 Tracing，不影响生产性能。

---

## 5. 三层评估体系

DeepEval 的 Agent 评估指标按三层组织，覆盖 Agent 所有可能的失败模式：

```
┌──────────────────────────────────────────────────────────────┐
│              DeepEval Agent Evaluation Layers                │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Execution Layer（执行层）                            │   │
│  │  TaskCompletionMetric | StepEfficiencyMetric          │   │
│  │  ↑ 分析完整 Agent Trace                              │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │  Reasoning Layer（推理层）                            │   │
│  │  PlanQualityMetric | PlanAdherenceMetric              │   │
│  │  ↑ 分析 Planning 和 CoT 步骤                         │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │  Action Layer（行动层）                               │   │
│  │  ToolCorrectnessMetric | ArgumentCorrectnessMetric    │   │
│  │  ↑ 附加在 LLM Span（工具调用决策点）上               │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### 5.1 推理层 Reasoning Layer

评估 Agent 的**计划生成质量**和**计划执行一致性**。

#### PlanQualityMetric

评估 Agent 生成的计划是否逻辑清晰、完整且高效：

```python
from deepeval.metrics import PlanQualityMetric
from deepeval.dataset import EvaluationDataset, Golden

plan_quality = PlanQualityMetric(
    threshold=0.7,
    model="gpt-4o",
    include_reason=True  # 返回评分理由
)

dataset = EvaluationDataset(goldens=[
    Golden(input="帮我预定下周一从北京到上海的最便宜机票")
])

for golden in dataset.evals_iterator(metrics=[plan_quality]):
    travel_agent(golden.input)
```

**计算公式**：
```
PlanQualityScore = AlignmentScore(Task, Plan)
```

**Score 含义**：LLM Judge 对计划的逻辑性、完整性、高效性综合评分（0–1）。

> 若 Trace 中检测不到显式计划，默认 Pass（Score=1），不会误报。

#### PlanAdherenceMetric

评估 Agent 是否**忠实执行**了自己制定的计划：

```python
from deepeval.metrics import PlanAdherenceMetric

plan_adherence = PlanAdherenceMetric(threshold=0.7, model="gpt-4o")

for golden in dataset.evals_iterator(metrics=[plan_quality, plan_adherence]):
    travel_agent(golden.input)
```

**计算公式**：
```
PlanAdherenceScore = AlignmentScore((Task, Plan), ExecutionSteps)
```

> 建议 `PlanQualityMetric` 和 `PlanAdherenceMetric` 组合使用：好计划 + 忠实执行，才算推理层合格。

### 5.2 行动层 Action Layer

评估 Agent 的**工具选择**和**参数生成**正确性。这两个指标是**组件级（Component-Level）**，直接附加在 LLM Span 的 `@observe` 装饰器上。

#### ToolCorrectnessMetric

```python
from deepeval.metrics import ToolCorrectnessMetric
from deepeval.test_case import ToolCall

tool_correctness = ToolCorrectnessMetric(
    threshold=0.8,
    # 可配置比对维度：默认只比 name，可扩展到 input_params / output
    evaluation_params=["name", "input_parameters"]
)

# 附加到 LLM Span（工具调用决策发生的地方）
@observe(type="llm", metrics=[tool_correctness])
def call_llm(messages):
    return client.chat.completions.create(
        model="gpt-4o", messages=messages, tools=tools_schema
    )
```

配合 Golden 中的 `expected_tools` 字段对比实际调用：

```python
Golden(
    input="搜索明天北京的天气",
    expected_tools=[ToolCall(name="get_weather", input={"city": "Beijing", "date": "tomorrow"})]
)
```

#### ArgumentCorrectnessMetric

评估工具调用的参数是否从上下文中正确提取和生成：

```python
from deepeval.metrics import ArgumentCorrectnessMetric

argument_correctness = ArgumentCorrectnessMetric(threshold=0.8)

@observe(type="llm", metrics=[tool_correctness, argument_correctness])
def call_llm(messages):
    ...
```

> 即便选对了工具，参数错误（如 `{"city": "San Francisco"}` vs 期望 `{"city": "San Francisco, CA, USA"}`）也会导致任务失败，需要独立评估。

### 5.3 执行层 Execution Layer

评估 Agent 的**整体任务完成度**和**执行效率**。这两个是**端到端（End-to-End）**指标，必须通过 `evals_iterator` 或 `@observe` 使用（依赖完整 Trace）。

#### TaskCompletionMetric

```python
from deepeval.metrics import TaskCompletionMetric

task_completion = TaskCompletionMetric(
    threshold=0.7,
    model="gpt-4o",
    include_reason=True
)

dataset = EvaluationDataset(goldens=[
    Golden(input="预定明天最便宜的纽约到洛杉矶机票")
])

for golden in dataset.evals_iterator(metrics=[task_completion]):
    travel_agent(golden.input)
```

LLM Judge 分析整条 Trace，判断用户意图是否被完整实现。

#### StepEfficiencyMetric

```python
from deepeval.metrics import StepEfficiencyMetric

step_efficiency = StepEfficiencyMetric(threshold=0.7)

for golden in dataset.evals_iterator(metrics=[task_completion, step_efficiency]):
    travel_agent(golden.input)
```

检测冗余步骤（如重复调用相同工具、绕弯路等），鼓励 Agent 以最少步骤完成任务。

---

## 6. 通用 LLM 评估指标（非 Agent 专项）

DeepEval 同样覆盖经典 LLM 评估场景：

### RAG 场景

```python
from deepeval.metrics import (
    AnswerRelevancyMetric,  # 答案与问题的相关性
    FaithfulnessMetric,     # 答案是否忠实于检索上下文
    ContextualRelevancyMetric,   # 检索内容与问题相关性
    ContextualPrecisionMetric,   # 检索精度
    ContextualRecallMetric,      # 检索召回率
)
from deepeval.test_case import LLMTestCase

test_case = LLMTestCase(
    input="退款需要多久？",
    actual_output="退款3-5个工作日到账",
    retrieval_context=["公司退款政策：标准退款3-5个工作日，极速退款当天到账"]
)

faithfulness = FaithfulnessMetric(threshold=0.8)
faithfulness.measure(test_case)
print(faithfulness.score, faithfulness.reason)
```

### 多轮对话（Chatbot）

```python
from deepeval.metrics import (
    KnowledgeRetentionMetric,     # 跨轮信息记忆
    RoleAdherenceMetric,          # 角色遵守
    ConversationCompletenessMetric, # 对话完整性
)
```

### 安全性评估

```python
from deepeval.metrics import (
    BiasMetric,     # 偏见检测
    ToxicityMetric, # 毒性内容检测
    PIILeakageMetric, # PII 泄露检测
)
```

---

## 7. 自定义指标：GEval 与 DAGMetric

### GEval（自然语言定义评估标准）

适合主观性评估（语气、风格、专业度等），用自然语言描述评分标准：

```python
from deepeval.metrics import GEval
from deepeval.test_case import LLMTestCaseParams

professional_tone = GEval(
    name="ProfessionalTone",
    criteria="评估回答是否使用了专业、友好的客服语气，避免口语化和情绪化表达",
    evaluation_params=[LLMTestCaseParams.INPUT, LLMTestCaseParams.ACTUAL_OUTPUT],
    threshold=0.7
)

# 也可以用 evaluation_steps 更精细控制评分逻辑
policy_compliance = GEval(
    name="PolicyCompliance",
    evaluation_steps=[
        "检查回答是否引用了正确的退款政策条款",
        "验证金额和时间说明是否与政策一致",
        "确认没有做出超出政策范围的承诺"
    ],
    evaluation_params=[
        LLMTestCaseParams.INPUT,
        LLMTestCaseParams.ACTUAL_OUTPUT,
        LLMTestCaseParams.CONTEXT
    ]
)
```

### DAGMetric（确定性决策树评估）

适合有明确规则逻辑的评估，消除 LLM Judge 的随机性：

```python
from deepeval.metrics.dag import (
    DAGMetric, TaskNode, BinaryJudgementNode, NonBinaryJudgementNode
)

# 构建决策树：先检查格式，再检查内容
dag_metric = DAGMetric(
    name="StructuredOutputQuality",
    dag=TaskNode(
        instructions="检查输出是否为有效 JSON",
        children=[
            BinaryJudgementNode(
                criteria="JSON 格式是否合法",
                children_if_true=[
                    NonBinaryJudgementNode(
                        criteria="JSON 包含必填字段 action 和 params",
                        score=1.0
                    )
                ],
                children_if_false=[]
            )
        ]
    )
)
```

---

## 8. 端到端 vs 组件级评估

DeepEval 的两种评估模式在使用方式上有明确区分：

| 模式 | 挂载方式 | 适用指标 | 分析范围 |
|---|---|---|---|
| **端到端（E2E）** | `evals_iterator(metrics=[...])` | PlanQuality / TaskCompletion / StepEfficiency | 完整 Agent Trace |
| **组件级（Component）** | `@observe(metrics=[...])` | ToolCorrectness / ArgumentCorrectness | 特定 Span |

```python
# 端到端：在迭代器传入，分析整条 Trace
for golden in dataset.evals_iterator(metrics=[task_completion, plan_quality]):
    travel_agent(golden.input)

# 组件级：在装饰器传入，只分析该 Span
@observe(type="llm", metrics=[tool_correctness, argument_correctness])
def call_llm(messages):
    ...
```

**为什么要区分？**  
工具调用的决策发生在 LLM Span 内，如果在整条 Trace 上评估工具调用，会引入其他 Span 的噪音。组件级评估可以精确定位到"LLM 在决策时选错了工具"还是"工具本身逻辑有问题"。

---

## 9. 与 Pytest 集成：`deepeval test run`

DeepEval 的 `assert_test()` 函数可以直接在 pytest 体系中使用：

```python
# test_agent.py
import pytest
from deepeval import assert_test
from deepeval.dataset import EvaluationDataset, Golden
from deepeval.metrics import TaskCompletionMetric, PlanQualityMetric
from deepeval.tracing import observe, update_current_trace

# 加载 Dataset
dataset = EvaluationDataset(goldens=[
    Golden(input="搜索明天北京到上海的最便宜航班并完成预订"),
    Golden(input="查询我上个月的退款记录"),
])

# 被测 Agent
@observe(type="agent")
def my_agent(query: str) -> str:
    update_current_trace(input=query)
    # ... Agent 逻辑
    result = "任务完成"
    update_current_trace(output=result)
    return result

# 参数化测试
@pytest.mark.parametrize("golden", dataset.goldens)
def test_agent(golden: Golden):
    my_agent(golden.input)
    assert_test(
        golden=golden,
        metrics=[
            TaskCompletionMetric(threshold=0.7),
            PlanQualityMetric(threshold=0.7)
        ]
    )
```

运行：

```bash
deepeval test run test_agent.py
```

`deepeval test run` 在 pytest 基础上叠加了 LLM 评估特性：并行运行 Metric、收集 Trace、生成可视化报告。

---

## 10. CI/CD 回归测试

```yaml
# .github/workflows/agent-eval.yml
name: Agent Evaluation Regression

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install deepeval openai anthropic

      - name: Run Agent Evaluations
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          CONFIDENT_API_KEY: ${{ secrets.CONFIDENT_API_KEY }}  # 可选，上报到 Confident AI
        run: |
          deepeval test run tests/test_agent.py \
            --exit-on-first-failure \
            --n-runs 3  # 多次运行取均值，应对 LLM 不确定性
```

**工作机制**：任何指标低于 `threshold`，`assert_test()` 抛出异常 → pytest 标记 FAIL → CI 阻断 PR 合并。

---

## 11. 生产监控：Confident AI

本地 `deepeval test run` 适合开发阶段。生产环境需要**异步评估**（不阻塞 Agent 响应）和**趋势监控**，这由 Confident AI 平台承接：

```python
# 生产环境：用 metric_collection 替换本地 metrics=[...]
@observe(
    type="llm",
    metric_collection="production-agent-metrics"  # 在 Confident AI 中预配置
)
def call_llm(messages):
    ...
```

Agent 运行时，DeepEval 以 OpenTelemetry 风格异步导出 Trace 到 Confident AI，平台负责：
- 按 `metric_collection` 配置异步打分
- 存储历史 Score，可视化趋势
- 告警当指标低于阈值

```bash
# 本地登录连接账号
deepeval login
```

---

## 12. 完整实战示例：客服 Agent Eval Pipeline

```python
"""
eval_pipeline_deepeval.py
客服 Agent 三层评估 Pipeline：
- 推理层：PlanQualityMetric + PlanAdherenceMetric
- 行动层：ToolCorrectnessMetric + ArgumentCorrectnessMetric（组件级）
- 执行层：TaskCompletionMetric + StepEfficiencyMetric
"""

import json
from openai import OpenAI
from deepeval.tracing import observe, update_current_trace, update_current_span
from deepeval.dataset import EvaluationDataset, Golden, get_current_golden
from deepeval.test_case import ToolCall
from deepeval.metrics import (
    TaskCompletionMetric,
    StepEfficiencyMetric,
    PlanQualityMetric,
    PlanAdherenceMetric,
    ToolCorrectnessMetric,
    ArgumentCorrectnessMetric,
)

client = OpenAI()

# ─── 行动层：工具定义 ────────────────────────────────────
@observe(type="tool")
def search_policy(query: str) -> str:
    """检索公司退款/售后政策"""
    return f"查询到政策：{query}相关条款——标准退款3-5工作日"

@observe(type="tool")
def lookup_order(order_id: str) -> dict:
    """查询订单状态"""
    return {"order_id": order_id, "status": "已发货", "estimated_return": "3-5工作日"}

TOOLS_SCHEMA = [
    {
        "type": "function",
        "function": {
            "name": "search_policy",
            "description": "检索公司服务政策，包括退款、售后、保修等",
            "parameters": {
                "type": "object",
                "properties": {"query": {"type": "string", "description": "检索关键词"}},
                "required": ["query"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "lookup_order",
            "description": "根据订单号查询订单状态",
            "parameters": {
                "type": "object",
                "properties": {"order_id": {"type": "string", "description": "订单号"}},
                "required": ["order_id"]
            }
        }
    }
]

# ─── 推理层：LLM 调用（附加 Action Layer 指标）────────────
@observe(
    type="llm",
    metrics=[
        ToolCorrectnessMetric(threshold=0.8),
        ArgumentCorrectnessMetric(threshold=0.8),
    ]
)
def call_llm(messages: list) -> dict:
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        tools=TOOLS_SCHEMA,
    )
    return response

# ─── 顶层 Agent ─────────────────────────────────────────
@observe(type="agent")
def customer_service_agent(user_input: str) -> str:
    update_current_trace(input=user_input)

    messages = [
        {"role": "system", "content": "你是专业的客服助手，使用工具查询信息后再回答用户问题。"},
        {"role": "user", "content": user_input}
    ]

    # 第一轮：LLM 决策调用哪个工具
    response = call_llm(messages)
    msg = response.choices[0].message

    if msg.tool_calls:
        tool_call = msg.tool_calls[0]
        func_name = tool_call.function.name
        args = json.loads(tool_call.function.arguments)

        # 执行工具
        if func_name == "search_policy":
            result = search_policy(**args)
        elif func_name == "lookup_order":
            result = lookup_order(**args)
        else:
            result = "未知工具"

        # 第二轮：LLM 基于工具结果生成最终回答
        messages.append({"role": "assistant", "content": None, "tool_calls": msg.tool_calls})
        messages.append({"role": "tool", "tool_call_id": tool_call.id, "content": str(result)})
        final_response = call_llm(messages)
        output = final_response.choices[0].message.content
    else:
        output = msg.content

    update_current_trace(output=output)
    return output


# ─── 构建评估 Dataset ─────────────────────────────────────
dataset = EvaluationDataset(goldens=[
    Golden(
        input="我的退款需要多久到账？",
        expected_tools=[ToolCall(name="search_policy", input={"query": "退款"})]
    ),
    Golden(
        input="我的订单号是 ORD-12345，当前状态是什么？",
        expected_tools=[ToolCall(name="lookup_order", input={"order_id": "ORD-12345"})]
    ),
    Golden(
        input="如何申请保修？",
        expected_tools=[ToolCall(name="search_policy", input={"query": "保修"})]
    ),
])

# ─── 运行端到端评估 ───────────────────────────────────────
if __name__ == "__main__":
    for golden in dataset.evals_iterator(
        metrics=[
            TaskCompletionMetric(threshold=0.7, include_reason=True),
            StepEfficiencyMetric(threshold=0.7),
            PlanQualityMetric(threshold=0.7),
            PlanAdherenceMetric(threshold=0.7),
        ]
    ):
        result = customer_service_agent(golden.input)
        print(f"Input: {golden.input}")
        print(f"Output: {result}\n")
```

运行：

```bash
deepeval test run eval_pipeline_deepeval.py
```

---

## 13. 最佳实践与常见陷阱

### ✅ 最佳实践

1. **三层组合覆盖**：推理层 + 行动层 + 执行层指标同时运行，不要只看 `TaskCompletionMetric`
2. **`expected_tools` 精细化**：在 Golden 中提供 `expected_tools`（含参数），让 `ToolCorrectnessMetric` 有对比基准
3. **多次运行取均值**：LLM Judge 有随机性，`deepeval test run --n-runs 3` 降低噪音
4. **GEval 替代主观口感测试**：避免 "感觉不错就上线"，用 GEval 定义具体评分标准
5. **Dataset 与生产对齐**：持续从生产日志中提取边缘用例补充 Golden，防止 Dataset 与实际分布偏移
6. **组件级指标精准定位**：Action Layer 指标必须挂在 LLM Span 上，而非 Agent 顶层 Span

### ⚠️ 常见陷阱

| 陷阱 | 后果 | 解决方案 |
|---|---|---|
| Agent 未添加 `@observe` 直接运行指标 | Agent 指标报错（依赖 Trace） | 所有 Agent 组件必须先 Tracing |
| 只用 `TaskCompletionMetric` | 无法定位失败根因 | 配合三层指标同步运行 |
| `threshold` 设置过高（如 0.95） | 评估永远失败，CI 无法通过 | 先设 0.5 跑基准，逐步提高 |
| Golden 中 `expected_tools` 缺失 | `ToolCorrectnessMetric` 无法有效对比 | 每个 Golden 补充 `expected_tools` |
| 用被测模型同款作 Judge | Self-serving Bias，评分虚高 | 用不同厂商或更强模型做 Judge |
| Dataset 全是正常用例 | 无法发现边缘失败 | 10–20% 加入故意难/异常用例 |

---

## 14. 与 Langfuse 方案对比

| 维度 | DeepEval | Langfuse |
|---|---|---|
| **定位** | 评估框架（测试套件） | LLMOps 平台（可观测性 + 评估） |
| **Tracing** | `@observe` 装饰器，轻量 | OTLP 标准协议，兼容全生态 |
| **评估触发** | 代码驱动（`evals_iterator` / pytest） | UI 配置规则 + 代码双模式 |
| **离线 Eval** | Dataset + `run_experiment` / pytest | Dataset + Experiment（UI/SDK） |
| **生产监控** | Confident AI 平台（商业） | Langfuse 自托管或云（开源可用） |
| **Agent 专项指标** | 6 个开箱即用（推理/行动/执行层） | 需自定义 LLM-as-Judge Prompt |
| **CI/CD 集成** | 原生 pytest 风格，零学习成本 | 需自行编写 check_regression 脚本 |
| **适合团队** | 开发者优先，习惯测试驱动开发 | 全栈 LLMOps，需要 Dashboard 协作 |

**最佳实践**：两者可以**组合使用** —— DeepEval 负责离线 Eval 和 CI/CD，Langfuse 负责生产 Tracing 和 Online Eval，互不冲突。

---

## 15. 参考资料

- [DeepEval AI Agent Evaluation Guide](https://deepeval.com/guides/guides-ai-agent-evaluation)
- [DeepEval AI Agent Evaluation Metrics](https://deepeval.com/guides/guides-ai-agent-evaluation-metrics)
- [DeepEval Metrics Introduction](https://deepeval.com/docs/metrics-introduction)
- [DeepEval LLM Tracing](https://deepeval.com/docs/evaluation-llm-tracing)
- [DeepEval GitHub](https://github.com/confident-ai/deepeval)
- [Regression Testing in CI/CD](https://deepeval.com/guides/guides-regression-testing-in-ci-cd)
