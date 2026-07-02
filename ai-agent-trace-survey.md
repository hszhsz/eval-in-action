# AI Agent Trace 实现综合调研报告

> **系列**：eval-in-action — AI Agent 可观测性与评估工程实践
> **定位**：不限定单一项目、不限定参考代码的 Trace 实现机理综合调研
> **成文时间**：2026-07

---

## 摘要

如果说 [评估（Eval）](https://github.com/hszhsz/eval-in-action/blob/main/ai-agent-eval-survey.md) 回答的是"Agent 好不好"，那么**追踪（Trace）**回答的是"Agent 到底做了什么、每一步花了多少、错在了哪里"。Trace 是可观测性的地基，也是一切评估、调试、成本核算的原始数据来源——没有高质量的 trace，评估就成了对黑盒的猜测。

本报告不局限于任何单一平台（Langfuse / Phoenix / LangSmith / MLflow），而是从**实现机理**层面综合梳理 AI Agent tracing 的完整技术栈：底层的 OpenTelemetry 数据模型、上下文如何在进程内与跨服务间传播、GenAI 语义约定如何标准化模型/token/成本维度、埋点（instrumentation）的三种范式及其取舍、一次 Agent 执行如何被建模成嵌套 span 树、采样如何控制成本、以及数据如何摄取与存储。目标是让读者理解"当我调用一次 Agent，背后那条 trace 是怎么被生产、传输、落库的"。

---

## 目录

1. [Trace 是什么：从可观测性三支柱说起](#1-trace-是什么从可观测性三支柱说起)
2. [底座：OpenTelemetry 数据模型](#2-底座opentelemetry-数据模型)
3. [Tracer 管道：Span 从产生到导出](#3-tracer-管道span-从产生到导出)
4. [上下文传播：如何把散落的 Span 串成一棵树](#4-上下文传播如何把散落的-span-串成一棵树)
5. [GenAI 语义约定：让 LLM Trace 可互操作](#5-genai-语义约定让-llm-trace-可互操作)
6. [埋点三范式：手动 / 自动 / 回调](#6-埋点三范式手动--自动--回调)
7. [数据模型：一次 Agent 执行如何变成嵌套 Span 树](#7-数据模型一次-agent-执行如何变成嵌套-span-树)
8. [每个 Span 到底记录了什么](#8-每个-span-到底记录了什么)
9. [采样：在成本与可观测性之间取舍](#9-采样在成本与可观测性之间取舍)
10. [摄取、传输与存储](#10-摄取传输与存储)
11. [平台实现横向对比](#11-平台实现横向对比)
12. [多 Agent 与分布式追踪](#12-多-agent-与分布式追踪)
13. [挑战与最佳实践](#13-挑战与最佳实践)
14. [参考资料](#14-参考资料)

---

## 1. Trace 是什么：从可观测性三支柱说起

可观测性传统上由三支柱构成：**Metrics（指标）**、**Logs（日志）**、**Traces（追踪）**。对 LLM Agent 而言，Trace 是最核心的一支——因为 Agent 的价值恰恰在于它的**执行过程**（规划、工具调用、多步推理），而非单个瞬时指标。

一条 **Trace** 记录了一次完整请求在系统中的生命周期；它由若干 **Span** 组成一棵树，每个 Span 是一次可观测的操作单元（一次 LLM 调用、一次工具调用、一次检索）。对 Agent 来说：

- **调试与归因**：当 Agent 给出错误答案，Trace 能精确定位是"规划错了""工具选错了""参数填错了"还是"检索没召回"。
- **成本与性能核算**：每个 span 记录 token 数、耗时，聚合即得单次任务的总成本与延迟分布。
- **评估的数据源**：轨迹级、组件级评估（见评估报告）都建立在 trace 之上——先有 trace，才能对某个 span 打分。

---

## 2. 底座：OpenTelemetry 数据模型

几乎所有现代 LLM 可观测性平台（Langfuse v3、Arize Phoenix、OpenLLMetry、可选的 MLflow）都构建在 **OpenTelemetry（OTel）** 之上。OTel 是 CNCF 的可观测性标准，理解它就理解了 tracing 的通用语言。

**核心标识**：
- **TraceId**：全链路唯一，同一条 trace 内所有 span 共享（128-bit）。
- **SpanId**：单个 span 唯一（64-bit）。
- **ParentSpanId**：指向创建它的父 span；根 span 无父 ID。父子关系即由此三元组建立。

**一个 Span 携带的信息**：

| 字段 | 含义 |
|---|---|
| **Name** | 操作名（如 `chat`、`execute_tool`） |
| **SpanKind** | SERVER / CLIENT / INTERNAL / PRODUCER / CONSUMER |
| **起止时间戳** | 相减即得该操作延迟 |
| **Attributes** | 键值对，承载业务/模型语义（如 `gen_ai.request.model`） |
| **Events** | 带时间戳的结构化日志点（如流式响应的首 token） |
| **Links** | 关联其他 trace/span，用于扇入、批处理场景 |
| **Status** | Unset / Ok / Error |

这套模型的精妙之处：它**足够通用**（能描述任何分布式系统的调用），又**足够可扩展**（通过 Attributes 挂载领域语义）——LLM tracing 正是在通用 span 上叠加 `gen_ai.*` 属性实现的[[OpenTelemetry Tracing API 规范]](https://opentelemetry.io/docs/specs/otel/trace/api)。

---

## 3. Tracer 管道：Span 从产生到导出

OTel 定义了一条清晰的生产-导出流水线：

```
业务代码
  │ 通过 Tracer 创建/结束 Span
  ▼
TracerProvider ──持有──► IdGenerator / Sampler / SpanLimits
  │ 分发给
  ▼
SpanProcessor ──┬── SimpleSpanProcessor（同步，逐条）
  │             └── BatchSpanProcessor（批量缓冲，异步，生产推荐）
  ▼
SpanExporter ──► 序列化（OTLP）──► 后端 / Collector
```

- **TracerProvider**：API 入口与配置持有者，管理采样器、处理器等；配置变更对已创建的 Tracer 立即生效。
- **Tracer**：创建 span 的工厂，带 `InstrumentationScope`（标识埋点来源）。
- **SpanProcessor**：span 结束后的钩子。**BatchSpanProcessor** 缓冲后批量导出，避免每个 span 都触发网络 I/O、不阻塞业务线程，是生产环境标配[[OpenTelemetry Tracing SDK 规范]](https://opentelemetry.io/docs/specs/otel/trace/sdk)。
- **SpanExporter**：把 span 序列化并发往后端（通常经 OTLP 协议）。

> **对 LLM 应用的启示**：短生命周期的脚本（如一次性评估任务）用 BatchSpanProcessor 时，进程退出前**必须显式 `flush()`**，否则缓冲区里的 span 会丢失——这是新手最常见的"trace 不见了"的原因。

---

## 4. 上下文传播：如何把散落的 Span 串成一棵树

一条 trace 的多个 span 可能分布在不同函数、线程、甚至不同服务里。让它们"认亲"的机制就是**上下文传播（Context Propagation）**。

**进程内**：OTel 用一个不可变的 `Context` 对象承载"当前活跃 span"。各语言用惯用机制隐式传递——Python/JS 用 `contextvars`，Java/Go 用显式 Context 或 thread-local。调用 `start_as_current_span` 会把新 span 设为当前上下文，其子 span 自动读取父 `SpanContext` 建立父子关系[[OpenTelemetry Context Propagation]](https://opentelemetry.io/docs/concepts/context-propagation)。这就是为什么 `@observe` 装饰器嵌套调用时，trace 树能自动成型——每层函数进入时读取当前上下文作为父级。

**跨进程/服务**：通过 **Propagator** 的 `inject`/`extract` 把 context 写入/读出"载体（carrier，通常是 HTTP headers）"。业界标准是 **W3C Trace Context**，两个 header：

- `traceparent`：格式 `version-traceId-parentId-traceFlags`，承载核心标识与采样决定。
- `tracestate`：承载厂商特定的键值对。

发送方注入、接收方提取，即可让 span 跨网络边界串成同一条 trace[[W3C Trace Context 详解]](https://www.dash0.com/knowledge/w3c-trace-context-traceparent-tracestate)。这直接映射到**分布式多 Agent** 场景：在 agent-to-agent、agent-to-tool-service 的 RPC 调用中传播 `traceparent`，各服务的子树就能拼成端到端 trace。

> **注意**：部分内存态实现（如 LangSmith 的 RunTree）在分布式场景下仅传 ID 不够，需要传递整个 tracing context，因为父子关系存于内存对象而非纯标识[[LangSmith Tracing Deep Dive]](https://medium.com/@aviadr1/langsmith-tracing-deep-dive-beyond-the-docs-75016c91f747)。

---

## 5. GenAI 语义约定：让 LLM Trace 可互操作

通用 span 能记录"发生了一次调用"，但要记录"这是对 GPT-4 的一次 chat，用了 1200 个输入 token"，就需要**标准化的属性命名**。这就是 **OpenTelemetry GenAI 语义约定**的使命——由 OTel GenAI SIG（2024 年 4 月成立）定义 `gen_ai.*` 命名空间，使不同厂商 SDK 产出一致的遥测[[OpenTelemetry GenAI Observability]](https://opentelemetry.io/blog/2026/genai-observability)。

**关键属性**：

| 属性 | 含义 |
|---|---|
| `gen_ai.provider.name` | 提供商（openai / anthropic …） |
| `gen_ai.operation.name` | 操作类型（chat / text_completion / embeddings / execute_tool / invoke_agent） |
| `gen_ai.request.model` / `gen_ai.response.model` | 请求/响应模型 |
| `gen_ai.request.temperature` / `max_tokens` | 生成参数 |
| `gen_ai.usage.input_tokens` / `output_tokens` | token 用量 |
| `gen_ai.response.finish_reasons` | 结束原因 |
| `gen_ai.conversation.id` | 会话聚合 ID |
| `gen_ai.agent.name` / `id` / `description` | Agent 专属 |
| `gen_ai.tool.name` / `gen_ai.tool.call.id` | 工具调用专属 |

**重要提醒——约定仍在演进**：
1. GenAI 属性已从主 semantic-conventions 仓库**迁出到独立仓库**，主仓库中相关条目标记为 Deprecated/Moved。
2. Prompt/Completion 内容捕获方式在变——早期"每条消息一个 Event"正被三个聚合属性（`gen_ai.system_instructions`、`gen_ai.input.messages`、`gen_ai.output.messages`）取代[[OpenTelemetry GenAI 语义约定分析]](https://greptime.com/blogs/2026-05-09-opentelemetry-genai-semantic-conventions)。
3. **默认不捕获 prompt/工具参数内容**（可能含敏感数据），需显式 opt-in。
4. 多 Agent 团队工作流的属性（`gen_ai.team.size` 等）仍处 RFC 阶段，尚未定稿。

因此，业界存在**两套并存的约定**：Arize 的 **OpenInference**（更成熟、agent 特化）与官方 **OTel GenAI**（演进中），预期未来收敛[[OpenInference vs OpenTelemetry GenAI]](https://www.arthur.ai/column/openinference-vs-opentelemetry-genai-conventions-agent-tracing)。引用具体属性名时务必标注来源与版本。

---

## 6. 埋点三范式：手动 / 自动 / 回调

"埋点（Instrumentation）"是指把 span 生产逻辑注入到应用中。有三种范式，各有取舍：

### 6.1 手动埋点（Manual）
用装饰器（如 `@observe`、`@traceable`、`@mlflow.trace`）、上下文管理器（`with tracer.start_as_current_span(...)`）或显式 start/end 手写 span。
- **优点**：精确控制、可挂载业务语义。
- **缺点**：侵入代码、易漏、维护成本高。

### 6.2 自动埋点 / Monkey-patching（Auto）
库在导入时替换 OpenAI/Anthropic/LangChain 的 SDK 方法，自动产出带 `gen_ai.*` 的 span，零代码改动。OpenLLMetry（Traceloop）一行 `Traceloop.init()` 即开始追踪，且输出标准 OTel 数据、可接任意后端[[OpenLLMetry GitHub]](https://github.com/traceloop/openllmetry)。
- **优点**：开箱即用、覆盖广。
- **缺点**：与 SDK 版本耦合、可能与其他改写全局 TracerProvider 的库冲突。

### 6.3 回调式（Callback）
LangChain / LlamaIndex 提供 callback handler，框架在各执行阶段回调，由 handler 转成 span。LangChain 的 `BaseCallbackHandler` 定义了完整生命周期钩子：`on_llm_start` / `on_llm_end` / `on_chain_start` / `on_tool_start` / `on_retriever_start` / `on_agent_action` 等[[LangChain BaseCallbackHandler]](https://reference.langchain.com/python/langchain-core/callbacks/base/BaseCallbackHandler)。可观测性平台实现自定义 handler，在 `_start` 开 span、`_end` 关 span。
- **优点**：官方支持、贴合框架执行模型。
- **缺点**：仅限支持回调的框架。

> **实践组合**：多数平台混合使用——对框架内的组件用回调/自动埋点省力，对自定义业务逻辑用手动埋点补充语义。

---

## 7. 数据模型：一次 Agent 执行如何变成嵌套 Span 树

Agent 的执行天然映射为**嵌套 span 树**。以一个"查天气并推荐穿衣"的 Agent 为例：

```
Trace: invoke_agent "weather-assistant"          [根 span, 3.2s, $0.018]
├── Span(chat): planner-llm                       [0.8s, 420 tok]  ← 规划
├── Span(execute_tool): get_location              [0.1s]           ← 工具调用
│     args={} → result={"city":"Beijing"}
├── Span(execute_tool): get_weather               [0.3s]           ← 工具调用
│     args={"city":"Beijing"} → result={"temp":5,"cond":"晴"}
├── Span(retrieval): dressing-knowledge-base      [0.2s, 3 docs]   ← 检索
│     docs=[{id, content, score:0.87}, ...]
└── Span(chat): response-llm                      [1.8s, 890 tok]  ← 最终回答
      input_messages=[...] → output="今天北京 5℃ 晴，建议..."
```

**关键映射规则**：
- 整个 Agent 循环是**根 span**（`invoke_agent`）。
- 每次 LLM 调用、工具调用、检索、子 Agent 都是**子 span**。
- 延迟由 span 起止时间戳相减得出；根 span 的时长即端到端延迟。
- 成本、token 逐 span 记录，聚合到根 span 即单次任务总成本。

**流式响应的特殊处理**：span 通常在流开始时创建，首 token / 中间 chunk 记为 events，流结束（收到 finish_reason）后再补全输出内容与 token 统计并 end span——这样才能同时捕获 **TTFT（首 token 延迟）** 与总时长[[Trace AI Agent Execution Flows]](https://oneuptime.com/blog/post/2026-02-06-trace-ai-agent-execution-flows-opentelemetry/view)。

不同平台对这套树的命名略有差异，但**核心 span 类型高度趋同**：LLM / CHAIN / TOOL / RETRIEVER / AGENT / EMBEDDING / RERANKER。

---

## 8. 每个 Span 到底记录了什么

综合 OpenInference、OTel GenAI、各平台约定，一个 LLM/Agent span 典型捕获以下字段[[OpenInference 语义约定]](https://arize-ai.github.io/openinference/spec/semantic_conventions.html)：

| 类别 | 字段示例 |
|---|---|
| **输入/输出消息** | `input.value` / `output.value`；`llm.input_messages.N.message.role/content`（展平索引） |
| **模型 + 参数** | `llm.model_name`、`llm.provider`、`llm.invocation_parameters`（temperature/top_p）、`finish_reason` |
| **token 用量** | prompt / completion / total；细分 `prompt_details.cache_read/cache_write`、`completion_details.reasoning` |
| **成本** | `llm.cost.prompt/completion/total` |
| **延迟** | span 起止时间戳 |
| **工具** | `llm.tools.N.tool.json_schema`、`function_call_name`、`function_call_arguments_json`、`tool_call_id`；TOOL span 的 input/output |
| **错误** | `exception.type/message/stacktrace` |
| **元数据/标签** | session_id、user_id、tags（trace 级属性传播到全部 span） |
| **检索文档** | `document.content/id/score/metadata`（RETRIEVER span） |

> **成本是"算"出来的，不是"记"出来的**：OTel GenAI 约定本身**不含成本字段**。成本由观测平台按 `token 数 × 模型价目表` **后处理推算**——这意味着换模型或调价后，历史 trace 的成本可能需要重算。这是一个容易被忽视的实现细节。

---

## 9. 采样：在成本与可观测性之间取舍

全量记录每一条 trace 在高流量场景下成本高昂（存储 + 处理）。采样是控制成本的主要杠杆，分两类：

**Head-based（头部采样）**：在 trace 起点由根服务决策保留/丢弃，并通过 `traceflags` 把决定沿 context 传播给下游。
- **优点**：开销低、简单。
- **缺点**：决策时**尚不知道** trace 是否会出错或很昂贵，可能丢掉关键失败 trace。

**Tail-based（尾部采样）**：在 Collector 中等 trace 所有 span 到齐后再决策，支持按 error status、latency、属性匹配、概率等策略组合[[Head-based vs Tail-based Sampling]](https://oneuptime.com/blog/post/2026-02-06-head-based-vs-tail-based-sampling-opentelemetry/view)。
- **对 LLM 应用尤为重要**：能确保保留**报错的、超长/超贵（高 token/高成本）** 的 trace，同时对海量正常 trace 只留小比例。
- **代价**：Collector 需缓存 span、有状态、迟到 span 可能被误判丢弃。

> **LLM 场景的采样哲学**：与传统微服务不同，LLM trace 单条价值高、体量大。推荐**尾部采样 + 强制保留异常/高成本 trace**，正常流量按 5–10% 采样。评估用的生产回捞往往正需要这些"异常样本"。

---

## 10. 摄取、传输与存储

**OTLP 协议**：遥测数据默认经 **OTLP（OpenTelemetry Protocol）** 传输，采用 Protobuf 编码，支持 **gRPC（4317）** 与 **HTTP（4318）** 两种传输方式[[OTLP 规范]](https://opentelemetry.io/docs/specs/otlp)。

**Collector 中间层**：OTel Collector 作为代理（receiver → processor → exporter），负责接收、加工（批处理、尾采样、**脱敏**）并转发到后端。脱敏尤其重要——LLM prompt 常含敏感数据。

**存储后端**：LLM 可观测平台普遍采用**列式 OLAP 存储**处理高基数、大体量 span，典型是 **ClickHouse**（Langfuse、SigNoz、OpenObserve 等均采用此架构）。Span 表常用 LowCardinality + ZSTD 压缩、Attributes 存为 Map[[ClickHouse + OpenTelemetry 集成]](https://clickhouse.com/docs/observability/integrating-opentelemetry)。

**开销控制的三个杠杆**：批量+异步导出、采样、内容 opt-in（是否记录完整 prompt/completion）。

---

## 11. 平台实现横向对比

| 平台 | 数据模型 | Span 类型 | OTel 关系 | 埋点方式 |
|---|---|---|---|---|
| **Langfuse v3** | Session > Trace > Observation | SPAN / GENERATION / EVENT | SDK 重建于 OTel；可作 OTLP 后端 | `@observe`、上下文管理器、集成包裹 |
| **Arize Phoenix** | OpenInference span | LLM/CHAIN/TOOL/RETRIEVER/AGENT/EMBEDDING/RERANKER/… | 定义 OpenInference（基于 OTel） | 30+ 框架自动插桩 |
| **OpenLLMetry** | 标准 OTLP | 遵循 OTel/OpenInference | 纯 OTel 扩展 | 一行 init 自动埋点 |
| **LangSmith** | RunTree（内存态嵌套 run） | llm/chain/tool/retriever/embedding/prompt/parser | 独立实现，非 OTel 原生 | `@traceable`、LangChain 回调 |
| **MLflow Tracing** | Trace > Span | Agent/Retriever/Tool/Chat/Chain/LLM/… | 可选导出为 OTLP + `gen_ai.*` | `@mlflow.trace`、`autolog()` |

**横向观察**：
1. **OTel 已成事实标准底座**——除 LangSmith 走独立内存态 RunTree 外，主流平台都产出或兼容 OTLP，后端可互换。
2. **两套语义约定并存**——OpenInference（更成熟）vs OTel GenAI（官方、演进中），预期收敛。
3. **span kind 高度趋同**——都收敛到 LLM/CHAIN/TOOL/RETRIEVER/AGENT/EMBEDDING/RERANKER 这组核心类型。

---

## 12. 多 Agent 与分布式追踪

多 Agent 系统（如 Planner Agent 调度多个 Worker Agent，或 Agent 通过 MCP 调用远程工具服务）的追踪依赖两个机制：

- **上下文传播串树**：同一 `trace_id` 下，span 跨函数/线程/服务/多个 agent 连成一棵树。进程内靠 OTel context 自动传递，跨服务靠 W3C `traceparent` 手动注入/提取。
- **会话分组**：多轮对话的多条 trace 需聚合成"会话"。Langfuse 用 `session_id` 聚成 Session（对应聊天 thread）；OTel GenAI 用 `gen_ai.conversation.id`。

多 Agent 团队工作流的专属属性（如 `gen_ai.team.size`、agent 间消息传递）仍处 RFC 提案阶段，尚未标准化[[Agent 可观测性 RFC]](https://github.com/traceloop/openllmetry/issues/3460)。这是当前 tracing 标准化的最前沿、也最不稳定的区域。

---

## 13. 挑战与最佳实践

### 核心挑战

| 挑战 | 本质 | 应对 |
|---|---|---|
| **标准演进中** | GenAI 语义约定（尤其 agent/multi-agent、消息格式）频繁变动 | 依赖平台的兼容映射层；引用属性标注版本 |
| **敏感数据** | prompt/工具参数常含 PII | 默认 opt-out 内容捕获；Collector 层脱敏 |
| **成本推算易错** | 成本非规范字段，由平台按价目表算 | 维护准确价目表；换模型/调价后重算 |
| **流式响应** | 需正确捕获 TTFT 与总时长 | 流开始建 span、chunk 记 event、结束才 end |
| **采样丢关键 trace** | 头部采样可能丢掉失败/高成本 trace | 尾部采样 + 强制保留异常 |
| **数据体量** | LLM span 大（含长 prompt）、高基数 | 列式存储（ClickHouse）+ 压缩 + 采样 |
| **埋点冲突** | 多个库改写全局 TracerProvider | 统一 OTel 管道，避免重复自动埋点 |

### 最佳实践

1. **拥抱 OTel 标准**：优先选择产出/兼容 OTLP 的方案，避免供应商锁定。
2. **分层埋点**：框架组件用自动/回调埋点，自定义逻辑用手动埋点补语义。
3. **生产用 BatchSpanProcessor**：短任务记得 `flush()`。
4. **尾部采样保异常**：正常流量 5–10%，错误/高成本 trace 全留。
5. **内容捕获显式 opt-in + 脱敏**：平衡可观测性与合规。
6. **trace 级属性传播**：session_id / user_id / tags 挂在 trace 上，自动传播到全部 span，便于后续聚合分析与评估。
7. **为评估预留字段**：确保 trace 记录了轨迹级/组件级评估所需的工具名、参数、检索分数——trace 质量决定评估上限。

---

## 14. 参考资料

**OpenTelemetry 基础**
- OpenTelemetry Tracing API 规范 — https://opentelemetry.io/docs/specs/otel/trace/api
- OpenTelemetry Tracing SDK 规范 — https://opentelemetry.io/docs/specs/otel/trace/sdk
- OpenTelemetry Context Propagation — https://opentelemetry.io/docs/concepts/context-propagation
- W3C Trace Context (traceparent/tracestate) — https://www.dash0.com/knowledge/w3c-trace-context-traceparent-tracestate
- OpenTelemetry Sampling 概念 — https://opentelemetry.io/docs/concepts/sampling
- OTLP 规范 — https://opentelemetry.io/docs/specs/otlp

**GenAI 语义约定**
- OpenTelemetry GenAI Observability (Inside the LLM Call) — https://opentelemetry.io/blog/2026/genai-observability
- GenAI 属性注册表 — https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai
- OpenTelemetry GenAI 语义约定演进分析 — https://greptime.com/blogs/2026-05-09-opentelemetry-genai-semantic-conventions
- OpenInference 语义约定规范 — https://arize-ai.github.io/openinference/spec/semantic_conventions.html
- OpenInference vs OpenTelemetry GenAI Conventions — https://www.arthur.ai/column/openinference-vs-opentelemetry-genai-conventions-agent-tracing

**采样与存储**
- Head-based vs Tail-based Sampling — https://oneuptime.com/blog/post/2026-02-06-head-based-vs-tail-based-sampling-opentelemetry/view
- Tail Sampling (New Relic) — https://newrelic.com/blog/observability/open-telemetry-tail-sampling
- ClickHouse + OpenTelemetry 集成 — https://clickhouse.com/docs/observability/integrating-opentelemetry
- ClickHouse Exporter 配置 — https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/exporter/clickhouseexporter/README.md

**平台实现**
- Langfuse 数据模型 — https://langfuse.com/docs/observability/data-model
- Langfuse OTEL-based Python SDK — https://langfuse.com/changelog/2025-05-23-otel-based-python-sdk
- Langfuse OpenTelemetry 集成 — https://langfuse.com/integrations/native/opentelemetry
- Arize Phoenix 语义约定 — https://arize.com/docs/phoenix/tracing/concepts-tracing/otel-openinference/semantic-conventions
- OpenLLMetry (Traceloop) GitHub — https://github.com/traceloop/openllmetry
- LangSmith Tracing Deep Dive — https://medium.com/@aviadr1/langsmith-tracing-deep-dive-beyond-the-docs-75016c91f747
- LangChain BaseCallbackHandler — https://reference.langchain.com/python/langchain-core/callbacks/base/BaseCallbackHandler
- MLflow 手动追踪 — https://mlflow.org/docs/latest/genai/tracing/app-instrumentation/manual-tracing
- LlamaIndex Instrumentation — https://developers.llamaindex.ai/python/framework/module_guides/observability/instrumentation

**Agent 追踪实践**
- Trace AI Agent Execution Flows with OpenTelemetry — https://oneuptime.com/blog/post/2026-02-06-trace-ai-agent-execution-flows-opentelemetry/view
- Agent 可观测性 RFC (#3460) — https://github.com/traceloop/openllmetry/issues/3460

**本系列相关文档**
- Eval in Langfuse — https://github.com/hszhsz/eval-in-action/blob/main/eval-in-langfuse.md
- Eval in DeepEval — https://github.com/hszhsz/eval-in-action/blob/main/eval-in-DeepEval.md
- AI Agent Eval 综合调研报告 — https://github.com/hszhsz/eval-in-action/blob/main/ai-agent-eval-survey.md
