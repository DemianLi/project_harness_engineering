# Factory Equipment Monitoring Chat Agent: The Execution Orchestration Layer (L3)

This document investigates execution orchestration mechanics for end-user-facing factory equipment monitoring and crisis-handling chat agents. The orchestration layer determines how multi-step conversations and tasks are decomposed, sequenced, and executed: whether as fixed sequences, LLM-driven plans, reactive step-by-step loops, or hybrid patterns. This layer sits above the information boundary (L1) and tool system (L2), and coordinates their use across a conversation.

Scope: multi-turn dialog/task orchestration on enterprise platforms; slot-filling and clarifying-question orchestration for diagnostic conversations; human-handoff mechanics and what state gets preserved across the boundary; plan-then-verify orchestration patterns for non-coding agents; and crisis/incident-mode execution changes.

---

## Search tooling notes

This pass used `anysearch` to discover primary-source URLs, then targeted `WebFetch` of official documentation for AWS Bedrock Agents, Azure AI Foundry, Google Vertex AI ADK, Microsoft Copilot Studio, ServiceNow, and PagerDuty. Primary sources include:

- **AWS Bedrock Agents**: ReAct (default), ReWoo (custom), custom orchestration via Lambda, multi-agent collaboration, multi-turn conversation in flows
- **Azure AI Foundry**: Workflows (visual + YAML-based), orchestration patterns (Sequential, Human-in-the-loop, Group chat)
- **Google Vertex AI ADK**: Graph-based workflows, Dynamic workflows, Collaborative workflows, Template workflows, multi-agent patterns
- **Microsoft Copilot Studio**: Generative orchestration (LLM-driven planning), Classic orchestration, Topics with trigger-based vs. description-based selection, entity slot filling
- **ServiceNow Virtual Agent**: Multi-turn conversation, escalation to live agent via `vaSystem.connectToAgent()`, context passing through interaction records
- **PagerDuty**: Incident Workflows (automated response, payload extraction, conditional orchestration)
- **Azure CLU (Conversational Language Understanding)**: Entity slot filling with progressive disclosure
- **Google Dialogflow ES**: Actions and parameters with slot filling for required parameters
- **Reactive Agents Framework**: ReAct vs. Blueprint (plan-then-execute) patterns
- **CXAS SCRAPI**: Slot Filling DAG Framework

No official documentation from Siemens, Honeywell, GE, or other industrial platforms on orchestration mechanics was located in public sources. Crisis-mode orchestration specifics (documented changes to agent behavior or tool availability during emergencies) were not found in verified sources across any platform checked. All AWS Bedrock Agents documentation cited (§1 orchestration, §1 multi-agent collaboration, §4 ReWoo) carries a live notice as of this writing that the service is now called "Bedrock Agents Classic" and closes to new customers on 2026-07-30 in favor of Bedrock AgentCore — see the timeliness note in §1 for the exact wording.

---

## 1. Multi-turn dialog/task orchestration mechanics on enterprise platforms

### AWS Bedrock Agents: ReAct, ReWoo, and custom orchestration

**Timeliness note (checked as of this writing, 2026-07-11):** [How Amazon Bedrock Agents works](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-how.html) currently carries a live notice that the product line this whole subsection describes is being sunset: "Amazon Bedrock Agents (launched November 2023) is now 'Amazon Bedrock Agents Classic' and will no longer be open to new customers starting on July 30, 2026. For capabilities similar to Bedrock Agents Classic, explore Amazon Bedrock AgentCore." That cutoff is under three weeks away at the time of this research. Existing customers can keep using it, but any new build should evaluate Bedrock AgentCore instead — the orchestration mechanics below describe the (Classic) service being replaced, not necessarily AgentCore's architecture, which was not researched in this pass.

That page describes, without using the term "ReAct" by name, the same rationale → action → observation loop: the agent generates a rationale, predicts an action or knowledge-base query, invokes it, and produces an "observation" that re-enters the loop until a final response is ready. Per [Getting started with Amazon Bedrock Agents custom orchestrator - AWS ML Blog](https://aws.amazon.com/blogs/machine-learning/getting-started-with-amazon-bedrock-agents-custom-orchestrator/), this loop is explicitly named there: "Using the default orchestration strategy, reasoning and action (ReAct), users can quickly build and deploy agentic solutions." The same post describes ReWoo (Reasoning without Observation) as a contrasting strategy: "The ReWoo technique optimizes performance by generating a complete task plan up front and executing it without checking intermediate outputs." Both are available as orchestration strategies, alongside fully custom orchestration.

Per [Customize your Amazon Bedrock Agent's behavior with custom orchestration - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-custom-orchestration.html), custom orchestration is "implemented by users as an AWS Lambda function" that "outlines your orchestration logic. The function controls how the agent responds to input by providing instructions to the Bedrock's runtime process on when and how to invoke the model, when to invoke actions tools, and then determining the final response." This allows builders to define orchestration strategies specific to their use case, distinct from the default ReAct or ReWoo approaches.

Per [Use multi-agent collaboration with Amazon Bedrock Agents - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-multi-agent-collaboration.html), multi-agent collaboration uses a supervisor-and-collaborator agent pattern: "you can quickly designate an Amazon Bedrock Agent as the supervisor and then associate one or more collaborator agents with the supervisor... When you invoke the supervisor agent, it automatically creates and executes a plan across a set of collaborator agents and routes relevant requests and tasks to the appropriate collaborator agent." Per [Create multi-agent collaboration - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/create-multi-agent-collaboration.html), a maximum of 10 collaborator agents can be associated with one supervisor; the supervisor can be configured as "Supervisor" (coordinating responses) or "Supervisor with routing" (routing to one collaborator for final response, reducing latency).

### Azure AI Foundry: declarative workflows (visual + YAML)

Per [Build a workflow in Microsoft Foundry - Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/workflow), workflows in Foundry are "declarative, predefined sequences of actions that orchestrate agents and business logic in a visual builder." The platform provides three orchestration templates: Sequential (passes result from one agent to the next), Human in the loop (asks user a question and awaits input), and Group chat (dynamically passes control between agents based on context or rules). Per [Introducing Multi-Agent Workflows in Foundry Agent Service - Microsoft Foundry Blog](https://devblogs.microsoft.com/foundry/introducing-multi-agent-workflows-in-foundry-agent-service/), workflows support "visual and YAML-based orchestration" with "Variables and JSON schema output" to "capture agent output as local variables, reference them in later steps," and the two representations are synchronized: "Any change in one view updates the other."

### Google Vertex AI ADK: graph-based, dynamic, collaborative, and template workflows

Per [Workflows: multi-agent, multi-node applications - Google ADK](https://adk.dev/workflows/), ADK supports four workflow structures: (1) **Graph-based workflows** (ADK 2.0+) combining "AI-powered agents and deterministic execution nodes into a flexible execution graph that can include decision branching"; (2) **Dynamic workflows** (ADK 2.0+) using "full programmatic code logic"; (3) **Collaborative workflows** (ADK 2.0+) where "a single agent acts in a dynamic coordinator role to accomplish tasks with a set of specified sub-agents"; and (4) **Template workflows** providing "fixed execution logic structures including sequences, loops, and parallel execution."

Per [Developer's guide to multi-agent patterns in ADK - Google Developers Blog](https://goo.gle/3Ng8qVt), two concrete patterns documented are the Sequential Pipeline (assembly-line: Agent A → Agent B → Agent C) and the Coordinator/Dispatcher pattern (a central agent analyzes intent and routes to specialist sub-agents). ADK's SequentialAgent primitive handles orchestration; output from one agent is written to shared session state and picked up by the next.

### Microsoft Copilot Studio: generative vs. classic orchestration

Per [Orchestrate agent behavior with generative AI - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/advanced-generative-actions), Copilot Studio agents can use either "generative orchestration" (new, LLM-driven, default for newly created agents) or "classic orchestration" (traditional, trigger-phrase-driven). The documentation provides a comparison table:

- **Generative orchestration**: Agent selects topics/tools/agents based on their descriptions; automatically generates questions to prompt for missing inputs; can use multiple topics, tools, and knowledge sources in combination; generates response automatically.
- **Classic orchestration**: Agent selects topics based on matching trigger phrases to user query; requires explicit Question nodes to prompt for information; tries to select single topic (falls back to knowledge); requires explicit Message nodes for responses.

Per [Apply generative orchestration capabilities - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/generative-orchestration), the orchestrator (LLM-driven planner) "turns an input, like a user message or event, into a structured plan" by identifying intents and choosing "which tools, topics, or agents to invoke for each step." The orchestrator outputs "an ordered list of steps (a 'plan') that the runtime executes."

---

## 2. Slot-filling / clarifying-question orchestration for diagnostic conversations

Diagnostic conversations (e.g., equipment troubleshooting) require progressively collecting specific details from the user. Platforms differ in whether this collection is LLM-driven or deterministically orchestrated.

### Microsoft Copilot Studio: automatic clarifying questions in generative mode

Per the Copilot Studio generative orchestration documentation above, the system "can automatically generate questions to prompt users for any missing information required to fill inputs for topics and tools." This is distinct from classic mode, where Question nodes must be explicitly authored. The orchestrator infers from the topic/tool definitions what information is required and asks the user for missing pieces before attempting execution.

### Azure CLU: progressive disclosure with entity slot filling

Per [Conversational language understanding (CLU) entity slot filling in multi-turn conversations - Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/language-service/conversational-language-understanding/concepts/multi-turn-conversations), CLU "progressively extracts and organizes the details it needs as users provide them throughout these multi-turn conversations." Slots are "predefined containers that hold those entities." The system "begins by recognizing the user's intent, then extracts any available entities from their initial input." If required slots are unfilled, "the model is prompted to ask for the missing pieces." The documentation notes: "Instead of requiring users to provide every detail at the outset, CLU enables progressive disclosure where information emerges naturally as the conversation unfolds."

### Google Dialogflow ES: slot filling with required parameters and prompts

Per [Actions and parameters - Dialogflow ES](https://docs.cloud.google.com/dialogflow/es/docs/intents-actions-parameters), each parameter has a "Required" flag; if set, parameters that are not supplied trigger "slot filling with required parameters" where the agent asks prompts configured on the parameter until it is filled. The prompts are manually authored in the intent configuration. Per [Slot Filling - CXAS SCRAPI](https://googlecloudplatform.github.io/cxas-scrapi/stable/guides/slot-filling/), the "Slot Filling DAG Framework" moves "control flow out of the LLM and into deterministic Python": configuration declares slots (data to collect) and tasks (backend operations), and a state machine decides what action to take on each turn—ask the next question, fire a task, request confirmation, or escalate.

### AWS Bedrock Flows: multi-turn with agent nodes pausing for clarification

Per [Converse with an Amazon Bedrock flow - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/flows-multi-turn-invocation.html) (multi-turn conversation in preview), "When an agent node requires clarification or additional context, it can intelligently pause the flow's execution and prompt the user for specific information." The agent can "ask follow-up questions to gather the necessary details. Once the user provides the requested information, the flow seamlessly resumes execution with the enriched input." The flow tracks a unique execution ID across turns, allowing the runtime to maintain state across the multi-turn interaction.

---

## 3. Human-handoff orchestration as a state transition

When an agent escalates a conversation to a human, what state gets transferred and how is it preserved?

### ServiceNow Virtual Agent: context passing via interaction records

Per [Transferring Virtual Agent conversations to a live agent - ServiceNow](https://www.servicenow.com/docs/r/conversational-interfaces/virtual-agent/transfer-to-live-agent.html), transfer is triggered via `vaSystem.connectToAgent()` in a topic script. The documentation states: "The transfer process is seamless, ensuring that users continue their conversation in the Virtual Agent interface" (or Agent Workspace, depending on setup). 

**Community-sourced, not confirmed in official docs:** the official [Transferring Virtual Agent conversations to a live agent](https://www.servicenow.com/docs/r/conversational-interfaces/virtual-agent/transfer-to-live-agent.html) and [Virtual Agent interaction records](https://www.servicenow.com/docs/r/xanadu/conversational-interfaces/virtual-agent/va-interactions.html) pages reference related topics ("Configure context variables for storing chat-related information," "Live agent chat context variables") but their content was not accessible to confirm field-level detail, and neither page's accessible content mentions `context_table`, `context_document`, `vaVars`, or `interactionBlobRecord` by name. The specific mechanics — per [Solved: Can you pass Live Agent Variables (Contexts) from Virtual Agent to an Incident - ServiceNow Community](https://www.servicenow.com/community/virtual-agent-nlu-forum/can-you-pass-live-agent-variables-contexts-from-virtual-agent-to/m-p/224261), a ServiceNow Community forum thread rather than official documentation — describe the pattern as: (1) the agent sets LiveAgent_* variables during conversation (e.g., `vaVars.LiveAgent_short_description = vaInputs.user_input`); (2) on escalation, these are stored in an Interaction record with `context_table` and `context_document` fields; (3) the live agent (or downstream script) retrieves the context by querying the Interaction via `interactionBlobRecord.getValue('value')` and parses the JSON blob to access the collected values. Treat this as a community-documented pattern, not an officially guaranteed API.

### Microsoft Copilot Studio → ServiceNow handoff: DirectLine relay with conversation history

Per [Hand off to ServiceNow - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/customer-copilot-servicenow), Copilot Studio can escalate to a ServiceNow live agent via a DirectLine relay (an Azure Function acting as a bridge). The page's "Sample response from the Azure Function" shows an `activities` array with message history and a `handoff.initiate` event; abbreviated below to the fields relevant here (the full example on the page includes additional fields per activity, such as `timestamp`, `from.id`, `from.name`, `conversation.id`, and the event's `value` payload):

```json
{
  "activities": [
    {"type": "message", "id": "0000001", "text": "Hello, I'm a Copilot Studio Agent.", "from": {"role": "bot"}},
    {"type": "message", "id": "0000002", "text": "speak to a live agent", "from": {"role": "user"}},
    {"type": "event", "id": "0000006", "name": "handoff.initiate"}
  ],
  "watermark": "10"
}
```

This allows ServiceNow Bot Interconnect to receive the full conversation context before the live agent joins.

### PagerDuty: escalation policies and incident orchestration rules (not chat-specific)

Per [Incident Workflows - PagerDuty Knowledge Base](https://support.pagerduty.com/main/docs/incident-workflows), incident workflows automate "manual steps in your incident response process – paging subject matter experts, setting up a conference bridge, establishing a Slack channel, sending out status updates." Per [How Security Extracts Relevant Data for Immediate Incident Coordination - PagerDuty](https://www.pagerduty.com/ops-guides/using-pd/security-team/), workflows use "orchestration rules" to "extract the relevant payload details into an incident custom field" and conditional logic ("if/then") to route and notify. This is escalation and routing of *incidents*, not routing of chat conversations to humans; the mechanics differ from chat handoff.

---

## 4. Plan-then-verify orchestration patterns for non-coding agents

Is there a documented equivalent to the coding-agent "Generator/Evaluator with Sprint Contract" pattern (L2 research), specifically for customer-service or equipment-monitoring agents?

### AWS Bedrock ReWoo: planning without observation

Per the AWS Bedrock custom orchestrator blog referenced above, ReWoo (Reasoning without Observation) is documented as a contrast to ReAct: instead of thought-action-observation-thought-action-observation loops, the agent plans all steps upfront with a single LLM call, then executes them deterministically (e.g., via Lambda) without re-planning. The blog states this is better suited "to scenarios where speed and rapid sequential processing are paramount" and where "the action shape is known before any observation arrives."

### Blueprint and Reflexion patterns: plan-verify-execute-solve and generate-critique-improve

Per [Reasoning - Reactive Agents](https://docs.reactiveagents.dev/guides/reasoning/), the Blueprint strategy for agent reasoning uses a "ReWOO-style Plan → Verify → Execute → Solve" flow: (1) Plan (1 LLM call) emits the entire tool plan as a dependency graph with evidence references; (2) Verify (no LLM) validates the DAG and repairs fixable gaps; (3) Execute (no LLM) runs tools in dependency order, independent steps in parallel; (4) Solve (1 LLM call, skipped when a step already produced the deliverable). The same source documents Reflexion, a "Generate → Self-Critique → Improve loop" used for "quality-critical output — writing, analysis, summarization."

### Comparison: ReAct vs. plan-then-execute

Per [ReAct (Reason + Act): Interleaved Reasoning-Action Loops - AgentPatterns.ai](https://agentpatterns.ai/agent-design/react-pattern/), the comparison is:

| Pattern | Decides next action by | Recomputes plan after each observation |
|---------|------------------------|----------------------------------------|
| ReAct | Reasoning that re-conditions on each new observation | Yes — every step |
| Plan-then-execute | A single upfront plan, then deterministic execution | No — plan is fixed |

The site's own stated reasoning for when the pattern is worth its overhead: "The mechanism only pays back when the observation actually disambiguates the next thought. When every observation is predictable from prior state, the extra inference step costs without buying signal." It cites the original ReAct paper's "6% vs. 56%" hallucination comparison on HotpotQA (Yao et al., 2022, arXiv:2210.03629) as evidence — checked directly against the paper's Table 2, these are two different rows rather than a single like-for-like metric: ReAct's 6% is its false-positive rate (hallucinated reasoning within otherwise-successful answers), while chain-of-thought's 56% is its hallucination-driven *failure* rate; CoT's own false-positive rate is 14%. The direction of the finding (ReAct hallucinates markedly less than chain-of-thought on this task) holds either way, but the specific "6% vs. 56%" framing compares two different categories, not the same metric for both methods.

### Status of plan-verify patterns for non-coding agents

No platform surveyed (AWS Bedrock, Azure, Google Vertex, Copilot Studio, ServiceNow) documents a first-class equivalent to the "Evaluator agent verifying Generator output against a Sprint Contract" pattern for customer-service or equipment-monitoring agents. ReWoo (planning then executing deterministically) exists, but it is not framed with an explicit separate verification or evaluation step. The equivalent pattern would require the builder to implement a verification agent or verification step within their custom orchestration (e.g., AWS Lambda-based orchestrator or Azure workflow logic), but this is not documented as a recommended practice.

---

## 5. Crisis/incident-mode orchestration specifics

Does the *execution flow itself* change when a conversation is classified as an incident/emergency, distinct from the information boundary changes documented in L1?

### PagerDuty Incident Workflows: conditional routing based on incident properties

Per the Incident Workflows documentation above, conditional logic in workflows can "post different types of Slack channel messages based on the content" of extracted incident fields. The workflow can route escalation differently based on incident severity, service, or other properties. However, this is routing of *incidents to responders*, not routing of *agent behavior* — the agent's tool set, action sequence, or approval requirements do not change at runtime based on incident severity classification.

### ServiceNow: role masking and supervised execution (applied regardless of crisis)

Per [Latest access control security enhancements for AI Agents and Skill Kit - ServiceNow Community](https://www.servicenow.com/community/now-assist-articles/latest-access-control-security-enhancements-for-ai-agents-and/ta-p/3374036), role masking and supervised execution mode (which can require human approval before executing a tool) apply to tools; they are configured at tool definition time, not activated dynamically based on crisis classification.

### Crisis-mode orchestration: not found in verified sources

No official documentation was found from AWS Bedrock, Azure AI Foundry, Google Vertex AI, Microsoft Copilot Studio, ServiceNow, or PagerDuty describing a pattern where an agent's orchestration strategy, tool set, or action sequence changes based on runtime detection of a crisis or emergency. Escalation mechanisms exist (routing to different humans, requiring approval), but these operate at the orchestration layer (who responds), not at the execution layer (how the agent itself adapts). The absence across multiple platforms suggests either:
- Distinct crisis orchestration is not a documented best practice.
- Some platforms implement it, but documentation is not publicly available.
- Crisis-mode adaptation happens via changes to system prompt or tool visibility, rather than a separate orchestration strategy.

---

## Comparison: orchestration mechanisms across platforms

| Platform | Default orchestration | Multi-step sequencing | Slot-filling mechanism | Handoff state preservation | Plan-verify pattern |
|----------|------------------------|------------------------|------------------------|---------------------------|----------------------|
| **AWS Bedrock Agents** | ReAct (reactive per-step) | ReAct or ReWoo (custom Lambda) | Not documented in verified sources | Not documented | ReWoo (deterministic) |
| **Azure AI Foundry** | Workflows (declarative; sequential, human-in-loop, group chat templates) | Visual + YAML-based composition; pass output as variables between steps | Not documented; managed by topic/agent definitions | Not documented | Not documented |
| **Google Vertex AI ADK** | Graph-based, dynamic, collaborative, or template workflows | Explicit graph edges (acyclic or cycle-constrained); SequentialAgent primitive | Not documented | Not documented | Blueprint (plan-verify-execute-solve) available as reasoning strategy |
| **Microsoft Copilot Studio** | Generative (LLM-driven planning from topic/tool descriptions) or Classic (trigger phrase matching) | Topics and inline agents; generative mode auto-chains multiple capabilities | Generative mode auto-generates clarifying questions; no slot-filling framework exposed | Not documented | Not documented in Copilot Studio; described in general ADK docs |
| **ServiceNow Virtual Agent** | Not explicitly documented; implied topic-based routing | Topic branching (scripted) | Not documented as formal slot-filling | Context passed via `vaSystem.connectToAgent()` and Interaction records | Not documented |
| **PagerDuty Incident Workflows** | Not agent-based (incident routing/orchestration, not conversation orchestration) | Conditional routing, payload extraction, sequential notifications | N/A | N/A | N/A |

---

## Key gaps: areas where primary-source material was not found

- **AWS Bedrock Agents**: Slot-filling and automatic clarifying-question mechanics not documented in verified sources checked.
- **Azure AI Foundry**: Human-handoff orchestration mechanics and what state is preserved not documented; slot-filling via generative mode exists but specifics not detailed in sources checked.
- **Google Vertex AI ADK**: Slot-filling and clarifying-question orchestration not documented; handoff mechanics not documented.
- **Microsoft Copilot Studio**: Exact handoff state preservation mechanics not documented; plan-verify-execute patterns not documented.
- **ServiceNow Virtual Agent**: Slot-filling framework not documented; orchestration strategy (reactive, planned, or other) not documented in sources checked.
- **Crisis/incident-mode orchestration**: No platform documented a pattern where agent execution flow changes based on crisis/emergency classification.
- **Industrial platforms (Siemens, Honeywell, GE, Schneider Electric)**: No public technical documentation on orchestration mechanics located.

---

## Observations

1. **Orchestration strategy is a platform-level decision, not a tool-level one.** A builder chooses the orchestration model (ReAct, ReWoo, workflow-based, generative planning, classic topic-routing) when creating an agent or flow; it is not typically a runtime decision per conversation. AWS Bedrock's custom orchestration via Lambda is an exception, allowing per-request control, but this requires builder-written Lambda code.

2. **Slot-filling is implemented two ways: LLM-driven (automatic clarifying questions) vs. deterministic (state machine).** Copilot Studio's generative mode and Azure CLU infer missing inputs and ask; Google's CXAS SCRAPI framework and Dialogflow ES require explicit configuration. Neither is exposed as a platform service across all systems — each is designed differently.

3. **Human handoff preserves conversation context but not uniformly.** ServiceNow passes context via interaction records; Copilot Studio → ServiceNow passes full activity history via DirectLine. AWS and Google platforms do not document handoff mechanics in public sources. Handoff context is typically one-way (agent → human); reverse handoff (human back to agent) is not documented.

4. **Plan-then-execute orchestration exists (AWS ReWoo, Google Blueprint) but is positioned as an alternative to reactive loops, not as a default.** It suits predictable, high-volume workflows where observation does not change the next action. It is not documented as the recommended pattern for equipment-monitoring or diagnostic agents, where the next question or action often depends on prior results (anomaly severity, sensor readings, user feedback).

5. **Crisis/incident-mode orchestration is not a documented pattern.** Platforms support escalation, routing, and approval gates, but no published documentation describes changing the agent's orchestration strategy, tool set, or action sequence based on runtime detection of an emergency. This may be because (a) the information boundary and tool restrictions (L1/L2) are deemed sufficient without orchestration changes, or (b) builders implement crisis logic in custom orchestration or system prompts rather than platform-level features.

---

## Primary sources referenced

### Multi-Turn Dialog and Task Orchestration

- [How Amazon Bedrock Agents works - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-how.html) — build-time vs. runtime operations, orchestration strategy overview
- [Customize your Amazon Bedrock Agent's behavior with custom orchestration - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-custom-orchestration.html) — custom orchestration via Lambda, control over model and tool invocation
- [Getting started with Amazon Bedrock Agents custom orchestrator - AWS ML Blog](https://aws.amazon.com/blogs/machine-learning/getting-started-with-amazon-bedrock-agents-custom-orchestrator/) — ReAct vs. ReWoo comparison, use case analysis
- [Use multi-agent collaboration with Amazon Bedrock Agents - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-multi-agent-collaboration.html) — supervisor-and-collaborator agent pattern, hierarchical collaboration
- [Create multi-agent collaboration - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/create-multi-agent-collaboration.html) — configuration steps, maximum collaborator agents limit (10)
- [Build a workflow in Microsoft Foundry - Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/workflow) — declarative workflows, orchestration patterns (Sequential, Human-in-the-loop, Group chat)
- [Introducing Multi-Agent Workflows in Foundry Agent Service - Microsoft Foundry Blog](https://devblogs.microsoft.com/foundry/introducing-multi-agent-workflows-in-foundry-agent-service/) — visual and YAML-based workflows, variable handling
- [Workflows: multi-agent, multi-node applications - Google ADK](https://adk.dev/workflows/) — graph-based, dynamic, collaborative, and template workflows
- [Developer's guide to multi-agent patterns in ADK - Google Developers Blog](https://goo.gle/3Ng8qVt) — Sequential Pipeline and Coordinator/Dispatcher patterns
- [Orchestrate agent behavior with generative AI - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/advanced-generative-actions) — generative vs. classic orchestration, comparison table
- [Apply generative orchestration capabilities - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/generative-orchestration) — LLM-driven planning, knowledge layer, tools and connectors, topics

### Slot-Filling and Clarifying-Question Orchestration

- [Conversational language understanding (CLU) entity slot filling in multi-turn conversations - Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/language-service/conversational-language-understanding/concepts/multi-turn-conversations) — progressive disclosure, entity extraction, slot-filling mechanics
- [Converse with an Amazon Bedrock flow - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/flows-multi-turn-invocation.html) — multi-turn conversation in flows, pause for clarification, execution ID tracking
- [Actions and parameters - Dialogflow ES](https://docs.cloud.google.com/dialogflow/es/docs/intents-actions-parameters) — required parameters, slot filling with prompts
- [Slot Filling - CXAS SCRAPI](https://googlecloudplatform.github.io/cxas-scrapi/stable/guides/slot-filling/) — Slot Filling DAG Framework, deterministic state machine approach
- [Use entities and slot filling in agents - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/advanced-entities-slot-filling) — prebuilt and custom entities, entity extraction in multi-turn conversations

### Human-Handoff Orchestration

- [Transferring Virtual Agent conversations to a live agent - ServiceNow](https://www.servicenow.com/docs/r/conversational-interfaces/virtual-agent/transfer-to-live-agent.html) — `vaSystem.connectToAgent()` method, transfer modes (automatic, scripted, manual)
- [Solved: Can you pass Live Agent Variables (Contexts) from Virtual Agent to an Incident - ServiceNow Community](https://www.servicenow.com/community/virtual-agent-nlu-forum/can-you-pass-live-agent-variables-contexts-from-virtual-agent-to/m-p/224261) — context passing via Interaction records, context_table/context_document structure
- [Hand off to ServiceNow - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/customer-copilot-servicenow) — DirectLine relay to ServiceNow, conversation history preservation in activities array

### Plan-Then-Verify Orchestration

- [Reasoning - Reactive Agents](https://docs.reactiveagents.dev/guides/reasoning/) — ReAct, Blueprint (plan-verify-execute-solve), Reflexion (generate-critique-improve) patterns
- [ReAct (Reason + Act): Interleaved Reasoning-Action Loops - AgentPatterns.ai](https://agentpatterns.ai/agent-design/react-pattern/) — ReAct vs. plan-then-execute comparison, when each pattern wins
- [ReAct: Synergizing Reasoning and Acting in Language Models - arXiv](https://arxiv.org/pdf/2210.03629v3) — original ReAct paper (Yao et al., 2022)

### Incident/Crisis Orchestration

- [Incident Workflows - PagerDuty Knowledge Base](https://support.pagerduty.com/main/docs/incident-workflows) — automated incident response, orchestration rules, conditional logic
- [A Deep-Dive Into PagerDuty's New Incident Workflows - PagerDuty Blog](https://www.pagerduty.com/blog/incident-management-response/incident-workflows-deep-dive/) — use cases for incident automation
- [How Security Extracts Relevant Data for Immediate Incident Coordination - PagerDuty](https://www.pagerduty.com/ops-guides/using-pd/security-team/) — orchestration rules for payload extraction, conditional notifications
- [Latest access control security enhancements for AI Agents and Skill Kit - ServiceNow Community](https://www.servicenow.com/community/now-assist-articles/latest-access-control-security-enhancements-for-ai-agents-and/ta-p/3374036) — role masking, supervised execution mode (applied at tool definition, not runtime)

