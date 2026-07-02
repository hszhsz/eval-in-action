# AI Agent 可观测性实现综合调研报告

> **系列**：eval-in-action — AI Agent 可观测性与评估工程实践
> **定位**：不限定单一项目、不限定参考代码的可观测性实现综合调研
> **成文时间**：2026-07

---

## 摘要

可观测性（Observability）是 AI Agent 从"能跑的 Demo"走向"可信赖的生产系统"的必经之路。但对 LLM Agent 而言，可观测性远不止"记录一条 trace"——一个返回 HTTP 200、耗时 800ms 的请求，仍可能给出一个完全错误、甚至有害的答案：没有异常、没有堆栈、故障还不可复现。传统 APM（应用性能监控）在这种非确定性面前几乎失灵。

本报告不局限于任何单一平台，从**实现与概念**层面综合梳理 AI Agent 可观测性的完整版图。它与本系列的 [Trace 实现报告](https://github.com/hszhsz/eval-in-action/blob/main/ai-agent-trace-survey.md) 形成互补——后者深入 tracing 的底层管道（span、上下文传播、语义约定、采样、存储），本文则聚焦**更宏观的可观测性体系**：为什么传统三支柱需要扩展出"质量"这第四支柱、如何做指标与成本可观测、如何在生产流量上做在线质量监控、护栏如何充当观测+控制层、用户反馈与漂移检测如何形成时间维度的闭环、Prompt 版本如何纳入可观测性，以及整个平台生态的分类。

---

## 目录

1. [为什么 LLM 可观测性是一门独立学科](#1-为什么-llm-可观测性是一门独立学科)
2. [从三支柱到第四支柱：Metrics / Logs / Traces / Quality](#2-从三支柱到第四支柱metrics--logs--traces--quality)
3. [指标可观测性：该追踪哪些量化信号](#3-指标可观测性该追踪哪些量化信号)
4. [成本与 Token 可观测性](#4-成本与-token-可观测性)
5. [在线质量监控：把评估器搬到生产流量上](#5-在线质量监控把评估器搬到生产流量上)
6. [护栏与安全监控：观测 + 控制层](#6-护栏与安全监控观测--控制层)
7. [用户反馈与人类信号](#7-用户反馈与人类信号)
8. [漂移与回归检测：时间维度的可观测性](#8-漂移与回归检测时间维度的可观测性)
9. [Prompt 管理与版本控制](#9-prompt-管理与版本控制)
10. [告警与事件响应：面向 AI 的 SLO](#10-告警与事件响应面向-ai-的-slo)
11. [可观测性平台生态全景](#11-可观测性平台生态全景)
12. [贯穿全局的元数据主线](#12-贯穿全局的元数据主线)
13. [成熟度评估与最佳实践](#13-成熟度评估与最佳实践)
14. [参考资料](#14-参考资料)

---

## 1. 为什么 LLM 可观测性是一门独立学科

传统可观测性回答系统事件的 **what / where / why**——服务挂没挂、慢在哪、为什么报错。它建立在一个隐含假设上：**正确性可以由状态码和异常判定**。

这个假设在 LLM Agent 上崩塌了。Agent 的失败往往是"静默的"：

- 请求成功返回，但答案是幻觉。
- 工具选对了，但参数填错，导致下游结果错误。
- 延迟正常、成本正常，但回答与用户意图完全无关。

这些失败**没有异常、不抛堆栈、且因非确定性而难以复现**。因此业界逐渐形成共识：LLM 可观测性是一门区别于传统 APM 的独立学科，它必须额外回答第四个问题——**"输出到底好不好"（Quality）**[[The three pillars of AI observability]](https://www.braintrust.dev/blog/three-pillars-ai-observability)。更进一步，"Agentic observability"还要追踪 Agent 的**决策逻辑**（为何选择某个工具、走某条推理路径），而不仅是模型生成了什么内容。

---

## 2. 从三支柱到第四支柱：Metrics / Logs / Traces / Quality

经典可观测性由三支柱构成[[Three Pillars of Observability]](https://www.ibm.com/think/insights/observability-pillars)：

| 支柱 | 传统含义 | 在 LLM/Agent 上的适配 |
|---|---|---|
| **Metrics（指标）** | 聚合的数值时序（CPU、QPS） | 延迟（TTFT/TPS）、token 用量、成本、错误率、工具成功率 |
| **Logs（日志）** | 离散事件记录 | prompt/completion 原文、工具调用参数与结果、异常 |
| **Traces（追踪）** | 请求的调用链 | Agent 执行的嵌套 span 树（见 Trace 报告） |

对 LLM Agent，这三支柱**必要但不充分**。业界普遍主张引入**第四支柱：质量（Quality）**——通过在采样流量上运行的**在线评估**来度量输出的正确性、忠实度、相关性、安全性。有观点甚至将 AI 语境下的支柱重构为 **Traces / Evals / Annotation**。

> **核心洞见**：传统三支柱告诉你"系统在正常运转"，第四支柱才告诉你"系统在做正确的事"。可观测性从"监控系统健康"扩展为"监控输出质量"，这是 LLM 可观测性的范式转变。

---

## 3. 指标可观测性：该追踪哪些量化信号

LLM/Agent 应用需要追踪一组**领域特有的指标**，远超传统的 QPS 与延迟：

**延迟类**（LLM 独有的细粒度延迟指标）：
- **TTFT（Time To First Token）**：首 token 延迟，直接影响用户"感知快慢"。
- **TPOT / ITL（每 token 延迟）**：token 间生成间隔。
- **TPS（Tokens Per Second）**：吞吐量。
- **E2EL（端到端延迟）**：整个请求耗时。

**用量与成本类**：token 用量（输入/输出/缓存）、按请求/用户/功能的成本。

**质量与可靠性类**：错误率、**工具调用成功率**、缓存命中率、重试率。

这些指标通常用 **Prometheus Histogram** 表达，取 mean/median/P99，配 Grafana 看板。vLLM 等推理引擎原生通过 `/metrics` 端点暴露 server-level 与 request-level 指标[[vLLM Metrics]](https://docs.vllm.ai/en/stable/design/metrics)。SRE 通常将 request-level 指标作为 SLO 的跟踪对象[[Understand LLM latency and throughput metrics]](https://docs.anyscale.com/llm/serving/benchmarking/metrics)。

> **实践要点**：TTFT 和 P99 尾延迟对流式 Agent 的用户体验尤为关键——平均延迟正常不代表没有"卡顿的长尾请求"。

---

## 4. 成本与 Token 可观测性

成本可观测性是 LLM 系统区别于传统应用的显著维度——因为**每次调用都在花钱**，且成本高度不均（一次复杂 Agent run 可能是简单问答的几十倍）。

**核心实现思路**：让每次模型调用携带足够的元数据（user / customer / feature / workflow），将 `gen_ai.usage.input_tokens` / `output_tokens` 结合模型价目表换算为美元，做**按模型/用户/会话/功能的成本归因**[[LLM Cost Monitoring with OpenTelemetry]](https://uptrace.dev/blog/llm-cost-monitoring)。

**为什么供应商账单不够用**：供应商账单只能告诉你"这个月涨了 30%"，却无法解释"是哪个用户、哪个功能、哪些重试、哪个 Agent run 导致的"。可观测性要补上这个归因能力[[How to track LLM costs 2026]](https://www.braintrust.dev/articles/how-to-track-llm-costs-2026)。

**两类工具**：
- **网关层（Gateway）**：拦截每次请求，实时限额、硬性上限（hard cap）、熔断（kill switch）。
- **观测层（Observability）**：通过 span 事后追溯与归因。

部分平台（如 Datadog）用公开定价自动估算每请求成本，并支持把 span tag 提升为成本/token 的自定义分析维度[[Datadog LLM Observability Cost]](https://docs.datadoghq.com/llm_observability/monitoring/cost)。

> **实现细节**：成本是"算"出来的而非"记"出来的——换模型或供应商调价后，历史成本口径可能需要重算。维护一份准确、及时更新的价目表是成本可观测性的隐性前提。

---

## 5. 在线质量监控：把评估器搬到生产流量上

这是第四支柱"质量"的落地形态，也是 LLM 可观测性最具特色的部分。

**离线 vs 在线评估的分工**：
- **离线评估**：在发布前 / CI 中对固定数据集打分，防止回归上线（详见 [Eval 综合报告](https://github.com/hszhsz/eval-in-action/blob/main/ai-agent-eval-survey.md)）。
- **在线评估**：上线后对**实时生产流量采样**打分，捕捉离线测试集覆盖不到的漂移、滥用、成本尖峰[[Online vs Offline LLM evaluation]](https://deepchecks.com/question/online-vs-offline-llm-evaluation)。

**常用手段**：LLM-as-judge、启发式/规则评估器，实时监控幻觉率、毒性、相关性、忠实度（faithfulness）、groundedness 的趋势变化。

**演进方向——观测级评估**：主流平台正从"整条链路评估"走向 **observation-level（单步级）评估**——例如只对 RAG 链的最终生成评 helpfulness、只对检索步评 relevance，比整链评估更快、更精准[[Observation-level evals]](https://langfuse.com/changelog/2026-02-13-observation-level-evals)。

> **成本约束**：在线评估本身要消耗 LLM 裁判的算力。因此实践中对生产流量**采样 5–10%** 评估，并优先保留异常/高成本 trace——这与 trace 采样策略天然协同。

---

## 6. 护栏与安全监控：观测 + 控制层

护栏（Guardrails）是可观测性与**控制**的结合体。区别于评估（回答"模型够不够好上线"）和观测（回答"生产里发生了什么"），护栏回答的是——**"这一条具体的请求/响应，现在发送出去安全吗"**。它内联运行在请求路径上，执行"检查 + 动作"（allow / block / rewrite / alert / route）。

**两类护栏**：
- **输入护栏**：PII 检测、prompt injection / jailbreak 检测、越界话题拦截。
- **输出护栏**：毒性检测、内容审核、schema 校验。

**代表方案**：Guardrails AI（可组合的 validator 库）、NVIDIA NeMo Guardrails（可编程对话 rails）、LLM Guard、Meta Llama Guard 3 / Prompt Guard、Azure Prompt Shields、OpenAI moderation、Lakera Guard[[AI Guardrails Production Safety Guide 2026]](https://myengineeringpath.dev/genai-engineer/ai-guardrails)。语义类攻击（礼貌措辞的越狱、流畅文本中的策略违规）难以被正则/关键词拦截，需要运行时分类器（可做到 <90ms）。

> **⚠️ 重要警示**：护栏并非万无一失。研究显示，通过字符注入与对抗性 ML 规避，可对 Azure Prompt Shield、Meta Prompt Guard 等主流护栏达到高达 100% 的逃逸率[[Bypassing Prompt Injection and Jailbreak Detection]](https://arxiv.org/html/2504.11168v1)。且护栏"失败即静默通过（fail open）"是重大风险——护栏本身也需要被观测（护栏触发率、拦截率、逃逸告警）。

---

## 7. 用户反馈与人类信号

用户反馈是把"真实世界的质量判断"接入可观测性的通道，分两类：

- **显式反馈**：thumbs up/down、评分、自由文本。用户知道自己在反馈，信号结构化、清晰。
- **隐式反馈**：点击率、会话时长、任务完成率、重试/追问/纠正等行为信号。样本量大、偏差小，但是**含噪的学习信号**[[Explicit and Implicit LLM User Feedback]](https://www.nebuly.com/blog/explicit-implicit-llm-user-feedback-quick-guide)。

**最佳实践**：把 thumbs up/down 当作一个完整的"反馈 + 观测系统"，而非两个孤立的图标——每条反馈关联唯一的 trace/session ID，以便回溯到具体的幻觉、工具失败或回归[[Tracking User Feedback]](https://www.traceloop.com/docs/openllmetry/tracing/user-feedback)。

**反馈闭环**：生产信号 → 数据集行 → 回归检查 → 下次部署的质量门禁。部分平台（如 Datadog）通过 Automations + Annotation Queues，按规则/采样自动把生产 trace 路由到数据集或人工标注队列[[Annotate traces / Annotation Queues]](https://www.datadoghq.com/blog/automations-annotation-queues)。这正是可观测性与评估形成闭环的关键——**今天生产里的失败 case，是明天离线测试集的边缘样本**。

---

## 8. 漂移与回归检测：时间维度的可观测性

单点的 trace 和指标是"快照"，而 Agent 的质量会**随时间退化**。漂移检测为可观测性引入时间维度：

- **数据漂移（Data Drift）**：输入分布 P(X) 变化。LLM 场景下最有效的度量是**输入 prompt 的 embedding 分布偏移**（PSI、KS 检验、embedding 余弦距离）[[5 methods to detect drift in ML embeddings]](https://www.evidentlyai.com/blog/embedding-drift-detection)。
- **概念/模型漂移（Concept/Model Drift）**：P(Y|X) 或模型质量变化——供应商静默更新模型、检索索引变化、工具行为变化都会触发。
- **Prompt 漂移（Prompt Drift）**：即使 prompt 一字未改，底层变化也会使同一 prompt 的输出随时间改变[[Prompt Drift]](https://agenta.ai/blog/prompt-drift)。

**2026 的实践共识**：对每条生产 trace 同时记录 embedding + eval score，在"**输入漂移 + 可测的 eval 分数下降**"这个联合条件上告警，可把故障平均检测时间（MTTD）从"数天（等客户投诉）"降到"数分钟"[[Model Drift vs Data Drift 2026]](https://futureagi.com/blog/model-vs-data-drift-how-to-identify-and-handle-it)。

> **⚠️ 演进中的领域**：LLM 输出漂移无法被传统统计监控完全捕获，语义级漂移检测仍在快速发展。这是可观测性中较不成熟、需谨慎对待的一环。

---

## 9. Prompt 管理与版本控制

Prompt 是 LLM 应用的核心"源代码"，把它纳入可观测性是回归定位的关键。

**核心实现**：集中式 **prompt 注册表 + 不可变版本 + 环境标签**，使每个输出都能回溯到产出它的确切 prompt 版本（连同模型、成本）[[Version Control - Langfuse]](https://langfuse.com/docs/prompt-management/features/prompt-version-control)。这带来两个能力：
1. **脱离代码部署改 prompt**：修改 prompt 无需重新发版。
2. **A/B 测试**：给不同版本打标签（如 `prod-a` / `prod-b`），随机选取并把 prompt 关联到 generation 做分析[[A/B Testing of LLM Prompts]](https://langfuse.com/docs/prompt-management/features/a-b-testing)。

**与 trace 的联动是精髓**——当质量回归时，能明确回答"是**哪个 prompt 版本**在**哪个模型**上产生了坏输出"。代表工具：Langfuse、LangSmith、PromptLayer、Humanloop、Maxim、Agenta、MLflow Prompt Registry、Amazon Bedrock Prompt Management。

---

## 10. 告警与事件响应：面向 AI 的 SLO

**推荐范式——基于 SLO 的告警**：不对基础设施症状（CPU 高）告警，而对**面向用户的影响**（可用性、响应性、错误率）设 SLI/SLO 与错误预算（error budget）[[SLO-Based Alerting]](https://openobserve.ai/blog/slo-based-alerting)。

**降低告警疲劳——burn rate + 多窗口告警**：长窗口确认"持续影响"、短窗口确认"仍在发生"（如"1 小时 burn rate > 14.4x"才触发），避免瞬时抖动误报[[How to Build SLO Alerting Strategies]](https://oneuptime.com/blog/post/2026-01-30-slo-alerting-strategies/view)。

**AI 系统特有的告警对象**（超出传统 APM）：
- 质量分数骤降（第四支柱信号）
- 幻觉率 / 毒性率上升
- 成本尖峰
- 工具调用失败率
- 输入/输出漂移

> **⚠️ 早期领域**：面向 Agent 的 SLO 框架仍处早期——自主系统正在重新定义"什么算一次 incident"。合成校验 Agent 工作流、eval 驱动的质量告警（如质量下降触发 PagerDuty/Slack）属于新兴实践，尚无成熟标准。

---

## 11. 可观测性平台生态全景

可观测性平台可按"出发点"分为四类，理解分类有助于选型：

| 类别 | 代表 | 特点 | 差异化 |
|---|---|---|---|
| **OSS / tracing-first（可自托管）** | Langfuse、Arize Phoenix、Helicone、OpenLLMetry、MLflow | tracing + prompt + eval + 成本一体 | Langfuse 自托管功能对等；Phoenix OTel 可移植性最佳；Helicone drop-in 代理分钟级接入 |
| **Eval-first** | Confident AI/DeepEval、Braintrust | "eval 即 observability"，每条 trace 多指标打分 | Braintrust 的 CI/CD eval-gate 工作流最强 |
| **APM-extended（企业）** | Datadog LLM Obs、New Relic AI Monitoring、Dynatrace | 在成熟 APM 上扩展 LLM 能力 | 适合已用该 APM 的团队；被评为"dashboard add-on"，多步 Agent 因果链建模较弱 |
| **Framework-native** | LangSmith、W&B Weave | 与特定框架深度集成 | LangSmith 在 LangChain 生态路径最短 |

**选型的三个关键差异点**：
1. **是否真开源/可自托管**（数据合规、成本）。
2. **形态**：drop-in 代理 vs 全平台 vs eval-gate。
3. **能否建模多轮/多 Agent 会话的因果链**——多个来源强调：单次 call 级监控看不到多轮 Agent 的失败，需要完整 session 的因果 trace[[LLM Observability Tools Comparison]](https://lushbinary.com/blog/llm-observability-tools-comparison-langfuse-helicone-phoenix)。

> **⚠️ 提醒**：各厂商的对比榜单多为自评，存在偏向。选型应以自身场景（合规要求、框架栈、团队规模）实测为准。

---

## 12. 贯穿全局的元数据主线

前述所有能力——成本归因、质量回溯、漂移告警、反馈闭环、prompt 定位——之所以能成立，靠的是一条**贯穿始终的元数据主线**：

```
每次 LLM/Agent 调用都携带：
  身份维度：user_id / session_id / feature / workflow
  资产维度：prompt_version / model / model_params
  度量维度：input_tokens / output_tokens / cost / latency
  质量维度：eval_scores / user_feedback / guardrail_flags
        │
        ▼
  以 trace_id / session_id 为主键关联起来
        │
        ▼
  ┌─────────────┬─────────────┬─────────────┐
  成本归因      质量回溯      漂移告警
  （按 user/    （从坏输出    （embedding +
   feature 切）  回溯 prompt）  eval 联合条件）
```

**这条主线是可观测性从"记录"升级为"洞察"的关键**——没有统一的元数据关联，trace、metrics、eval score 就是三堆互不相干的数据；有了它，才能回答"哪个用户的哪个功能、用哪个 prompt 版本、花了多少钱、质量如何、为什么退化"。

---

## 13. 成熟度评估与最佳实践

### 各能力的成熟度

| 能力 | 成熟度 | 说明 |
|---|---|---|
| 指标 / 延迟 / 吞吐 | ✅ 成熟 | 复用传统 Prometheus/Grafana 栈 |
| 成本 / token 归因 | ✅ 成熟 | 依赖准确价目表 |
| Prompt 版本管理 | ✅ 成熟 | 多平台标准能力 |
| 在线质量评估 | ✅ 渐成熟 | 观测级评估是新方向 |
| 用户反馈闭环 | 🟡 发展中 | 隐式信号利用尚不充分 |
| 护栏对抗鲁棒性 | 🟡 演进中 | 存在高逃逸率风险 |
| 语义漂移检测 | 🟠 早期 | 传统统计难完全捕获 |
| 面向 Agent 的 SLO | 🟠 早期 | "incident"定义仍在重塑 |
| 多 Agent 因果链建模 | 🟠 早期 | 标准未收敛 |

### 最佳实践

1. **建立第四支柱意识**：不要只监控系统健康，要监控输出质量——在线评估是标配而非可选。
2. **元数据主线优先**：从第一天就为每次调用挂上 user/session/feature/prompt-version + token/cost，这是一切归因的地基。
3. **成本设硬上限 + 归因**：网关层熔断防失控，观测层归因找根因。
4. **护栏 + 观测护栏**：护栏本身也要被监控（触发率、逃逸告警），不可 fail open 而无感知。
5. **反馈接入数据集**：把生产失败 case 自动路由进离线测试集，形成"观测→评估→回归门禁"的闭环。
6. **漂移联合告警**：embedding 漂移 + eval 分数下降双条件触发，降低误报、缩短 MTTD。
7. **SLO 而非症状告警**：对用户可感知的影响设错误预算与多窗口 burn rate。
8. **选型看因果链能力**：优先能建模多轮/多 Agent 会话因果链的平台，而非仅有单次 call 级 dashboard。

> **一句话总结**：LLM/Agent 可观测性 = 传统三支柱（指标/日志/追踪）+ 第四支柱"质量"（在线评估）+ 控制层（护栏）+ 人类信号（反馈/标注）+ 时间维度（漂移/回归）+ 资产版本（prompt registry），由一条 user/session/feature/prompt/cost 的元数据主线串联成一个可归因、可回溯、可告警的整体。

---

## 14. 参考资料

**可观测性理念与三/四支柱**
- The three pillars of AI observability — Braintrust — https://www.braintrust.dev/blog/three-pillars-ai-observability
- Three Pillars of Observability: Logs, Metrics and Traces — IBM — https://www.ibm.com/think/insights/observability-pillars
- What is LLM Observability? — Comet — https://www.comet.com/site/blog/llm-observability
- Master LLM Observability — Galileo — https://galileo.ai/blog/understanding-llm-observability

**指标**
- Key metrics for LLM inference — BentoML — https://bentoml.com/llm/llm-inference-basics/llm-inference-metrics
- Understand LLM latency and throughput metrics — Anyscale — https://docs.anyscale.com/llm/serving/benchmarking/metrics
- Metrics — vLLM — https://docs.vllm.ai/en/stable/design/metrics

**成本与 Token**
- LLM Cost Monitoring with OpenTelemetry — Uptrace — https://uptrace.dev/blog/llm-cost-monitoring
- How to track LLM costs 2026 — Braintrust — https://www.braintrust.dev/articles/how-to-track-llm-costs-2026
- Cost — Datadog LLM Observability — https://docs.datadoghq.com/llm_observability/monitoring/cost

**在线质量评估**
- Online vs Offline LLM evaluation — Deepchecks — https://deepchecks.com/question/online-vs-offline-llm-evaluation
- Evaluation of LLM Applications — Langfuse — https://langfuse.com/docs/evaluation/overview
- Observation-level evals — Langfuse — https://langfuse.com/changelog/2026-02-13-observation-level-evals

**护栏与安全**
- AI Guardrails Production Safety Guide 2026 — https://myengineeringpath.dev/genai-engineer/ai-guardrails
- LLM Guardrails: Six Production Safety Layers — https://www.digitalapplied.com/blog/llm-guardrails-production-safety-layers-reference-2026
- Bypassing Prompt Injection and Jailbreak Detection — arXiv 2504.11168 — https://arxiv.org/html/2504.11168v1

**反馈与标注**
- Tracking User Feedback — Traceloop/OpenLLMetry — https://www.traceloop.com/docs/openllmetry/tracing/user-feedback
- Explicit and Implicit LLM User Feedback — Nebuly — https://www.nebuly.com/blog/explicit-implicit-llm-user-feedback-quick-guide
- Annotate traces / Annotation Queues — Datadog — https://www.datadoghq.com/blog/automations-annotation-queues

**漂移与回归**
- Model Drift vs Data Drift 2026 — Future AGI — https://futureagi.com/blog/model-vs-data-drift-how-to-identify-and-handle-it
- 5 methods to detect drift in ML embeddings — Evidently AI — https://www.evidentlyai.com/blog/embedding-drift-detection
- Prompt Drift — Agenta — https://agenta.ai/blog/prompt-drift
- Detecting drift in production — AWS Prescriptive Guidance — https://docs.aws.amazon.com/prescriptive-guidance/latest/gen-ai-lifecycle-operational-excellence/prod-monitoring-drift.html

**Prompt 管理**
- Version Control — Langfuse — https://langfuse.com/docs/prompt-management/features/prompt-version-control
- A/B Testing of LLM Prompts — Langfuse — https://langfuse.com/docs/prompt-management/features/a-b-testing
- What is Prompt Management — LangWatch — https://langwatch.ai/blog/what-is-prompt-management-and-how-to-version-control-deploy-prompts-in-productions

**告警与 SLO**
- SLO-Based Alerting — OpenObserve — https://openobserve.ai/blog/slo-based-alerting
- How to Build SLO Alerting Strategies — OneUptime — https://oneuptime.com/blog/post/2026-01-30-slo-alerting-strategies/view

**平台生态**
- LLM Observability Tools Comparison — Lushbinary — https://lushbinary.com/blog/llm-observability-tools-comparison-langfuse-helicone-phoenix
- 10 LLM Observability Tools 2026 — Confident AI — https://www.confident-ai.com/knowledge-base/compare/10-llm-observability-tools-to-evaluate-and-monitor-ai-2026
- Top 5 Agent Observability Tools 2026 — MLflow — https://mlflow.org/top-5-agent-observability-tools

**本系列相关文档**
- Eval in Langfuse — https://github.com/hszhsz/eval-in-action/blob/main/eval-in-langfuse.md
- Eval in DeepEval — https://github.com/hszhsz/eval-in-action/blob/main/eval-in-DeepEval.md
- AI Agent Eval 综合调研报告 — https://github.com/hszhsz/eval-in-action/blob/main/ai-agent-eval-survey.md
- AI Agent Trace 实现综合调研报告 — https://github.com/hszhsz/eval-in-action/blob/main/ai-agent-trace-survey.md
