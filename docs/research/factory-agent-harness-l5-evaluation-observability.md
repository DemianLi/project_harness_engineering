# Factory Equipment Monitoring Chat Agent: The Evaluation/Observability Layer (L5)

This document investigates evaluation and observability mechanisms for end-user-facing factory equipment monitoring and crisis-handling chat agents. The L5 layer determines how builders and operators gain visibility into agent execution, measure response quality and safety, collect and act on user feedback, and validate agent behavior against domain-specific requirements. Unlike the information boundary (L1), tool system (L2), orchestration mechanics (L3), and memory layer (L4), evaluation/observability is not about what agents can see or do, but about *how builders and operators observe, measure, and improve agent behavior in production*.

Scope: tracing and telemetry mechanics for agent runs (tool invocations, reasoning steps, latency, token usage, error rates); built-in evaluation frameworks for conversation quality and task success; human feedback loops and closed-loop improvement; domain-specific evaluation for equipment-monitoring and crisis-handling scenarios; cost and latency observability; and red-teaming / adversarial testing practices for deployed agents.

---

## Search tooling notes

This pass used `anysearch` for web search, followed by targeted `WebFetch` of official documentation for AWS Bedrock (AgentCore, noting the Classic line's July 2026 sunset), Azure AI Foundry, Google Vertex AI Agent Platform, Microsoft Copilot Studio, and ServiceNow. Primary sources include:

- **AWS Bedrock AgentCore**: Observability documentation, OpenTelemetry-compatible telemetry, built-in metrics (latency, token usage, error rates), CloudWatch integration
- **Azure AI Foundry**: Agent evaluators (system/process/quality evaluation), built-in evaluators for task completion, tool accuracy, intent resolution, etc.
- **Google Vertex AI / Agent Platform**: Evaluation metrics (predefined and custom LLM/code metrics), offline and online evaluation modes, metric registry
- **Microsoft Copilot Studio**: Agent analytics and evaluation (community-sourced documentation; official technical depth limited)
- **ServiceNow Now Assist**: Agentic Evaluations framework, LLM-based judge evaluation, task completeness, tool calling correctness, tool choice accuracy
- **AWS Prescriptive Guidance**: User feedback loops, human-in-the-loop patterns, feedback data pipeline schema
- **NIST AI Agent Security**: Red-teaming findings, agent-specific vulnerabilities, task-hijacking success rates
- **Industrial AI research**: AssetOpsBench (equipment-monitoring agent benchmarking), CausalPulse (manufacturing diagnostics evaluation), SEMAS (multi-agent anomaly detection evaluation)
- **Cost tracking**: AgentMark token/cost observability patterns

Industrial platforms (Siemens, Honeywell, GE) and Microsoft Copilot Studio did not yield vendor-authored technical documentation on evaluation/observability mechanics beyond marketing material; those gaps are noted explicitly. The research specifically looked for documented mechanisms where user feedback or evaluation results feed back into agent improvement or retraining — this pattern was not found as a platform-native feature across any of the five primary platforms checked.

---

## 1. Tracing / telemetry mechanics for agent execution visibility

### AWS Bedrock AgentCore — OpenTelemetry-compatible observability with CloudWatch

Per [Observe your agent applications on Amazon Bedrock AgentCore Observability - AWS](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability.html), AgentCore provides observability for tracing, debugging, and monitoring agent performance. The key capability is that "AgentCore emits telemetry data in standardized OpenTelemetry (OTEL)-compatible format, enabling you to easily integrate it with your existing monitoring and observability stack."

Metrics exposed by default (per the same documentation):

- **Key operational metrics**: session count, latency, duration, token usage, and error rates
- **Rich metadata tagging and filtering**: to simplify issue investigation and quality maintenance at scale
- **Three metric categories**: built-in metrics for agents, gateway resources, and memory resources (memory resources also output spans and log data if enabled)

Storage and access:

Per the documentation, "All of the metrics, spans, and logs output by AgentCore are stored in Amazon CloudWatch, and can be viewed in the CloudWatch console or downloaded from CloudWatch using the AWS CLI or one of the AWS SDKs." Additionally, "for agent runtime data only, the CloudWatch console provides an observability dashboard containing trace visualizations, graphs for custom span metrics, error breakdowns, and more."

**Default-direction**: Built-in metrics are enabled by default; custom instrumentation is opt-in.

### Azure AI Foundry — conversation history with implicit tracing via conversations

Per [Build with agents, conversations, and responses in Foundry Agent Service - Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/runtime-components), Foundry stores "conversations, which are durable objects with unique identifiers" that "store items, which can include messages, tool calls, tool outputs, and other data." The conversation itself serves as the tracing artifact; responses can be chained via `previous_response_id` for multi-turn tracing.

**Tracing format**: Not explicitly documented as OpenTelemetry-compatible. Traces are conversation artifacts (messages, tool calls, outputs) persisted server-side.

### Google Vertex AI Agent Platform — trace export and evaluation integration

Per [Manage evaluation metrics - Gemini Enterprise Agent Platform](https://docs.cloud.google.com/gemini-enterprise-agent-platform/optimize/evaluation/manage-metrics), Google provides "Metric Registry" for managing evaluation metrics, but explicit documentation on trace/telemetry export mechanics for raw execution visibility was not found in the sources checked.

**Not found in verified sources**: Specific tracing format (OpenTelemetry or proprietary), default vs. opt-in status for telemetry emission.

### Microsoft Copilot Studio — conversation transcripts (format not detailed)

Per [Agent analytics (preview) - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/analytics-agent-evaluation-overview), Copilot Studio provides agent analytics including conversation counts, turn counts, and session data. However, detailed mechanics of how tracing data is stored, formatted, or exported were not documented in verified sources checked.

### ServiceNow Now Assist — execution logs as tracing artifact

Per [Agentic Evaluations FAQ - ServiceNow](https://www.servicenow.com/community/now-assist-articles/agentic-evaluations-faq/ta-p/3429085) (community-sourced, not official documentation), ServiceNow describes "execution logs—complete records of AI agent actions when completing a task, including every tool selection, parameter, and decision made." The documentation indicates these logs are queryable for evaluation, but the storage format and export mechanics are not detailed in the sources checked.

### Real-time observability vs. post-hoc analysis

Across all platforms checked:
- **Default**: Tracing is primarily accessed post-hoc (after execution completes), stored in backend services (CloudWatch, conversation records, execution logs).
- **Real-time dashboards**: AWS AgentCore explicitly provides real-time CloudWatch dashboards; others do not document real-time visualization.
- **Export/integration**: Only AWS explicitly documents industry-standard OpenTelemetry format; others are platform-proprietary.

---

## 2. Conversation-quality / accuracy evaluation mechanisms

### Azure AI Foundry — system and process evaluators as "unit tests"

Per [Agent Evaluators for Generative AI - Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/concepts/evaluation-evaluators/agent-evaluators), Foundry provides built-in evaluators that "function like unit tests for agentic systems—they take agent messages as input and output binary Pass/Fail scores (or scaled scores converted to binary scores based on thresholds)."

**System evaluation** (end-to-end outcome assessment):
- Task Completion (preview) — measures if the agent completed the requested task with a usable deliverable meeting all user requirements
- Customer Satisfaction (preview) — measures holistic user satisfaction across six dimensions (helpfulness, completeness, clarity, tone, resolution, adaptability)
- Task Adherence (preview) — measures if agent actions adhere to assigned tasks per system message and rules
- Intent Resolution (preview) — measures whether agent correctly identifies user intent

**Process evaluation** (step-by-step execution assessment):
- Tool Call Accuracy — measures if agent made right tool calls with correct parameters
- Tool Selection — measures if agent selected correct tools without redundancy
- Tool Input Accuracy — measures if tool call parameters are correct per six strict criteria (groundedness, type compliance, format compliance, required parameters, unexpected parameters, value appropriateness)
- Tool Output Utilization — measures if agent correctly understood and used tool results contextually

Per the documentation, evaluators require "LLM judge" models (supported: Azure OpenAI or OpenAI reasoning models; gpt-5-mini recommended for balance of performance, cost, and efficiency).

### Google Vertex AI — Metric Registry with predefined and custom metrics

Per [Manage evaluation metrics - Gemini Enterprise Agent Platform](https://docs.cloud.google.com/gemini-enterprise-agent-platform/optimize/evaluation/manage-metrics), Vertex AI provides:

**Predefined Metrics** (managed by Google):
- Single-turn: Agent Final Response Quality, Agent Hallucination, Agent Tool Use Quality, Safety
- Multi-turn: Agent Multi-turn Task Success, Agent Multi-turn Tool Use Quality, Agent Multi-turn Trajectory Quality

**Custom Metrics**:
- Custom LLM Metrics — natural language rubrics evaluated by a judge LLM with rating scales
- Custom Code Metrics — Python functions for programmatic validation (e.g., output format verification)

**Evaluation modes**: Offline (batch) and online (continuous monitors) supported.

### ServiceNow — LLM-based judge evaluation of execution logs

Per the Agentic Evaluations FAQ, ServiceNow employs "LLM-based judges to evaluate your agent's execution logs" using ServiceNow's Now LLM as the judge model. The framework analyzes "every tool selection, every parameter passed, every step in the workflow."

**Core metrics**:
1. Overall Task Completeness — validates whether workflows accomplished assigned tasks
2. Tool Calling Correctness — ensures agents construct tool calls with accurate parameters and required fields
3. Tool Choice Accuracy — confirms agents select appropriate tools at each decision point

**Evaluation data**: Can use existing logs, generate new logs in AI Agent Studio, or autonomously run workflows to create evaluation datasets.

### AWS Bedrock AgentCore — evaluation not documented in verified sources

Per the AgentCore observability documentation reviewed, built-in evaluation frameworks for quality/accuracy assessment were not detailed in the sources checked. (AWS Bedrock Classic, the prior product line, documented guardrails and content moderation but not structured quality evaluators like Foundry's.)

### Microsoft Copilot Studio — quality evaluation (analytics-level only)

Per [Agent analytics - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/analytics-agent-evaluation-overview), Copilot Studio provides analytics on conversation counts, turn counts, and user satisfaction metrics (community-sourced detail: top-bot evaluation on conversation level). However, detailed mechanisms for evaluating individual agent responses or tool-call quality are not documented in verified sources.

---

## 3. Human feedback / human review loops

### AWS — structured feedback pipeline with traceability

Per [Architecting the production feedback loops - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/gen-ai-lifecycle-operational-excellence/prod-monitoring-feedback.html), AWS documents a comprehensive feedback architecture:

**User feedback collection**:
- Mechanism: "implement a feedback mechanism that captures user feedback when the application is not functioning as intended. This can be as simple as a binary good or not-good response. It could also be a more comprehensive open-text system where users can provide verbose feedback."
- Workflow: (1) User performs action in application, (2) Application prompts for feedback, (3) User provides feedback (thumbs-up/down or commentary), (4) Human reviewer evaluates feedback, (5) Human reviewer fine-tunes model and updates application, (6) Updated application deployed to production.

**Feedback data schema** (essential fields):
- `feedback_id` — unique identifier for feedback event
- `trace_id` — **critical**: foreign key linking to end-to-end application trace, allowing retrieval of prompt, retrieved documents, model response, latency, model version, prompt template version, tool calls
- `timestamp` — when feedback submitted
- `user_id` — identifier for user providing feedback
- `feedback_type` — enum indicating source (e.g., explicit_thumb, implicit_copy, explicit_comment)
- `feedback_value` — value of feedback (e.g., 1 for thumbs up, 0 for thumbs down, 1-5 score)
- `feedback_comment` — optional free-text user comments

The documentation states: "A foundation of continuous improvement is a well-architected data pipeline that captures and centralizes all forms of user feedback. This feedback is the most valuable source of ground truth data about the application's real-world performance."

**Human-in-the-loop for high-risk actions**:
The guidance prescribes allowing agents to make automated actions only in low-risk scenarios, with "a human-in-the-loop mechanism for high-risk or unfamiliar scenarios not previously covered in testing scenarios."

### Azure AI Foundry — feedback via evaluation

The documentation references evaluation results (from agent evaluators, see §2) as the feedback loop, but does not document a mechanism for end users to provide feedback or for that feedback to feed back into model improvement.

### Google Vertex AI — feedback not documented in verified sources

Specific mechanisms for collecting user feedback and closing improvement loops are not detailed in the sources checked.

### ServiceNow — feedback via escalation (documented indirectly)

Per L3 research, ServiceNow supports "Supervised Execution mode" where human oversight is required before tool execution, and escalation via `vaSystem.connectToAgent()` transfers conversation to a live agent. However, a documented pattern for human feedback from post-escalation interaction feeding back into agent learning is not found in verified sources.

### All platforms — post-human-resolution learning not found

**Key finding**: Across AWS, Azure, Google, Microsoft, and ServiceNow, no official documentation describes a pattern where agents "learn" from the human's resolution after escalation or handoff. The feedback loop documented in AWS prescriptive guidance is uni-directional: users → feedback collection → human review → model fine-tuning → deployment. But the specific mechanism for agents extracting insights from human corrections or resolution steps is not documented as a platform-native capability.

---

## 4. Domain-specific evaluation for equipment monitoring / crisis handling

### AssetOpsBench — benchmark framework for industrial asset operations

Per [AssetOpsBench: Benchmarking AI Agents for Task Automation in Industrial Asset Operations and Maintenance - arXiv 2506.03828](https://doi.org/10.48550/arxiv.2506.03828), the paper introduces "AssetOpsBench, a unified framework for orchestrating and evaluating domain-specific agents for Industry 4.0." The paper defines evaluation across the full asset lifecycle:

- Condition monitoring and fault detection (anomaly detection accuracy)
- Maintenance planning (scheduling quality)
- Intervention scheduling (timing and resource allocation)
- Root-cause analysis (diagnostic accuracy and explanation quality)

The paper frames the underlying challenge this way: "The ability to monitor and interpret heterogeneous data from diverse sources, such as IoT SCADA sensors, operational KPIs, failure mode libraries, maintenance work orders, and technical manuals, is key to effective Asset Lifecycle Management." The framework's evaluation is built around agents that must work across these heterogeneous, multimodal data sources to complete asset-lifecycle tasks.

### CausalPulse — neurosymbolic multi-agent evaluation for manufacturing

Per [CausalPulse: An Industrial-Grade Neurosymbolic Multi-Agent Copilot for Causal Diagnostics in Smart Manufacturing - arXiv 2603.29755](https://arxiv.org/html/2603.29755), CausalPulse is "being deployed in a Robert Bosch manufacturing plant" and evaluated using specific domain metrics:

**Evaluation results on industrial benchmarks** (Future Factories and proprietary Planar Sensor Element datasets):
- Overall success rate: 98.0% and 98.73%
- Per-criterion success rates: 98.75% for planning and tool use, 97.3% for self-reflection, 99.2% for collaboration
- Runtime: 50-60 seconds per diagnostic workflow, near-linear scalability (R²=0.97)

The paper evaluates four specific agentic capabilities — planning, tool use, self-reflection, and multi-agent collaboration — computing "three quantitative metrics, Criterion success (%), Aggregate criterion success (%) per stage, and the Aggregate success rate per agentic criterion" for each. These are framed as assessments of agentic capabilities/criteria specific to the diagnostic task, not generic language-quality metrics.

### Hitachi fault diagnosis evaluation framework

Per [Exploring LLM-based Agentic Frameworks for Fault Diagnosis - Hitachi/Annual Conference of the PHM Society](https://papers.phmsociety.org/index.php/phmconf/article/view/4350), Hitachi's research framework studies LLM agents diagnosing faults while "producing inherently explainable outputs through natural language reasoning. Such explainability enables users to interpret and audit agent decisions." The evaluation methodology includes:

- Fault detection accuracy (across varied degradation scenarios)
- Uncertainty estimation calibration (confidence alignment with correctness)
- Continual learning / confidence calibration over time based on historical ground truth outcomes

The abstract frames explainability and interpretability as features of the LLM-based approach that let users interpret and audit agent decisions, rather than as formally named, separately scored evaluation criteria.

### SEMAS — self-evolving multi-agent system evaluation

Per [Self-Evolving Multi-Agent Network for Industrial IoT Predictive Maintenance - arXiv 2602.16738](https://arxiv.org/html/2602.16738), the paper introduces "SEMAS, a self-evolving hierarchical multi-agent system" (the paper does not spell out SEMAS as a formal acronym beyond this description), evaluated across:

- Anomaly detection performance (prediction accuracy across evolving operational contexts)
- Stability under adaptation (the abstract claims "exceptional stability under adaptation")
- Latency / real-time capability ("substantial latency improvements enabling genuine real-time deployment" — the abstract does not give a specific sub-second figure)
- Interpretability (explainability of agent decisions)

### Domain-specific gaps across platforms

**Key finding**: No official vendor documentation from AWS, Azure, Google, Microsoft, or ServiceNow describes domain-specific evaluation frameworks for equipment monitoring or crisis-handling agents. The industrial research literature (AssetOpsBench, CausalPulse, SEMAS, Hitachi) describes evaluation approaches at the *research level*, but platforms do not expose specialized evaluators for:

- Equipment fault detection accuracy
- Severity triage correctness (is this a crisis or routine issue?)
- Crisis escalation appropriateness
- Root-cause analysis quality in equipment diagnostics

Instead, platforms offer generic evaluators (task completion, tool accuracy, intent resolution). Domain-specific evaluation is builder-implemented custom logic, not platform-level capability.

---

## 5. Cost / latency observability specific to production chat agents

### AWS Bedrock AgentCore — token usage and duration in built-in metrics

Per the AgentCore observability documentation, built-in metrics include "token usage" and "duration" as first-class telemetry. Per [Amazon Bedrock AgentCore generated observability data - AWS](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability-service-provided.html) (referenced in search results; specific content not fetched, but documented as providing token counts, latency distribution, error rates), these metrics are available per-session and aggregated across all agent invocations.

**Cost attribution**: Token usage is exported; cost calculation (tokens × model pricing) is builder-implemented, not platform-calculated.

### AgentMark — third-party cost tracking pattern

Per [Cost and token tracking - AgentMark Docs](https://docs.agentmark.co/observe/cost-and-token-tracking), AgentMark "automatically tracks costs and token usage for every LLM call" by:
- Recording input tokens, output tokens, total tokens
- Calculating cost as: `cost = (input_tokens × input_price_per_token) + (output_tokens × output_price_per_token)`
- Using exact token counts from LLM provider API responses (not estimation)
- Tracking reasoning tokens separately for models supporting chain-of-thought (OpenAI o1, o3)

This pattern is third-party (not official vendor), but demonstrates a standard approach: token counts per span, pricing lookups per model, aggregation at trace/dashboard level.

### Azure AI Foundry — cost tracking not explicitly documented

Evaluation results and operation metrics are documented; explicit cost attribution per conversation or per tenant is not detailed in verified sources.

### Google Vertex AI — cost tracking not explicitly documented

Token usage and cost tracking are not detailed in verified sources checked.

### Microsoft Copilot Studio — cost/performance analytics

Per community-sourced documentation and agent analytics pages, Copilot Studio provides conversation counts and turn counts, but explicit cost/token tracking is not documented in official sources.

### ServiceNow — cost tracking not documented

**Not found in verified sources**: Official documentation on cost attribution per conversation, per tenant, or per agent invocation.

### Latency SLA monitoring for crisis scenarios

**Not found in verified sources**: No platform explicitly documents latency SLA monitoring or timeout/performance-degradation alerting specific to crisis/incident-mode conversations (e.g., response time budgets for equipment shutdowns or emergency escalations).

---

## 6. Red-teaming / adversarial testing for deployed conversational agents

### NIST AI Agent Security — agent-specific red-teaming findings

Per [NIST AI Agent Security: Red-Teaming Guidance and Enterprise Compliance - Cloud Security Alliance](https://labs.cloudsecurityalliance.org/research/csa-research-note-nist-ai-agent-red-teaming-standards-202603/) (published 2026-03-31), NIST's Center for AI Standards and Innovation (CAISI) launched the AI Agent Standards Initiative on 2026-02-17, establishing "the first time NIST has established a dedicated organizational initiative around agent security as a category."

**Red-teaming research findings**:
- Novel agent-specific attack techniques achieved 81% task-hijacking success rate vs. 11% for strongest baseline attacks (conventional LLM attacks)
- With 25 repeated attempts per task, average attack success rates climbed from 57% to 80%, showing that LLM non-determinism requires probabilistic rather than single-shot testing
- March 2025 update to NIST AI 100-2 (Adversarial ML Taxonomy) "extended the adversarial ML attack taxonomy to cover autonomous AI agent vulnerabilities for the first time, including indirect prompt injection, agent memory poisoning, and supply chain attacks on agent tools"

**Agent-specific threat categories**:
1. Indirect Prompt Injection — adversarial instructions embedded in documents, emails, retrieved data that agents act upon
2. Memory Poisoning — gradual corruption of agent knowledge bases through compromised data sources
3. Authorization Bypass — agents inheriting excessive permissions or operating under generic credentials
4. Dynamic Tool-Switching — expanding attack surface as agents access different systems per session
5. Specification Gaming — agents optimizing measurable outcomes in ways violating original intent
6. Autonomous Real-World Actions — lacking human oversight mechanisms present in static model inference

**Methodological principles** (per NIST research):
1. Continuous Iteration — single benchmark scores provide false security confidence
2. Novel Technique Superiority — exercises relying on existing tools/standard playbooks mask real attack surface
3. Task-Specific Variation — aggregate success statistics hide important differences; different injection tasks present different risk profiles
4. Multi-Attempt Modeling — probabilistic rather than single-shot protocols required

### Platform-specific red-teaming documentation

**AWS Bedrock**: No specific red-teaming harness or adversarial testing service documented in verified sources.

**Azure AI Foundry**: No specific red-teaming harness documented; Azure Resilience Assessment (AIRA) is mentioned in some community posts but technical details not found in official docs.

**Google Vertex AI**: No specific red-teaming service documented in verified sources.

**Microsoft Copilot Studio**: No specific red-teaming harness documented.

**ServiceNow**: No specific red-teaming service documented.

### Status of platform-native red-teaming

**Key finding**: No official vendor documentation from any of the five platforms (AWS, Azure, Google, Microsoft, ServiceNow) describes a first-class, built-in red-teaming or adversarial-testing harness specifically designed for deployed conversational agents. Red-teaming for agents is documented as a *research area* (NIST) and a *recommendation* (build it yourself, use novel agent-specific techniques), not as a platform capability. This stands in contrast to static LLM red-teaming tools, which some vendors offer.

---

## Comparison: evaluation/observability mechanisms across platforms

| Platform | Tracing format | Quality evaluators | Human feedback loop | Domain-specific eval | Cost/latency observability | Red-teaming |
|---|---|---|---|---|---|---|
| **AWS Bedrock AgentCore** | OpenTelemetry-compatible, CloudWatch storage, real-time dashboard | Not documented | Prescriptive guidance: user feedback → human review → fine-tuning (no closed loop documented) | Not documented | Token usage and duration in built-in metrics; cost calculation builder-implemented | Not documented |
| **Azure AI Foundry** | Conversation artifacts (messages, tool calls, outputs); implicit tracing via conversation records | 11 built-in evaluators (System: Task Completion, Customer Satisfaction, Task Adherence, Intent Resolution, Task Navigation Efficiency; Process: Tool Call Accuracy, Tool Selection, Tool Input Accuracy, Tool Output Utilization, Tool Call Success; Quality: Quality Grader); LLM judge model (gpt-5-mini recommended) | Feedback loop documented in evaluation results; no end-user feedback mechanism documented | Not documented | Not explicitly documented | Not documented |
| **Google Vertex AI / Agent Platform** | Not explicitly documented | Predefined metrics (single-turn: Final Response Quality, Hallucination, Tool Use Quality, Safety; multi-turn: Task Success, Tool Use Quality, Trajectory Quality); Custom LLM and Code metrics | Not documented | Not documented | Not explicitly documented | Not documented |
| **Microsoft Copilot Studio** | Implicit conversation records; conversation transcripts | Agent analytics (conversation counts, turn counts, user satisfaction); Quality Grader (same as Azure Foundry, used in analytics) | Not documented in verified sources | Not documented | Not documented | Not documented |
| **ServiceNow Now Assist** | Execution logs (tool selections, parameters, decisions); queryable via LLM-based judge | Agentic Evaluations: 3 metrics (Overall Task Completeness, Tool Calling Correctness, Tool Choice Accuracy); LLM judge = ServiceNow Now LLM | Escalation to live agent documented; post-human-resolution feedback not documented | Not documented | Not documented | Not documented |
| **NIST / Industry Research** | N/A (guidance tier) | N/A | N/A | AssetOpsBench, CausalPulse, SEMAS evaluate task success, tool use, self-reflection, collaboration; Hitachi framework evaluates fault detection accuracy and uncertainty calibration | N/A | Agent-specific red-teaming taxonomy; 81% task-hijacking success with novel techniques vs. 11% baseline |

---

## Key gaps: areas where primary-source material was not found

- **Real-time telemetry streaming for AWS, Azure, Google, Microsoft, ServiceNow** — only AWS explicitly documents real-time CloudWatch dashboards; others' telemetry access patterns are batch or implicit.
- **OpenTelemetry adoption across platforms** — only AWS explicitly documents OTEL compatibility; other platforms' telemetry formats (if exported) are not specified in verified sources.
- **Closed-loop learning from user feedback** — AWS prescribes the pipeline (feedback → human review → fine-tuning → deployment); no platform documents automatic feedback incorporation or A/B testing of model updates based on production feedback.
- **Domain-specific evaluation frameworks** — platforms offer generic evaluators (task completion, tool accuracy, intent resolution); none document specialized evaluators for equipment fault detection, severity triage, or crisis escalation. Domain-specific evaluation in the research literature (AssetOpsBench, CausalPulse) is research-tier, not platform-integrated.
- **Crisis/incident-mode evaluation changes** — consistent with L3 and L4 findings, no platform documents changing evaluation criteria, thresholds, or SLA requirements based on runtime detection of crisis/emergency.
- **Cost attribution per tenant / per conversation** — no platform explicitly documents cost tracking or billing attribution per multi-tenant conversation or per customer.
- **Latency SLA monitoring and crisis-mode timeouts** — no platform documents performance-based alerting or timeout policies specific to equipment-monitoring or crisis-handling agents (e.g., <5s response time for emergency escalation).
- **Platform-native red-teaming for conversational agents** — NIST has published threat taxonomy and red-teaming methodology; no vendor provides built-in red-teaming harness or adversarial-testing service integrated into their agent platform.
- **Industrial platforms (Siemens, Honeywell, GE)** — no public technical documentation on evaluation/observability mechanisms located.
- **Microsoft Copilot Studio evaluation details** — only high-level analytics (conversation counts, turn counts) documented; detailed evaluator capabilities and quality assessment mechanics not found in verified sources.

---

## Observations

1. **Observability varies by platform maturity and default settings.** AWS AgentCore provides OpenTelemetry-compatible telemetry with real-time dashboards by default; others expose tracing implicitly via conversation records or execution logs. Tracing format standardization (OpenTelemetry) is not universal.

2. **Quality evaluation is now a documented platform feature (Azure, Google, ServiceNow), but evaluation granularity differs.** Azure's 11 built-in evaluators and ServiceNow's 3-metric framework provide structured, LLM-judge-based assessment. However, "evaluation" typically addresses task completion and tool accuracy, not domain-specific goals (equipment fault identification, crisis triage, root-cause explanations).

3. **Human feedback loops are documented in prescriptive guidance (AWS) but not as a platform-native closed loop.** AWS prescribes feedback collection → human review → model fine-tuning → redeployment. No platform documents automatic feedback incorporation or continuous retraining triggered by production feedback.

4. **Domain-specific evaluation for equipment monitoring is research-tier, not platform-integrated.** Academic and industrial research (AssetOpsBench, CausalPulse, SEMAS, Hitachi) defines evaluation for fault detection accuracy, severity triage, and interpretability. Platforms do not expose these as first-class evaluators; builders must implement custom logic.

5. **Red-teaming for agents is identified as a critical need (NIST), but no platform provides built-in capability.** NIST research demonstrates agent-specific attacks (indirect injection, memory poisoning, authorization bypass) are far more successful than generic LLM attacks. Platforms document static LLM red-teaming tools, if any; agent-specific red-teaming is a gap.

6. **Cost and latency observability are underdocumented.** Token usage is tracked; cost calculation is builder-implemented. Latency is a built-in metric (AWS, others implicit), but SLA monitoring, timeout policies, and cost attribution per tenant or conversation are not documented.

7. **Post-human-resolution learning is not a documented platform pattern.** Across all five platforms, feedback flows one direction: users → platform capture → human review. Whether agents extract insights from human corrections during handoff, or whether resolution outcomes feed back into long-term memory or retraining, is not documented as a platform capability.

8. **Evaluation is mostly asynchronous and offline.** Quality evaluators (Azure, Google, ServiceNow) assess after execution completes, not in real-time. This suits batch assessment and regression testing but not runtime quality gates or adaptive response generation.

---

## Primary sources referenced

### AWS Bedrock (AgentCore)

- [Observe your agent applications on Amazon Bedrock AgentCore Observability - AWS](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability.html) — telemetry, OpenTelemetry format, CloudWatch integration, built-in metrics
- [Amazon Bedrock AgentCore generated observability data - AWS](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability-service-provided.html) — token usage, latency, duration, error rates, tracing
- [Architecting the production feedback loops - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/gen-ai-lifecycle-operational-excellence/prod-monitoring-feedback.html) — user feedback loop, human-in-the-loop patterns, feedback data schema (trace_id, feedback_type, feedback_value)

### Azure AI Foundry

- [Agent Evaluators for Generative AI - Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/concepts/evaluation-evaluators/agent-evaluators) — system/process/quality evaluators, Task Completion, Intent Resolution, Tool Accuracy, Quality Grader, LLM judge model
- [Build with agents, conversations, and responses in Foundry Agent Service - Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/runtime-components) — conversation persistence, tracing via conversation/response records

### Google Vertex AI / Agent Platform

- [Manage evaluation metrics - Gemini Enterprise Agent Platform](https://docs.cloud.google.com/gemini-enterprise-agent-platform/optimize/evaluation/manage-metrics) — predefined metrics (single-turn, multi-turn), custom LLM metrics, custom code metrics, Metric Registry, offline/online evaluation

### Microsoft Copilot Studio

- [Agent analytics (preview) - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/analytics-agent-evaluation-overview) — conversation counts, turn counts, user satisfaction metrics (community-sourced detail from prior research)

### ServiceNow Now Assist

- [Agentic Evaluations FAQ - ServiceNow Community](https://www.servicenow.com/community/now-assist-articles/agentic-evaluations-faq/ta-p/3429085) — Agentic Evaluations framework, LLM-based judge (ServiceNow Now LLM), three metrics (Task Completeness, Tool Calling Correctness, Tool Choice Accuracy), execution logs

### AI Safety and Red-Teaming

- [NIST AI Agent Security: Red-Teaming Guidance and Enterprise Compliance - Cloud Security Alliance](https://labs.cloudsecurityalliance.org/research/csa-research-note-nist-ai-agent-red-teaming-standards-202603/) — agent-specific red-teaming findings, task-hijacking success rates (81% vs. 11% baseline), six threat categories (indirect injection, memory poisoning, authorization bypass, dynamic tool-switching, specification gaming, autonomous actions), NIST AI Agent Standards Initiative

### Industrial AI and Domain-Specific Evaluation

- [AssetOpsBench: Benchmarking AI Agents for Task Automation in Industrial Asset Operations and Maintenance - arXiv 2506.03828](https://doi.org/10.48550/arxiv.2506.03828) — evaluation framework for industry 4.0 agents, condition monitoring, maintenance planning, root-cause analysis evaluation
- [CausalPulse: An Industrial-Grade Neurosymbolic Multi-Agent Copilot for Causal Diagnostics in Smart Manufacturing - arXiv 2603.29755](https://arxiv.org/html/2603.29755) — manufacturing diagnostics evaluation, 98%+ success rates on Bosch deployment, per-criterion metrics (planning, tool use, self-reflection, collaboration), 50-60s latency
- [Exploring LLM-based Agentic Frameworks for Fault Diagnosis - Hitachi/Annual Conference of the PHM Society](https://papers.phmsociety.org/index.php/phmconf/article/view/4350) — fault detection accuracy, uncertainty calibration, continual learning in equipment diagnostics
- [Self-Evolving Multi-Agent Network for Industrial IoT Predictive Maintenance - arXiv 2602.16738](https://arxiv.org/html/2602.16738) — SEMAS framework, anomaly detection performance, stability under adaptation, real-time latency, interpretability evaluation

### Cost Tracking and Observability Patterns

- [Cost and token tracking - AgentMark Docs](https://docs.agentmark.co/observe/cost-and-token-tracking) — third-party pattern: token counts, cost calculation, provider pricing tables, reasoning tokens (third-party implementation, not official vendor)
