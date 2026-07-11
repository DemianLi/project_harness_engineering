# Factory Equipment Monitoring Chat Agent: The Constraint/Validation/Recovery Layer (L6)

This document investigates constraint enforcement, input/output validation, and failure recovery mechanisms for end-user-facing factory equipment monitoring and crisis-handling chat agents. The L6 layer determines what safeguards are applied to agent inputs, outputs, and actions, and what happens when things go wrong. Unlike the information boundary (L1), tool system (L2), orchestration mechanics (L3), memory layer (L4), and evaluation/observability (L5), the constraint/validation/recovery layer is where the last line of defense operates — after an agent has planned an action, but before it executes or surfaces results to the user.

Scope: output/response validation and guardrails (content moderation, hallucination detection, PII redaction, topic/scope restriction); input validation and injection defense (user prompt and tool-output sanitization); hard vs. soft constraint enforcement (infrastructure-level blocks vs. model-level guidance); failure/error recovery mechanics (retry logic, fallback strategies, graceful degradation); crisis/safety-critical validation specifics (whether validation tightens for emergencies); and rollback/compensating-action mechanics for reversing incorrect agent actions.

---

## Search tooling notes

This pass used `anysearch` for web search and targeted `WebFetch` of official vendor documentation for AWS Bedrock, Azure AI Foundry, Google Vertex AI, Microsoft Copilot Studio, and ServiceNow. Academic research papers on constraint enforcement (SARC, BlueGate, AgentSpec, Agent-C) and open-source patterns on tool-output validation, error recovery, and prompt injection defense were also reviewed. Primary sources include:

- **AWS Bedrock Guardrails**: ApplyGuardrail API, guardrail configuration mechanics, content filters, PII redaction, contextual grounding (hallucination detection), validation flow and intervention points
- **Azure AI Foundry Guardrails**: System guardrails, intervention points (user input, tool call, tool response, output), content filtering, safety control mechanisms
- **Google Vertex AI**: Safety filters, configurable thresholds, prompt/response feedback codes
- **Microsoft Copilot Studio**: Content moderation (limited technical detail in verified sources)
- **ServiceNow Now Assist**: Tool validation patterns, parameter validation, agentic evaluations framework
- **AWS Prescriptive Guidance**: Error handling, automated recovery, fallback strategies
- **AWS Step Functions**: Saga pattern and compensating transactions for distributed workflows
- **Academic Research**: SARC (governance-by-architecture), BlueGate (hard vs. soft constraint classification), AgentSpec (runtime enforcement DSL), Agent-C (temporal constraints)
- **Prompt injection defense patterns**: Tool output poisoning, indirect injection defense, result parsing and trust labeling

Industrial platforms (Siemens, Honeywell, GE) and comprehensive Microsoft Copilot Studio technical depth on constraint/validation mechanics were not found in verified public sources.

---

## 1. Output/response validation and guardrails

### AWS Bedrock Guardrails — multi-layer safeguard architecture

Per [Detect and filter harmful content by using Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html) and [Use the ApplyGuardrail API in your application - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-use-independent-api.html), Bedrock Guardrails provides six configurable safeguards:

1. **Content filters** — Detect and filter harmful content categories (Hate, Insults, Sexual, Violence, Misconduct, Prompt Attack) based on configurable filter strength (NONE, LOW, MEDIUM, HIGH). Can be applied to both prompts and completions independently.

2. **Denied topics** — Builder-defined topics that are blocked if detected in user queries or model responses. With Standard tier, detection extends into code elements (comments, variable names, function names, string literals).

3. **Word filters** — Custom words or phrases (exact match) and managed word lists (e.g., profanity). Blocked on detection.

4. **Sensitive information filters** — PII detection (SSN, Date of Birth, Address, phone, email, credit card, etc.) with configurable action: BLOCKED or ANONYMIZED. Also supports regex-based custom patterns.

5. **Contextual grounding checks** — Detects hallucinations in RAG applications by checking if model responses deviate from retrieved sources or fail to answer the user's question. Measured via grounding score thresholds.

6. **Automated Reasoning checks** — Validates model responses against logical rules using formal logic; the guardrails.html page itself describes this filter as helping "validate the accuracy of foundation model responses against a set of logical rules" to "detect hallucinations, suggest corrections, and highlight unstated assumptions." Two separate AWS blog posts add marketing framing not found on the technical docs page: the preview announcement states the feature "is the first and only generative AI safeguard that helps prevent factual errors due to hallucinations using logically accurate and verifiable reasoning," and the general-availability post states it "delivers up to 99% verification accuracy, providing provable assurance in detecting AI hallucinations."

**Intervention points** (per the documentation): These safeguards can be applied at four points in the agent flow:
- User input (before processing)
- Tool call (before the agent invokes a tool — agent-only)
- Tool response (after a tool returns, before the agent reasons about it — agent-only)
- Output (final model completion before returning to user)

**Default-direction**: Guardrails are opt-in; builders must explicitly create and apply them. When applied to a model or agent, they are enabled by default at their configured thresholds.

**ApplyGuardrail API**: Builders can decouple guardrails from model invocation and use `ApplyGuardrail` independently to assess any text input or output. The API returns action ("GUARDRAIL_INTERVENED" or "NONE"), output (masked or blocked content, or empty if no intervention), and detailed assessments per safeguard type.

### Azure AI Foundry Guardrails — intervention-point driven architecture

Per [Guardrails and controls overview in Microsoft Foundry - Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/guardrails/guardrails-overview), Azure Foundry Guardrails consists of named collections of "controls" that define a risk, intervention points to scan, and response actions. Four intervention points are supported (same as AWS):

- User input (prompt to model or agent)
- Tool call (agent proposal to call a tool)
- Tool response (content returned from tool to agent)
- Output (final completion to user)

**Risk categories and controls**: Guardrails cover Hate, Sexual, Self-harm, Violence, User prompt attacks, Indirect attacks (tool-output injection), Spotlighting (preview, models only), Protected material for code, Protected material for text, and Groundedness (preview, models only).

**Groundedness control**: Available for models and agents to detect hallucinations and irrelevance in RAG systems, specifically flagging content that deviates from or isn't grounded in the source material.

Per [Default Guardrail policies for Azure OpenAI - Microsoft Foundry Learn](https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/default-safety-policies), default severity thresholds are set to "Medium" across content categories (hate, violence, sexual, self-harm) for both prompts and completions. Content at low/safe severity is labeled but not filtered; medium and high are filtered by default (unless customer has "Approval" for full control via Limited Access Review).

**Configurable vs. non-configurable**: Hate, Violence, Sexual, Self-harm, User prompt attacks, and Indirect attacks are configurable per severity threshold. CSAM (Child Sexual Abuse Material) and Copyright Recitation protections are non-configurable.

### Google Vertex AI / Gemini — configurable content filters, default OFF for current models

Per [Safety settings | Gemini API - Google AI for Developers](https://ai.google.dev/gemini-api/docs/safety-settings), Gemini provides four configurable harm categories (harassment, hate speech, sexually explicit, dangerous content) with threshold options `BLOCK_LOW_AND_ABOVE` (strictest), `BLOCK_MEDIUM_AND_ABOVE`, and `BLOCK_ONLY_HIGH`.

**Default-direction**: the page states verbatim, "If the threshold is not set, the default block threshold is Off for Gemini 2.5 and 3 models." This means the configurable filters do **not** block anything unless the builder explicitly sets a threshold — the opposite direction from assuming a filtered-by-default posture. This is a materially different default than AWS Bedrock Guardrails (opt-in, but once applied, active at configured thresholds) and Azure Foundry (Medium severity by default across categories); a factory-agent builder using current Gemini models must explicitly configure thresholds to get any content-filter blocking at all.

**Non-configurable filters** (always on regardless of configuration): per the same page, "the Gemini API has built-in protections against core harms, such as content that endangers child safety. These types of harm are always blocked and cannot be adjusted."

**Prompt and response feedback**: the page describes a `SAFETY` finish reason indicating the model stopped generation due to safety filters. A `blockReason` enum value of `PROHIBITED_CONTENT` was referenced in secondary sources for the Vertex AI (Gemini Enterprise Agent Platform) product surface specifically, but was not independently confirmed on an official page fetched directly in this pass — treat that specific enum name as unconfirmed pending a direct check of Vertex-specific (as opposed to the direct Gemini API) documentation.

### Microsoft Copilot Studio — content moderation (limited technical detail)

Per [Agent analytics (preview) - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/analytics-agent-evaluation-overview), Copilot Studio provides agent analytics including conversation counts and session data. However, specific mechanics of content filtering/moderation controls are not detailed in the verified sources checked. Prior L1 research found references to custom instructions for knowledge base configuration, but output validation specifics were not documented.

### ServiceNow Now Assist — tool-level parameter validation and agentic evaluations

Per [Lab 3: AI agent tool configuration debugging - ServiceNow Knowledge 2026](https://servicenow-events-or-lab-guidebo.gitbook.io/knowledge-2026/knowledge-2026/ccl6230-k26/lab-3-ai-agent-tool-configuration-debugging), ServiceNow emphasizes tool-level input validation as a layer of defense: "Don't trust the LLM to always pass the right input. Don't trust it to interpret raw, unstructured output. Build smart tools — tools that validate their own inputs, handle edge cases at the platform level, return clean structured output, and tell the agent exactly what to expect."

Per [Agentic Evaluations FAQ - ServiceNow Community](https://www.servicenow.com/community/now-assist-articles/agentic-evaluations-faq/ta-p/3429085), Agentic Evaluations is an LLM-based judge that evaluates agent performance across three metrics: Overall Task Completeness, Tool Calling Correctness, and Tool Choice Accuracy. This is an evaluation/observability layer (L5), not a real-time constraint enforcement layer, but it reflects that "Tool Calling Correctness" includes validation that "tool parameters are accurate, properly formatted, and complete."

---

## 2. Input validation and injection defense

### Direct prompt injection: user input filtering

AWS Bedrock Guardrails and Azure Foundry Guardrails both offer content filters and prompt-attack detection on the "User input" intervention point. Per the AWS documentation, the "Prompt Attack" filter detects jailbreak attempts in user input.

Per [Prompt Injection Defense at the AI Gateway - TrueFoundry](https://www.truefoundry.com/blog/prompt-injection-defense-llm-gateway), direct injection occurs when a user types malicious instructions into the chat. This is distinguished from indirect injection, which is embedded in data the agent retrieves or tool output it reads.

### Indirect prompt injection via tool output — the critical input validation attack surface

Per [Defense Against Indirect Prompt Injection via Tool Result Parsing](https://arxiv.org/pdf/2601.04795) (Harbin Institute of Technology, 2026), the most dangerous vector for agent compromise is indirect prompt injection: "By embedding adversarial instructions into the results of tool calls, attackers can hijack the agent's decision making process to execute unauthorized actions." The paper notes that as "LLM agents transition from digital assistants to physical controllers in autonomous systems and robotics, they face an escalating threat from indirect prompt injection."

**Defense pattern**: Per [patterns/tool-output-poisoning.md - Agent Patterns Catalog](https://github.com/agentpatternscatalog/patterns/blob/main/patterns/tool-output-poisoning.md), the recommended defense is to:
1. Wrap every tool result in a typed envelope with a trust label (`trust: low|medium|high`)
2. Apply instruction-stripping on low-trust output
3. Forbid tool-output-driven follow-up tool calls without re-validating against the user's original intent
4. Pair with input/output guardrails

Per [AI Agent Tool Poisoning & Prompt Injection Attacks: Production Defense Strategies 2026 - Md Sanwar Hossain](https://mdsanwarhossain.me/blog-ai-agent-tool-poisoning.html), the attack surface of autonomous agents includes "Web pages retrieved by a browsing tool containing hidden SYSTEM blocks, database rows where a customer name field contains SQL injection, PDF documents with white-on-white text injecting override commands, API responses from third-party services embedding adversarial JSON string values."

**Detection and blocking implementation**: Per [safehere GitHub](https://github.com/Expl0dingCat/safehere), an open-source tool for Cohere agents, tool-output scanning layers include pattern matching (regex rules for known injection signatures, unicode normalization, base64/hex decoding), schema drift detection (unexpected fields, structural changes), and semantic analysis. The tool distinguishes between blocking (replacing tool output with safe message) and quarantining (raising exception).

**Platform-level support**: AWS Bedrock Guardrails includes an "Indirect attacks" control (per the risk categories table). Azure Foundry Guardrails similarly lists "Indirect attacks (tool-output injection)" as a configurable control. However, both platforms apply these as OUTPUT-side checks after the tool result is returned to the agent, not necessarily with the per-tool trust-labeling pattern described in research.

**Not found in verified sources**: Specific documentation from any platform on mandatory input sanitization for user queries (beyond content filtering) or on per-tool trust labels before agent processing.

---

## 3. Constraint enforcement mechanisms — hard limits vs. soft guidance

### Hard vs. soft constraints: research literature on infrastructure-level enforcement

Per [SARC: A Governance-by-Architecture Framework for Agentic AI Systems](https://arxiv.org/html/2605.07728) (Besanson, arXiv 2026), SARC formalizes the distinction between hard and soft constraints:

- **Hard constraints**: Enforced at infrastructure/execution level; cannot be overridden by the model. Violation results in action rejection, blocking, or rollback.
- **Soft constraints**: Expressed as guidance to the model (system prompts, instructions); can in principle be argued around or bypassed.

SARC proposes four enforcement sites in the agent loop:
1. Pre-Action Gate — blocks action before execution
2. Action-Time Monitor — checks during execution
3. Post-Action Auditor — checks after execution
4. Escalation Router — routes violations to human review

The paper states: "finite reward penalties do not generally substitute for hard runtime constraints" — a crucial distinction for safety-critical systems like factory equipment monitoring.

Per [BlueGate: A Constraint-Native Execution Fabric](https://doi.org/10.5281/zenodo.19952208) (Zenodo 2026), BlueGate's abstract distinguishes:

- **Hard constraints**: "trigger immediate state rollback and rejection" (verbatim from the abstract). The equipment-shutdown/supervisor-approval framing below is this document's own illustrative example for the factory-agent context, not a quote from the paper — the abstract and description did not surface a matching example, and the full PDF was not fetched to check.
- **Soft constraints**: "route actions through a sequential least squares programming (SLSQP) optimizer to find the minimally altered, feasible representation" (verbatim from the abstract) of the original intent — e.g., illustratively, preferring lower power usage without failing the action outright if unavoidable (again, this document's own illustration, not a paper quote).

BlueGate's abstract also confirms "a proportional-integral (PI) adaptive controller that tunes constraint sensitivity in real-time based on observed violation rates."

Per [AgentSpec: Customizable Runtime Enforcement for Safe and Reliable LLM Agents](https://arxiv.org/pdf/2503.18666) (Wang et al., arXiv 2025), AgentSpec is a domain-specific language for specifying and enforcing runtime constraints with triggers, predicates, and enforcement mechanisms. The paper demonstrates "over 90% prevention of unsafe executions in code agent cases, 100% compliance by autonomous vehicles (AVs)" while remaining computationally lightweight (millisecond overheads).

Per [Agent-C: Enforcing Temporal Constraints for LLM Agents](https://arxiv.org/pdf/2512.23738) (Kamath et al., arXiv 2025), temporal constraints (e.g., "authenticate before accessing data") can be expressed in formal logic and enforced via SMT solving to ensure "every action generated by the LLM complies with the specification." The paper reports "100% conformance" on retail customer service and airline reservation tasks.

**Platform documentation of hard vs. soft**: None of the vendor platforms (AWS, Azure, Google, Microsoft, ServiceNow) explicitly document this distinction in their agent-specific documentation. AWS Bedrock Guardrails and Azure Foundry Guardrails operate as filters that block or mask output (hard), but system prompts and agent instructions are soft. No vendor platform documents infrastructure-level constraint enforcement separate from output filtering.

### Approval gates as a form of constraint

Per [Tool Approval - Convex](https://docs.convex.dev/agents/tool-approval.md) and [Tool Approvals - Vercel AI SDK](https://ai-sdk.dev/v7/docs/agents/tool-approvals), approval gates (requiring human confirmation before tool execution) are a documented pattern across multiple frameworks. These are hard runtime constraints: the tool is not executed until approval is given.

Per L2 research, AWS Bedrock supports `x-requireConfirmation` in OpenAPI schemas; Google Vertex AI ADK supports `require_confirmation` callable.

---

## 4. Failure/error recovery mechanics

### AWS prescriptive guidance on automated recovery

Per [AGENTOPS07-BP01 Implement automated response and recovery mechanisms - AWS Well-Architected](https://docs.aws.amazon.com/wellarchitected/latest/agentic-ai-lens/agentops07-bp01.html), AWS prescribes:

**Automatic cutoffs for every external dependency**: Store cutoff state (healthy, degraded, open) in a fast data store. Thresholds include error rate threshold (e.g., 50% errors in 60 seconds), timeout threshold (e.g., 5 consecutive timeouts), recovery probe interval (e.g., attempt recovery every 30 seconds).

**Fallback strategies per capability**:
- Tool failures: primary tool → alternative tool with equivalent capability → graceful degradation → manual escalation
- LLM inference failures: model fallback chains (e.g., Claude 3.5 Sonnet to Claude 3 Haiku)
- Multi-agent coordination failures: single-agent fallback mode

**Notification on degradation**: "Each fallback should notify users when quality is degraded, not silently return a worse answer."

**Durable orchestration**: Use AWS Step Functions or equivalent for recovery workflows with built-in error handling, retry logic, and compensating transactions.

### Error classification and retryability

Per [LLM Tool Calling Error Handling: Retries and Fallbacks - n8n Blog](https://blog.n8n.io/llm-tool-calling-error-handling/), production tool-call errors fall into four categories:

1. **Transport and network failures** (dropped TCP, DNS timeout, 503) — transient, external to application logic. Handle via silent infrastructure-level retries.
2. **External service errors** (API rejects request, 400 Bad Request) — upstream operational constraints. May or may not be retryable depending on error code.
3. **Invalid output** (tool returns malformed/unexpected data) — application-level problem. May require fallback or escalation.
4. **Policy violations** (tool succeeded but output violates constraint) — semantic problem. Requires agent reasoning or escalation.

**Classification before retry**: Check `is_retryable` predicate before entering retry loop. A 400 Bad Request is generally not retryable; a 503 is.

### Retry patterns and exponential backoff

Per multiple sources (AWS, n8n, ClaudePedia), the standard retry pattern is exponential backoff with jitter:
- Formula: `delay = min(base_delay * (2 ** attempt_number), max_delay) + random(0, jitter)`
- Prevents retry storms on overloaded services

**Retry loop bounds**: All sources emphasize maximum retry count limits (no infinite retries) and maximum total timeout to prevent indefinite blocking.

### Fallback strategy and graceful degradation

Per [Fallback-Recovery Agent Pattern - Agent Patterns](https://www.agentpatterns.tech/en/agent-patterns/fallback-recovery-agent), fallback is "a different implementation of the same capability" — smaller model, slower API, cached result, heuristic approximation — that produces "something useful, but not as good as the primary path."

The escalation ladder (in order):
1. **Retry** (cost: latency) — same operation, wait and retry
2. **Fallback** (cost: reduced quality) — alternative implementation
3. **Degrade** (cost: lost capability) — remove the capability from this session
4. **Fail** (cost: lost task) — stop entirely

**Not found in verified platform-specific documentation**: AWS, Azure, Google, Microsoft, ServiceNow do not expose detailed recovery mechanics for agent-specific scenarios in their primary documentation. AWS prescriptive guidance exists, but specific agent framework integration patterns are not detailed.

---

## 5. Constraint/validation changes specific to crisis or safety-critical scenarios

### Direct search for crisis-mode validation patterns

Queries searched specifically for: "AWS Bedrock crisis mode validation," "Azure Foundry emergency constraint," "crisis classification agent safety," "incident-triggered validation," and "safety-critical validation tightening."

**Result**: No vendor platform (AWS Bedrock, Azure AI Foundry, Google Vertex AI, Microsoft Copilot Studio, ServiceNow) documents a pattern where agent constraint/validation configuration changes based on runtime classification of a conversation or incident as a crisis/emergency/safety-critical scenario.

### Consistency with prior layers' findings

This finding continues the pattern identified in L1, L3, L4, and L5:
- **L1 (Information Boundary)**: "No official documentation was found from Azure AI Foundry, AWS Bedrock, Google Vertex AI, Microsoft Copilot Studio, ServiceNow, PagerDuty, or the industrial platforms (Siemens, Honeywell, GE) describing a documented pattern where the *agent's information boundary* (what data it retrieves, what tools it can call, what system instructions apply) changes based on classification of a conversation as a crisis/emergency."
- **L3 (Execution Orchestration)**: "No platform documented a pattern where agent execution flow changes based on crisis/incident classification."
- **L4 (Memory/State)**: "No platform publicly documents crisis-mode memory changes. Consistent with L1 and L3 findings."
- **L5 (Evaluation/Observability)**: "No platform documents evaluation criteria, thresholds, or SLA requirements changing based on runtime detection of crisis/emergency."

**L6 extension**: No platform documents that constraint enforcement, validation rules, approval-gate requirements, or error recovery mechanisms change based on safety-critical/crisis/emergency classification.

### Possible explanations

1. **Same-agent model**: Platforms enforce the philosophy that the same agent handles both routine and crisis queries with identical constraint/validation configuration, relying on system instructions and the query itself to guide behavior.
2. **Undocumented proprietary pattern**: Some platforms may implement crisis-mode validation privately, but this is not reflected in public technical documentation.
3. **Orchestration-layer distinction**: Crisis handling is addressed at the orchestration layer (L3) through escalation, approval gates, and human routing, not by tightening the validation layer itself.

---

## 6. Rollback / undo / compensating-action mechanics

### AWS Step Functions and the saga pattern

Per [Implement the serverless saga pattern by using AWS Step Functions - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/patterns/implement-the-serverless-saga-pattern-by-using-aws-step-functions.html), the saga pattern coordinates distributed transactions across microservices and defines compensating transactions to undo previous steps on failure.

**Pattern**: Each forward step (e.g., "Charge Payment") has a corresponding compensating step (e.g., "Refund Payment") defined in a Catch block. If step N fails, the saga executes compensating transactions for steps N-1, N-2, down to step 1, in reverse order, achieving eventual consistency.

**Key properties**:
- Durable state machine (stored in AWS Step Functions) persists across retries and failures
- Idempotent steps (each step produces the same result when re-run with the same input)
- Explicit compensation logic (defined per step, not auto-generated)

**Example (travel reservation saga)**: ReserveFlight → ReserveCarRental → ProcessPayment. On payment failure, Step Functions executes RefundPayment, then ReleaseCarRental, then CancelFlight, restoring consistent state.

**Not found in verified vendor documentation**: AWS, Azure, Google, Microsoft, and ServiceNow do not document compensating-transaction patterns specifically for agent-orchestrated workflows. Step Functions is a general orchestration tool, not an agent-specific feature. No vendor platform documents automatic or semi-automatic generation of compensating actions from agent action logs.

### ServiceNow and approval/supervised execution

Per L2 and L5 research, ServiceNow's Supervised Execution mode (requiring human approval before tool execution) is technically a gating mechanism, not a rollback mechanism. Once a tool executes, there is no documented automated undo.

### Manual escalation and human resolution

Across all platforms, when an agent action turns out to be wrong, the documented pattern is escalation to a human (via "Escalate" system topic, `vaSystem.connectToAgent()`, or approval-gate rejection). The human then manually reverses or mitigates the damage. No platform documents automated rollback of already-executed agent actions.

**Research literature on compensating actions**: [Agent-C: Enforcing Temporal Constraints for LLM Agents](https://arxiv.org/pdf/2512.23738) mentions that temporal constraints can prevent invalid action sequences, but does not address undoing already-executed actions.

---

## Comparison: constraint/validation/recovery mechanisms across platforms

| Platform | Output validation | Input validation | Hard vs. soft constraints | Error recovery | Crisis-mode changes | Rollback/undo |
|---|---|---|---|---|---|---|
| **AWS Bedrock Guardrails** | Six safeguards (content, denied topics, word, PII, grounding, reasoning); multi-intervention points (input, tool call, tool response, output); configurable severity thresholds | Content and prompt-attack filters on user input; Indirect attacks control for tool output (post-hoc); ApplyGuardrail API available | Not explicitly documented; guardrails act as hard output filters (blocking/masking); system prompts are soft guidance | Per AWS prescriptive guidance: exponential backoff retry, fallback strategies, circuit breakers, max retries/timeouts; not automated in guardrails themselves | Not documented | AWS Step Functions saga pattern for compensating transactions (general orchestration, not agent-specific) |
| **Azure AI Foundry Guardrails** | Controls per risk category; four intervention points (input, tool call, tool response, output); configurable severity thresholds (medium default); Groundedness control for hallucination detection in RAG | Content filtering on user input; Indirect attacks control for tool output detection | Not explicitly documented; guardrails block/mask (hard); system messages and agent instructions are soft | Not documented in verified sources; Azure broader platform has general error handling but agent-specific patterns unclear | Not documented | Not documented |
| **Google Gemini / Vertex AI Safety Filters** | Configurable harm-category filters with blocking thresholds (BLOCK_LOW_AND_ABOVE, BLOCK_MEDIUM_AND_ABOVE, BLOCK_ONLY_HIGH); default threshold is **Off** for Gemini 2.5/3 models (no filtering unless explicitly configured); non-configurable CSAM/child-safety filters always on | Non-configurable CSAM/child-safety filters; configurable harm filters applied to prompts and responses once thresholds are set | Not explicitly documented; filters block by stopping generation (hard) once configured; system instructions are soft | Not documented in verified sources | Not documented | Not documented |
| **Microsoft Copilot Studio** | Content moderation referenced; technical details not found in verified sources | Not documented in verified sources | Not documented | Not documented | Not documented | Not documented |
| **ServiceNow Now Assist** | Agentic Evaluations (L5, not real-time L6); tool-level parameter validation emphasized as platform pattern | Tool-level validation: builders expected to validate inputs and handle edge cases | Not explicitly documented; approval gates (Supervised Execution mode) are hard gating; tool descriptions are soft guidance | Not documented as automated; manual escalation to human is prescribed pattern | Not documented | Not documented; escalation to human is recovery pattern |
| **Research/Academia** | — | Tool output trust labeling, instruction-stripping, re-validation; SARC pre-action gates, post-action auditors | SARC, BlueGate, AgentSpec, Agent-C: hard (infrastructure-enforced, cannot be overridden) vs. soft (prompt/instruction guidance); formal logic enforcement possible | Escalation ladder (retry → fallback → degrade → fail); explicit bounds required | Not studied in reviewed literature | SARC audit and projection-based governance; Agent-C temporal constraints prevent sequences; saga pattern for microservices |

---

## Key gaps: areas where primary-source material was not found

- **Microsoft Copilot Studio constraint/validation/recovery details** — agent analytics and evaluation are referenced, but specific guardrail mechanics, input validation, error recovery strategies, and rollback capabilities were not found in technical documentation checked.
- **Industrial platforms (Siemens, Honeywell, GE, Schneider Electric)** — only marketing material was available; no technical documentation on constraint enforcement or validation layer mechanics for their industrial agent products.
- **Vendor platform documentation on hard vs. soft constraint distinction** — while AWS, Azure, Google, and Microsoft provide guardrails and filters, they do not explicitly document the distinction between infrastructure-enforced hard constraints and prompt/instruction-based soft guidance. This distinction is well-established in academic literature (SARC, BlueGate) but not in vendor documentation.
- **Crisis/emergency-triggered validation tightening** — no platform documents this pattern, continuing the absence found in L1, L3, L4, and L5.
- **Automatic rollback/compensating-action generation from agent logs** — AWS Step Functions supports saga pattern for orchestrated workflows, but no vendor documents automatic detection and reversal of incorrect agent actions post-execution.
- **Per-tool trust labeling and instruction-stripping mechanics** — research literature describes this pattern (tool output poisoning defense), but vendor platforms do not document native support; builders must implement custom logic.
- **ServiceNow crisis-mode validation specifics** — no documentation found on whether ServiceNow's Supervised Execution mode or approval gates change their behavior for classified crisis/safety-critical incidents.

---

## Observations

1. **Output validation is documented and multi-layered across major platforms, but default-on vs. default-off varies significantly.** AWS Bedrock Guardrails, Azure Foundry Guardrails, and Google Gemini/Vertex AI all provide content filtering, PII redaction, and hallucination detection, operating at multiple intervention points (input, tool call, tool response, output) and configurable per severity threshold. But the defaults point in different directions: Azure defaults configurable categories to Medium severity (filtering active out of the box); AWS Bedrock Guardrails require explicit creation and application (nothing filtered until a builder builds a guardrail); and Google's configurable content filters default to **Off** for current Gemini 2.5/3 models — meaning a builder who assumes "the platform filters by default" would be wrong specifically for Google, and would need to explicitly set thresholds to get any content-filter blocking at all. Only each platform's small set of non-configurable, always-on filters (CSAM/child-safety material) is a safe assumption across all three.

2. **Input validation for tool outputs is asymmetrically documented.** Vendor platforms (AWS, Azure, Google) include controls for detecting indirect prompt injection in tool responses as part of their guardrail suites, but the implementation is a post-hoc output filter, not a pre-validation envelope with per-tool trust labels. Research literature describes a more robust pattern (tool-output poisoning defense with trust labeling), but vendor platforms do not expose this as a built-in, configurable mechanic. Builders must implement custom input validation for low-trust tool output.

3. **Hard vs. soft constraint distinction is absent from vendor agent documentation but central to governance research.** Academic work (SARC, BlueGate, AgentSpec, Agent-C) rigorously formalizes hard constraints (infrastructure-enforced, cannot be overridden) vs. soft constraints (prompt/instruction guidance). Vendor platforms enforce hard constraints via output filtering/blocking, but do not document architectural separation or allow builders to specify which constraints are hard vs. soft. System prompts and agent instructions are implicitly soft.

4. **Error recovery is prescribed but not platform-automated for agents.** AWS prescriptive guidance describes exponential backoff, fallback strategies, circuit breakers, and maximum retry bounds. AWS Step Functions supports saga pattern for compensating transactions in orchestrated workflows. However, for individual agent-orchestrated tasks, no vendor platform documents automatic retry, fallback, or recovery logic internal to the agent. Builders must implement custom retry/fallback logic or use orchestration tools like Step Functions.

5. **Crisis/emergency-triggered validation changes are not documented for any platform.** This is consistent with L1, L3, L4, and L5 findings — no layer of the six-layer taxonomy shows a platform-documented "crisis mode" switch. No platform describes tightening validation rules, mandatory approval gates, mandatory human review, or changed constraint enforcement based on runtime classification of a conversation/incident as safety-critical or emergency. This suggests either: (a) constraint enforcement is configuration-static, unchanging regardless of conversation context, or (b) crisis/emergency handling is addressed at orchestration layers (L3), not validation layers (L6).

6. **Rollback and compensating-action mechanisms are orchestration-level, not validation-level.** AWS Step Functions supports saga pattern (compensating transactions), but this is a general-purpose orchestration primitive, not an agent-specific feature. No platform documents mechanisms for reversing or mitigating incorrect agent actions that have already executed. Manual escalation to a human is the documented recovery pattern.

7. **ServiceNow emphasizes tool-level validation over agent-level guardrails.** Unlike AWS and Azure, ServiceNow's constraint enforcement is framed as "build smart tools" — validating inputs, handling edge cases, returning structured output — rather than applying platform-wide guardrails. This is consistent with L2 findings on tool system mechanics but represents a philosophical difference: ServiceNow places the burden on tool builders, while AWS/Azure provide platform-native guardrails.

8. **No vendor platform documents validation changes triggered by agent classification or contextual safety metrics.** Unlike human safety systems that might tighten review procedures for high-risk actions, no agent platform adjusts its validation rules based on agent-detected risk levels or external crisis signals.

---

## Primary sources referenced

### Output Validation & Guardrails — AWS Bedrock

- [Detect and filter harmful content by using Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html) — six safeguard types, configurable severity, multiple intervention points
- [Use the ApplyGuardrail API in your application - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-use-independent-api.html) — ApplyGuardrail API, deployment flexibility, independent assessment
- [Prevent factual errors from LLM hallucinations with mathematically sound Automated Reasoning checks (preview) - AWS Blog](https://aws.amazon.com/blogs/aws/prevent-factual-errors-from-llm-hallucinations-with-mathematically-sound-automated-reasoning-checks-preview/) — "first and only generative AI safeguard" marketing claim (preview announcement)
- [Minimize AI hallucinations and deliver up to 99% verification accuracy with Automated Reasoning checks: Now available - AWS Blog](https://aws.amazon.com/blogs/aws/minimize-ai-hallucinations-and-deliver-up-to-99-verification-accuracy-with-automated-reasoning-checks-now-available/) — "up to 99% verification accuracy" marketing claim (GA announcement)

### Output Validation & Guardrails — Azure AI Foundry

- [Guardrails and controls overview in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/guardrails/guardrails-overview) — control-based architecture, intervention points, risk categories
- [Default Guardrail policies for Azure OpenAI - Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/default-safety-policies) — default severity thresholds, configurability, coverage
- [Guardrails and controls overview in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/guardrails/guardrails-overview) — agent-specific vs. model-specific controls

### Output Validation — Google Vertex AI

- [Safety and content filters | Gemini Enterprise Agent Platform](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/configure-safety-filters) — configurable and non-configurable filters, blocking thresholds, feedback codes
- [Safety in Gemini Enterprise Agent Platform](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/safety-overview) — multi-layered safety approach, tool taxonomy, risk mitigation strategies

### Input Validation & Injection Defense

- [Defense Against Indirect Prompt Injection via Tool Result Parsing](https://arxiv.org/pdf/2601.04795) — indirect injection taxonomy, tool-output poisoning, defense via result parsing
- [patterns/tool-output-poisoning.md - Agent Patterns Catalog](https://github.com/agentpatternscatalog/patterns/blob/main/patterns/tool-output-poisoning.md) — trust labeling, instruction-stripping, re-validation pattern
- [AI Agent Tool Poisoning & Prompt Injection Attacks: Production Defense Strategies 2026](https://mdsanwarhossain.me/blog-ai-agent-tool-poisoning.html) — attack surface taxonomy, real-world attack scenarios, defense architecture
- [Prompt Injection Defense at the AI Gateway - TrueFoundry](https://www.truefoundry.com/blog/prompt-injection-defense-llm-gateway) — direct, indirect, and tool-mediated injection; defense layering at gateway

### Hard vs. Soft Constraints — Research Literature

- [SARC: A Governance-by-Architecture Framework for Agentic AI Systems](https://arxiv.org/html/2605.07728) — hard vs. soft constraint formalization, enforcement sites (Pre-Action Gate, Action-Time Monitor, Post-Action Auditor, Escalation Router)
- [BlueGate: A Constraint-Native Execution Fabric for Adaptive Control of Autonomous Systems](https://doi.org/10.5281/zenodo.19952208) — hard constraint (rollback/rejection) vs. soft constraint (optimization-based projection), adaptive PI controller
- [AgentSpec: Customizable Runtime Enforcement for Safe and Reliable LLM Agents](https://arxiv.org/pdf/2503.18666) — domain-specific language for runtime constraint enforcement, multi-domain evaluation, >90% safety in code agents
- [Enforcing Temporal Constraints for LLM Agents - Agent-C](https://arxiv.org/pdf/2512.23738) — formal temporal properties (e.g., authenticate before access), SMT-based enforcement, 100% conformance reported

### Error Recovery & Failure Handling

- [AGENTOPS07-BP01 Implement automated response and recovery mechanisms - AWS Well-Architected](https://docs.aws.amazon.com/wellarchitected/latest/agentic-ai-lens/agentops07-bp01.html) — automatic cutoffs, fallback strategies, circuit breakers, recovery time objectives
- [LLM Tool Calling Error Handling: Retries and Fallbacks - n8n Blog](https://blog.n8n.io/llm-tool-calling-error-handling/) — error classification (transport, external service, invalid output, policy violation), retryability predicate
- [Fallback-Recovery Agent Pattern - Agent Patterns](https://www.agentpatterns.tech/en/agent-patterns/fallback-recovery-agent) — escalation ladder (retry → fallback → degrade → fail), bounded recovery, checkpoint-based resumption
- [Error Recovery and Resilience - ClaudePedia](https://claudepedia.dev/docs/error-recovery) — escalation ladder formalization, retryability classification, query-source partitioning

### Rollback & Compensating Transactions

- [Implement the serverless saga pattern by using AWS Step Functions - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/patterns/implement-the-serverless-saga-pattern-by-using-aws-step-functions.html) — saga orchestration, compensating transactions, eventual consistency
- [AWS Step Functions: Saga Pattern Implementation](https://awsforengineers.com/blog/aws-step-functions-saga-pattern-implementation/) — saga steps, compensating transactions, idempotence, monitoring

### ServiceNow Tool Validation & Evaluation

- [Lab 3: AI agent tool configuration debugging - ServiceNow Knowledge 2026](https://servicenow-events-or-lab-guidebo.gitbook.io/knowledge-2026/knowledge-2026/ccl6230-k26/lab-3-ai-agent-tool-configuration-debugging) — smart tools, input validation, edge case handling, structured output
- [Agentic Evaluations FAQ - ServiceNow Community](https://www.servicenow.com/community/now-assist-articles/agentic-evaluations-faq/ta-p/3429085) — Tool Calling Correctness metric, parameter validation, tool choice accuracy
- [A Field Guide to Evaluating, Analyzing, and Debugging AI Agents - ServiceNow Community](https://www.servicenow.com/community/ceg-ai-coe-articles/a-field-guide-to-evaluating-analyzing-and-debugging-ai-agents-on/ta-p/3545229) — three evaluation disciplines (evaluations, analytics, debugging), validation as quality gate

### Tool Output Safety

- [safehere GitHub](https://github.com/Expl0dingCat/safehere) — runtime tool-output scanning, six detection layers, trust-based handling, polytot encoding defenses

### Related Foundation Layers

- [Factory Equipment Monitoring Chat Agent: The Information Boundary Layer (L1)](./factory-agent-harness-l1-information-boundary.md) — L1 findings on absence of crisis-mode information boundary changes
- [Factory Equipment Monitoring Chat Agent: The Tool System Layer (L2)](./factory-agent-harness-l2-tool-system.md) — L2 tool definition, access control, parameter schema
- [Factory Equipment Monitoring Chat Agent: The Execution Orchestration Layer (L3)](./factory-agent-harness-l3-execution-orchestration.md) — L3 findings on absence of crisis-mode orchestration changes
- [Factory Equipment Monitoring Chat Agent: The Memory/State Layer (L4)](./factory-agent-harness-l4-memory-state.md) — L4 findings on absence of crisis-mode memory changes
- [Factory Equipment Monitoring Chat Agent: The Evaluation/Observability Layer (L5)](./factory-agent-harness-l5-evaluation-observability.md) — L5 evaluation metrics, human feedback loops, observability

