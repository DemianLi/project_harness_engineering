# Factory Equipment Monitoring Chat Agent: The Memory/State Externalization Layer (L4)

This document investigates memory and state externalization mechanisms for end-user-facing factory equipment monitoring and crisis-handling chat agents. The L4 layer determines how agents retain conversation context within and across sessions, how that memory is scoped in multi-tenant deployments, how state is persisted during long-running or multi-step tasks, and what memory persists after human handoff. Unlike the information boundary (L1), tool system (L2), and orchestration mechanics (L3), the memory layer is responsible for *what the agent remembers*, not what it can access, invoke, or execute.

Scope: session-based conversation memory mechanics; cross-session long-term memory extraction and retrieval; multi-tenant memory isolation patterns; state externalization and checkpointing for long-running or polling-based tasks; memory persistence and updating across human handoff boundaries; and the treatment of real-time telemetry as decaying state vs. per-query retrieval.

---

## Search tooling notes

This pass used the `anysearch` skill for web search, followed by targeted `WebFetch` of official documentation for AWS Bedrock (both Classic/Agents and AgentCore), Azure AI Foundry, Google Vertex AI ADK, Microsoft Copilot Studio, and ServiceNow. Primary sources include:

- **AWS Bedrock Agents**: Memory documentation for the Classic line (now in sunset maintenance), plus AgentCore Memory (the replacement product)
- **AWS Bedrock Flows**: Execution state tracking for async/long-running flows, documented in the Flows asynchronous execution guide
- **Azure AI Foundry**: Threads and memory concepts (preview), conversation state persistence
- **Google Vertex AI ADK**: Session/State/Memory service architecture; Memory Bank long-term memory
- **Google Vertex AI Agent Platform**: Memory Bank with memory generation, extraction, consolidation, TTL, and identity-scoped retrieval
- **Microsoft Copilot Studio**: Memory (preview) feature, per-user per-agent memory stores, conversation history in Teams persistence
- **ServiceNow Now Assist**: Long-term memory categories, multi-turn context retention (community-sourced detail and Masterclass); Interaction records for handoff state
- **AWS Well-Architected Lens (Agentic AI)**: Multi-tenant memory isolation best practices and namespace scoping
- **Third-party frameworks and research**: Dakera session handoff patterns, Agentium multi-tenant primitives, agent-memory library (decay/forgetting policies), ZeptoDB for telemetry + memory timeline, industrial IoT RAG patterns

Industrial platforms (Siemens, Honeywell, GE) did not yield technical documentation on memory/state externalization beyond the L1 doc's findings. Community-sourced material from ServiceNow forums and Azure Q&A are explicitly marked as such. A distinct gap was identified across all five primary platforms: no official documentation describes an agent extracting and persisting insights from a human's resolution post-handoff as a learning mechanism.

---

## 1. Conversation/session memory mechanics on enterprise chat/agent platforms

How do platforms persist conversation state across multiple turns within a single session?

### AWS Bedrock Agents (Classic)

**Timeliness note:** AWS Bedrock Agents (launched November 2023) is now termed "Amazon Bedrock Agents Classic" and enters maintenance-only mode on 2026-07-30, no longer accepting new customers. The replacement is AWS Bedrock AgentCore, documented below. For existing Classic customers, the Classic documentation remains valid through the sunset date.

Per [Retain conversational context across multiple sessions using memory - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-memory.html), Classic agents support agent memory that retains conversational context by default (within a single session). Sessions are identified by a builder-provided `sessionId`; the same ID across requests continues the same conversation. When the agent ends a session (via `endSession: true` or idle timeout), that session's memory context is given a unique memory identifier (`memoryId`). The agent stores the memory for each user against that `memoryId`. Memory retention is configurable from 1 to 365 days.

### AWS Bedrock AgentCore — managed memory with automatic persistence

Per [Add memory to your Amazon Bedrock AgentCore agent - Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory.html) and [Memory - Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-memory.html), AgentCore Memory is a fully managed service providing two memory types:

1. **Short-term memory**: Captures turn-by-turn interactions within a single session. Per the documentation, "Short-term memory stores raw interactions that help the agent maintain context within a single session."
2. **Long-term memory**: Automatically extracts and stores key insights from conversations across multiple sessions.

Critically, per the harness documentation: "The harness automatically persists conversation state in AgentCore Memory. On every invocation, the conversation is saved, scoped by session ID (and additionally by actor ID, if provided). On subsequent invocations with the same session ID, the agent's history is loaded from Memory before it reasons — it remembers what happened in previous turns, even after the underlying microVM session has expired. You do not need to pass previous messages yourself; just send the new message."

Memory events are scoped by `actorId + sessionId`, isolating each user's memory. Memory is enabled by default in managed mode with 30-day event expiry, with optional custom configuration of event expiry (the example sets 60 days) and strategy selection at harness creation time.

### Azure AI Foundry — conversations (current) and threads (classic)

**Terminology note:** the current (non-classic) Foundry Agent Service documentation has moved away from "threads" as the primary conversation-state concept. Per [Build with agents, conversations, and responses in Foundry Agent Service - Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/runtime-components), the service now uses **conversations**: "Conversations are durable objects with unique identifiers. After creation, you can reuse them across sessions," and "Conversations store items, which can include messages, tool calls, tool outputs, and other data." The same page explicitly frames this as a replacement for the older model: building multi-turn flows by chaining response IDs "gives you more flexibility than the older thread-based pattern, where state was tightly coupled to thread objects." By default, "the service stores response history server-side, so you can reference `previous_response_id` for multi-turn context"; setting `store` to `false` means "the service doesn't persist the response" and the caller must carry forward context manually — useful "in a zero-data-retention environment."

Per [Threads, Runs, and Messages in the Foundry Agent Service - Microsoft Learn](https://github.com/MicrosoftDocs/azure-ai-docs/blob/main/articles/foundry-classic/agents/concepts/threads-runs-messages.md) (the **classic** variant, a separate, older documentation tree from the current page above), "Threads are conversation sessions between an agent and a user. They store messages and automatically handle truncation to fit content into a model's context." A thread can hold "up to 100,000 [messages] per thread," and "In the Standard agent setup, threads are stored in your Azure Cosmos DB account." This thread-based model is the one being superseded by the conversations/responses model described above, not a currently-recommended pattern for new builds.

### Google Vertex AI ADK — session and state services

Per [Conversational Context: Session, State, and Memory - Agent Development Kit](https://adk.dev/sessions/), ADK provides three core concepts:

- **Session**: The current conversation thread, containing the chronological sequence of messages and actions (referred to as "Events") during a specific interaction.
- **State** (`session.state`): Key-value pairs of data within a specific Session, used to manage information relevant only to the current conversation.
- **Memory**: A store of information spanning multiple past sessions or including external data sources.

Per [State: The Session's Scratchpad](https://github.com/google/adk-docs/blob/main/docs/sessions/state.md), `session.state` uses key-value pairs, and keys can carry prefixes that define scope and persistence behavior:

- No prefix (session state): Scoped to the current session (`id`), persists only if the SessionService is persistent.
- `user:` prefix: Scoped to the `user_id`, shared across all sessions for that user; persistent with Database or VertexAI services.
- `app:` prefix: Tied to `app_name`, shared across all users and sessions; persistent with Database or VertexAI services.

SessionService implementations vary: `InMemorySessionService` (not persistent), `DatabaseSessionService` (persistent), and `VertexAiSessionService` (persistent, cloud-backed).

### Microsoft Copilot Studio — per-user, per-agent memory store

Per [Memory (preview) - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/agents-experience/memory-overview), Copilot Studio Memory (preview) "lets an agent remember details from interactions and apply that context on future ones." "An agent with Memory turned on maintains a separate memory store for each user."

The lifecycle is: (1) Capture—the agent records signals like user preferences or notes; (2) Store—signals are saved as files in the user's per-agent memory folder, in a tenant-scoped store; (3) Apply—on subsequent interactions, the agent reads from this memory to inform responses.

Memory is a controlled, agent-level capability; makers decide whether an agent uses memory. If a user has no activity with an agent for 28 days, their memories for that agent are deleted.

### ServiceNow Now Assist — conversation context and multi-turn persistence

Per [AI Agent Masterclass - Session 4 - ServiceNow Community](https://www.servicenow.com/community/now-assist-articles/ai-agent-masterclass-session-4/ta-p/3469309) (a community-sourced masterclass, not official documentation), ServiceNow provides three types of memory: Short-Term Memory (current conversation context, single session), Long-Term Memory (persistent user information across sessions), and Episodic Memory (specific past events).

Per [ServiceNow Virtual Agent: Multi-Turn Context in 2026 - CRM Curator](https://crmcurator.com/articles/servicenow/servicenow-virtual-agent-multi-turn-context/) (third-party blog, community-sourced), the 2026 release supports multi-turn conversations with context retention across sessions. Session state lives in `sys_cs_conversation` with a configurable retention window per topic. Session resume policies include: default (resume within 24h), extended (resume within 7d), short (resume within 1h), and ephemeral (do not retain context across sessions).

---

## 2. Cross-session / long-term memory: persistence across separate conversations for the same end user

Do platforms support memory that persists ACROSS separate sessions? If so, how is it scoped, and is it opt-in or opt-out?

### AWS Bedrock AgentCore — automatic long-term memory extraction

Per [Add memory to your Amazon Bedrock AgentCore agent - Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory.html), "Long-term memory automatically extracts and stores key insights from conversations across multiple sessions, including user preferences, important facts, and session summaries — for persistent knowledge retention across multiple sessions."

Long-term memory generation is documented as asynchronous: "Long-term memory generation is an asynchronous process that runs in the background and automatically extracts insights after raw conversation/context is stored in short-term memory via `CreateEvent`."

Per [Memory types - Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-types.html), the system "consolidates newly extracted information with existing memories, allowing memories to evolve as new information is ingested." Examples include user preferences (e.g., "prefer window seats") that persist across multiple interactions.

Managed-mode default event expiry: 30 days. A maximum retention bound for long-term memory records was not stated in the AgentCore sources checked. (The 1–365 day range is documented for Bedrock Agents Classic, not AgentCore.)

**Status: opt-out (enabled by default)** — managed memory is enabled by default when creating a harness; it can be disabled by passing `--no-harness-memory`.

### Azure AI Foundry — memory (preview) with custom persistence

Per [Memory in Microsoft Foundry Agent Service (preview) - Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/what-is-memory), "Memory in Foundry Agent Service is a managed, long-term memory solution. It enables agent continuity across sessions, devices, and workflows."

The documentation describes three phases: Extraction (system actively extracts key information from conversations), Consolidation (extracted memories are consolidated to keep the store efficient, with LLM-driven merging of duplicates and conflict resolution), and Retrieval (agent searches the memory store for the most relevant memories).

However, per a community-sourced answer in [Azure AI Foundry: How to persist memory across threads - Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/5542041/azure-ai-foundry-how-to-persist-memory-across-thre) (dated 2025-09-02, community-sourced, not official guidance), "Since there is no out-of-the-box 'shared memory across threads' feature in Azure AI Foundry today, you can design a custom persistence strategy" using external state stores (Azure Cosmos DB, Azure Table Storage, Azure Redis Cache) or custom middleware.

**Status: opt-in preview feature; requires custom persistence beyond threads as of the community response date.**

### Google Vertex AI — Memory Bank with configurable strategies and identity scoping

Per [Agent Platform Memory Bank - Gemini Enterprise Agent Platform](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/memory-bank), "Agent Platform Memory Bank lets you dynamically generate long-term memories based on conversations between the user and your agent. These memories are personalized information that persists across multiple sessions, enabling your agent to adapt and personalize responses for context and continuity."

Memory Bank provides:
- Memory generation: Create, refine, and manage memories using an LLM
- Memory extraction: Extract only the most meaningful information from source data
- Memory consolidation: Consolidate newly extracted information with existing memories
- Asynchronous generation: Generate memories in the background
- Data isolation across identities: Memory consolidation and retrieval is isolated to a specific identity
- Automatic expiration: Set a TTL on memories to ensure stale information is automatically deleted

The documentation states: "Memory revisions: Automatically create and maintain memory revisions which allow you to inspect how memories transform as new information is ingested."

Per [Building Stateful and Personalized Agents with ADK - Google Codelabs](https://codelabs.developers.google.com/codelabs/agent-memory/instructions), the pattern for cross-session memory is: Session → State → MemoryService → long-term store.

**Status: available as a configurable component of Vertex AI Agent Platform; requires explicit provisioning and configuration per the documentation checked.**

### Microsoft Copilot Studio — memory (preview) across interactions

Per [Memory (preview) - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/agents-experience/memory-overview), "With Memory turned on, agents can: Recall relevant details from previous interactions, Apply learned patterns or corrections to future interactions, [and] Provide more contextual, personalized, and efficient responses."

The documentation confirms per-user isolation: "Memory is stored per user, per agent. Each user has their own memory folder for each memory-enabled agent they interact with."

**Status: opt-in preview feature** — builders toggle Memory on/off in the agent's Build tab.

### ServiceNow — long-term memory with category-based organization

Per official ServiceNow documentation, [Long-Term Memory for AI Agents - ServiceNow](https://www.servicenow.com/docs/r/intelligent-experiences/long-term-memory-aia.html) describes LTM as enabling AI agents to retain and reference information across multiple user interactions. The system operates through **LTM categories**, which are organizational units for storing and managing persistent information. Organizations must map long-term memory categories to specific AI agents to enable the functionality.

Per [Long-Term Memory Categories for Now Assist AI Agents - ServiceNow](https://www.servicenow.com/docs/r/intelligent-experiences/aia-ltm-categories.html), setting up long-term memory involves: (1) Creating categories tailored to specific use cases, (2) Mapping those categories to particular AI agents, and (3) Integrating the memory system with agent workflows. The system supports "citation tracking for retrieved information" and enables agents to maintain contextual information over time.

The Masterclass and third-party sources (e.g., [AI Memory in ServiceNow - The Digital Iceberg](https://www.thedigitaliceberg.com/post/ai-memory-in-servicenow)) note that memory categories can store domain-specific information, user preferences, and contextual details; relevance scores track how often memories are recalled to surface the most valuable information.

**Status: configurable capability; official documentation confirms multi-interaction persistence but does not specify opt-in vs. default.**

---

## 3. Multi-tenant memory isolation: server-side derivation and scoping patterns

This section builds on L1's multi-tenant data isolation findings (silo/pool/bridge patterns, server-side tenant ID derivation) by focusing specifically on how memory/state storage is partitioned at the data layer.

### AWS Bedrock AgentCore — hierarchical namespace system with IAM scoping

Per [AGENTSEC01-BP01 Implement memory isolation and integrity controls - AWS Well-Architected Lens (Agentic AI)](https://docs.aws.amazon.com/wellarchitected/latest/agentic-ai-lens/agentsec01-bp01.html), "Amazon Bedrock AgentCore Memory expresses this partitioning through a hierarchical namespace system built on actorId and sessionId identifiers, with dynamic placeholders like `{actorId}` and `{sessionId}` that resolve at call time."

The guidance states: "A namespace template like `/support-agent/{actorId}/preferences` gives each user an automatically scoped partition without hardcoding identifiers. The invoking session determines which namespace the agent reads from or writes to, and the IAM policy on the agent's role bounds which namespace prefixes it can touch, so even a namespace constructed incorrectly by the agent is denied by the authorization layer."

Critically: "Shared namespaces are not an anti-pattern when they are intentional. Model them as named namespaces (for example, `/support-agent/shared/product-knowledge` or `/team-{teamId}/shared/working-context`) and scope each agent role's IAM policy to the specific shared namespaces that agent is authorized to reach. The failure mode to help prevent is sharing by default, not sharing that has been designed and authorized."

The documentation also notes integrity controls: "AgentCore Memory keeps an append-only audit trail where the consolidation process marks outdated records as INVALID rather than deleting them, so the full sequence of changes is recoverable for forensic analysis." It adds that this "helps protect history, but it doesn't detect whether a specific record was silently modified outside the normal consolidation flow" — for that, the guidance recommends layering AWS KMS HMAC signatures on individual records.

### Google Vertex AI ADK — state prefixes and SessionService scoping

Per [Conversational Context: Session, State, and Memory - ADK](https://adk.dev/sessions/) and [State: The Session's Scratchpad](https://github.com/google/adk-docs/blob/main/docs/sessions/state.md), ADK's state prefixes define scope isolation:

- No prefix: Session-scoped, example `session.state['current_intent'] = 'book_flight'`
- `user:` prefix: User-scoped (tied to `user_id`, shared across all sessions for that user), example `session.state['user:preferred_language'] = 'fr'`
- `app:` prefix: App-scoped (tied to `app_name`, shared across all users and sessions)

Persistence depends on the SessionService implementation: InMemorySessionService (not persistent, not suitable for production multi-tenant), DatabaseSessionService (persistent, tenant-aware via external configuration), VertexAiSessionService (persistent, cloud-backed).

### Azure AI Foundry — thread-level and memory store scoping

Per the Memory documentation, "You control access using the `scope` parameter, which segments memory across users to ensure secure and isolated experiences." Threads persist until explicitly deleted, and the builder controls where threads are stored (Azure Cosmos DB or Microsoft-managed storage).

### ServiceNow — ACL-based memory access control

Per [Implement access control in Now Assist AI agents - ServiceNow](https://www.servicenow.com/docs/r/intelligent-experiences/aia-security-implementation.html) (documented in L1), ServiceNow's security model comprises:
- **Access Control Lists (ACLs)** — determine which role(s) a user must have to invoke an agent.
- **User identity** — determines which user the agent operates as during execution (Dynamic user or AI user).
- **Role masking** — minimizes the roles with which AI Agents can execute.

The agent's user identity and roles determine what memory it can access via ACL checks. This is a role-based access pattern rather than a namespace-based partition.

### Multi-tenant namespace rules — third-party framework pattern

Per [Memory/isolation - Multi-User Isolation](https://xhipment.mintlify.app/memory/isolation) (third-party framework, Agentium, not official vendor documentation), a structured approach to multi-tenant isolation defines a scope hierarchy: user → agent → tenant → global. The documentation prescribes: "**`scope` defaults to `'user'`** when omitted on a save — so existing code that didn't know about scopes keeps the same isolation guarantees. **Auto-extracted learnings/procedures always save as `'user'`** — the framework never auto-promotes an LLM-extracted insight to a shared scope without the caller explicitly choosing that."

---

## 4. State externalization for long-running / multi-step tasks with checkpoint/resume patterns

Equipment monitoring and crisis handling can span multiple turns or require background polling. How do platforms externalize state for such tasks?

### AWS Bedrock Flows — execution ID state tracking and async checkpoint management

Per [Run Amazon Bedrock flows asynchronously with flow executions - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/flows-create-async.html), flow executions allow longer durations than synchronous invocations: "Individual nodes can run up to five minutes, and your entire flow can run for up to 24 hours." Per the same documentation, synchronous flows (via InvokeFlow) time out at one hour, while flow executions (async via StartFlowExecution) can run up to 24 hours.

Crucially, per the StartFlowExecution API documentation: "When you run a flow by using the Amazon Bedrock console or with the InvokeFlow operation, the flow runs until it finishes or times out at one hour (whichever is first). When you run a flow execution, your flow can run much longer."

Each flow execution is assigned an execution ARN (Amazon Resource Name). Per [StartFlowExecution - Amazon Bedrock API Reference](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_StartFlowExecution.html), the builder specifies:
- `flowExecutionName` (optional) — a user-provided name for tracking
- `inputs` — the initial input data matching the flow's input schema
- `modelPerformanceConfiguration` (optional) — performance tuning

The response includes `executionArn`, which uniquely identifies the flow execution.

Per [GetFlowExecution - Amazon Bedrock API Reference](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_GetFlowExecution.html), the builder polls the execution state using this ARN. The response includes:
- `status` — "Running", "Succeeded", "Failed", "TimedOut", or "Aborted"
- `startedAt` / `endedAt` — timestamps
- `errors` — list of errors with node names
- Traces (if `enableTrace` was set) — inputs and outputs from each node

This is a **checkpoint-and-resume pattern**: the platform stores the execution state server-side, keyed by execution ARN. The builder polls for status and retrieves results asynchronously.

Per [Long-running execution flows now supported in Amazon Bedrock Flows - AWS ML Blog](https://aws.amazon.com/blogs/machine-learning/long-running-execution-flows-now-supported-in-amazon-bedrock-flows-in-public-preview/), flow executions enable "decoupling the workflow execution (asynchronously through long-running flows that can run for up to 24 hours) from the user's immediate interaction, you can now build applications that can handle large payloads that take longer than 5 minutes to process, perform resource-intensive tasks, apply multiple rules for decision-making, and even run the flow in the background."

### AWS Bedrock Flows (synchronous) — in-session execution ID

Per [InvokeFlow - Amazon Bedrock API Reference](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_InvokeFlow.html), the synchronous flow invocation accepts an optional `executionId` parameter: "The unique identifier for the current flow execution. If you don't provide a value, Amazon Bedrock creates the identifier for you."

This allows reusing the same execution ID across multiple `InvokeFlow` calls within a session, maintaining execution context. The response header `x-amz-bedrock-flow-execution-id` echoes back the execution ID.

### Azure AI Foundry — workflow variables and state passing

Per [Build a workflow in Microsoft Foundry - Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/workflow), workflows are "declarative, predefined sequences of actions that orchestrate agents and business logic." Variables are used to pass state between steps: the documentation states "Variables and JSON schema output to capture agent output as local variables, reference them in later steps."

While explicit long-running state checkpointing is not documented with the same granularity as Bedrock Flows, the variable passing mechanism allows intermediate state to be persisted across workflow steps.

### Google Vertex AI ADK — session persistence for multi-turn dialogs

Per [Conversational Context: Session, State, and Memory - ADK](https://adk.dev/sessions/), the Session object holds Events (the chronological sequence of messages and actions). A SessionService manages the lifecycle of Session objects. When using a persistent SessionService (DatabaseSessionService or VertexAiSessionService), the session state is persisted server-side and can be retrieved by session ID across invocations.

This is a session-resumption pattern rather than explicit checkpoint/polling, but it achieves long-task state externalization by persisting session context.

### Durable execution patterns — temporal/Step Functions not officially documented in agent platforms

AWS Step Functions and Temporal are often used as backing infrastructure for agent state in custom implementations, but none of the platforms surveyed document this as a first-class feature. Bedrock Flows' execution tracking is the closest official pattern.

---

## 5. What state crosses the human-handoff boundary vs. what memory persists after handoff

This section builds on L3's handoff findings (ServiceNow Interaction records, Copilot Studio DirectLine relay) by examining memory-specific behaviors: does the platform continue accumulating/updating agent-visible memory after handoff, or is it frozen at handoff time? Is there a documented pattern for agents "learning" from human resolutions?

### ServiceNow — context passing via Interaction records (preserved at handoff)

Per L3's research on [Transferring Virtual Agent conversations to a live agent - ServiceNow](https://www.servicenow.com/docs/r/conversational-interfaces/virtual-agent/transfer-to-live-agent.html) and community-sourced detail from [Solved: Can you pass Live Agent Variables from Virtual Agent to an Incident - ServiceNow Community](https://www.servicenow.com/community/virtual-agent-nlu-forum/can-you-pass-live-agent-variables-contexts-from-virtual-agent-to/m-p/224261) (a community forum thread, not official documentation), the pattern is:

1. Agent sets LiveAgent_* variables during conversation (e.g., `vaVars.LiveAgent_short_description = vaInputs.user_input`)
2. On escalation, these are stored in an Interaction record with `context_table` and `context_document` fields
3. The live agent retrieves the context by querying the Interaction record

**Frozen at handoff:** The documented flow does not describe the agent continuing to receive or update memory after the human takes over. The Interaction record captures what was gathered; it is not clear whether new information collected by the human gets written back to long-term memory.

### Microsoft Copilot Studio → ServiceNow — DirectLine relay with history

Per L3's [Hand off to ServiceNow - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/customer-copilot-servicenow), Copilot Studio escalates to a ServiceNow live agent via a DirectLine relay. The documentation shows an `activities` array with full message history and a `handoff.initiate` event, allowing ServiceNow Bot Interconnect to receive the full conversation context before the live agent joins.

**Frozen at handoff:** The conversation history up to handoff is passed; subsequent updates from the human are not documented as being written back to the agent's long-term memory.

### Learning from human resolution — NOT FOUND in verified sources

**Key finding:** Across all five platforms (AWS Bedrock, Azure AI Foundry, Google Vertex AI, Microsoft Copilot Studio, ServiceNow), no official documentation describes a pattern where the agent "learns" from the human's resolution by automatically extracting and persisting the resolution as a long-term memory update or correction.

Third-party patterns (Dakera, agent-memory libraries, handoff frameworks) discuss "storing handoff summaries" and "capturing corrections from the user," but these are application-level patterns, not platform-native capabilities. The research specifically looked for:
- An API call or feature to write human-generated resolution summaries into long-term memory
- Automatic extraction of "what the human did" from the handoff conversation to update agent beliefs
- Documented patterns for agents retrieving "how similar cases were resolved by humans"

None of these were found in official vendor documentation. This is a consistent absence across platforms, suggesting either:
1. Platforms expect builders to implement this pattern in application code (not as a platform capability)
2. Platforms treat agent memory as frozen at handoff, not updated post-human-resolution
3. Some platforms implement it but do not document it publicly

---

## 6. Real-time / streaming telemetry state as a distinct memory concern

Is a rolling window of recent sensor readings treated as part of the agent's working memory/state (with defined retention/decay), or is it purely a per-query RAG lookup with no memory component?

### L1 findings revisited — Flash-Fusion and edge summarization

The L1 document (factory-agent-harness-l1-information-boundary.md, §4) established that "raw IoT data is not fed to the LLM at all. Instead, pre-processed, summarized features selected for the specific query are injected into the prompt."

Per [Flash-Fusion: Enabling Expressive, Low-Latency Queries on IoT Sensor Streams with LLMs](https://ar5iv.labs.arxiv.org/html/2511.11885) (arXiv 2511.11885, Georgia Tech, 2025), edge-based statistical summarization reduces telemetry by 73.5% on a university bus fleet by summarizing sensor streams (averages, percentiles, anomalies) at the edge, followed by cloud-based query planning to select relevant data slices.

### Memory vs. retrieval distinction — not found in sources checked

Across the official documentation checked for AWS Bedrock, Azure AI Foundry, Google Vertex AI, Microsoft Copilot Studio, and ServiceNow, there is **no documented distinction** between:
- State (mutable, decaying, session/user-scoped, retained in memory)
- Static knowledge (stable, retrieved via RAG, not memory)
- Real-time telemetry (stream of measurements, refreshed at query time)

The absence in verified sources suggests that platforms expect telemetry to be handled via tool calls or RAG retrieval, not as part of the agent's memory/state layer. This is consistent with L1's findings that telemetry is injected at query time via tool calls (AegisFlow, Jeltz) or RAG retrieval, not accumulated in agent memory.

### Industrial IoT patterns — RAG-based, not memory-based

Per [RAG in Industrial IoT: How AI Agents Actually Reason About Telemetry - Cloud Studio IoT](https://cloudstudioiot.com/hub/rag-in-industrial-iot), industrial IoT agents retrieve telemetry on-demand from four sources: (1) device metadata (entity catalog), (2) latest sensor readings (device shadow / current state), (3) historical time-series data (analytics and trend), and (4) alert/event logs. These are retrieved per query, not accumulated in agent memory.

The pattern is: query → retrieval (latest readings from state store, historical from time-series DB, metadata from catalog) → augmented prompt → LLM response.

### Conclusion on telemetry state

Real-time telemetry is treated as **per-query retrieval** (RAG), not as **decaying memory state**. This is consistent across all verified sources. If an agent needs to reason about trends or anomalies (e.g., "the pump temperature has been rising for 30 minutes"), the application is responsible for:
1. Retrieving the current and historical readings at query time
2. Composing them into the prompt or session context
3. The agent then reasons over that snapshot

No platform documents automatic memory decay or rolling-window state retention for sensor data.

---

## Comparison: memory/state mechanisms across platforms

| Platform | Short-term session memory | Long-term cross-session memory | Multi-tenant isolation | Long-running state externalization | Post-handoff memory | Telemetry as state |
|---|---|---|---|---|---|---|
| **AWS Bedrock AgentCore** | Managed, automatic, scoped by sessionId + actorId | Automatic extraction via async LLM, asynchronous, 30-day event expiry (configurable) | Hierarchical namespace with {actorId}, {sessionId} placeholders; IAM-scoped access | Flow executions: execution ARN tracking, up to 24h, polling via GetFlowExecution | Not documented as updating post-handoff; context frozen | Not found |
| **Azure AI Foundry** | Conversations (current) persist server-side by default, reusable across sessions; classic Threads persisted in Cosmos DB, up to 100K messages per thread | Memory (preview) managed long-term solution; custom persistence via external stores (Cosmos, Table, Redis) required per community guidance | Thread-level and memory store scoping via `scope` parameter | Workflow variables pass state between steps; no explicit async checkpoint API documented | Not documented as updating post-handoff | Not found |
| **Google Vertex AI / ADK** | Session/State with SessionService; state keys use prefixes (none=session, `user:`=user-scoped, `app:`=app-scoped) | Memory Bank: configurable strategies (semantic, summarization, user preference, episodic), TTL, asynchronous generation | State prefixes and SessionService provide scope isolation; identity-scoped Memory Bank retrieval | Session persistence via SessionService; no explicit async checkpoint API | Not documented as updating post-handoff | Not found |
| **Microsoft Copilot Studio** | Per-user per-agent memory store (preview), captured/stored/applied lifecycle, tenant-scoped files | Memory (preview) learns patterns across interactions, 28-day idle deletion | Per-user per-agent memory folders, tenant-scoped storage | Not documented | Not documented as updating post-handoff | Not found |
| **ServiceNow Now Assist** | Multi-turn context via session resume policies (24h/7d/1h/ephemeral); `sys_cs_conversation` table | Long-term memory (LTM) with memory categories, relevance scoring; episodic and short-term also supported | ACL-based memory access via user identity and roles | Not documented | Interaction records preserve context at handoff; frozen thereafter | Not found |

---

## Key gaps: areas where primary-source material was not found

- **Azure AI Foundry**: Exact mechanics of automatic memory consolidation (conflict resolution, deduplication logic) across threads not detailed in sources checked.
- **Google Vertex AI ADK**: Post-handoff memory updating patterns not documented.
- **Microsoft Copilot Studio**: Exact memory retention/expiration mechanics beyond the 28-day idle deletion not specified.
- **ServiceNow**: Official documentation on long-term memory was retrieved confirming category-based LTM and mapping to agents; however, implementation details (relevance scoring mechanics, automatic vs. manual category creation) remain sourced from community posts and Masterclass content.
- **All platforms**: No documented pattern for agents extracting and persisting resolution insights from human takeovers as learning. This is a consistent cross-platform gap. Third-party patterns (Dakera handoff summaries, Arize context graphs for disagreement mining, Luis Beqja's human-in-the-loop feedback loops) demonstrate the capability at the application level, but none are documented as first-class platform features in the official vendor documentation checked.
- **All platforms**: No distinction found in sources checked between "decaying telemetry state" and "per-query RAG retrieval" for real-time sensor data. Industrial IoT patterns uniformly treat telemetry as retrieved-on-demand, not accumulated in memory.
- **Industrial platforms (Siemens, Honeywell, GE, Schneider Electric)**: No public technical documentation on memory/state mechanics located.

---

## Observations

1. **Session memory is automatic; cross-session memory is intentional.** Within a single session (or thread), all platforms automatically persist conversation history server-side. Cross-session long-term memory requires explicit opt-in or is a preview feature, reflecting architectural effort to extract and consolidate insights.

2. **Multi-tenant isolation defaults to safe (per-user scoping).** AWS (namespaces with {actorId}), Google (user: prefix), and Microsoft (per-user memory folders) all default to user-scoped isolation. Shared or tenant-level memory requires explicit builder configuration.

3. **Long-running state is externalized, not streamed back to client.** AWS Bedrock Flows and Azure workflows externalize state as server-side artifacts (execution ARN, workflow variables) that the application polls. The agent does not maintain state in the client session.

4. **Handoff preserves context; agent learning from human resolution is not a platform feature.** ServiceNow and Copilot Studio pass conversation history to the human at handoff. However, no official platform documentation describes agents extracting and persisting resolution insights from human takeovers as a learning mechanism. Third-party implementations demonstrate this pattern at the application level (Arize context graphs for disagreement mining, Dakera handoff summaries with feedback loops), but it is not documented as a first-class vendor capability. This suggests either (a) platforms expect builders to implement this pattern in application code, or (b) platforms deliberately freeze agent memory at handoff, leaving post-resolution learning to the business logic layer.

5. **Real-time telemetry is retrieved, not remembered.** Across all platforms and industrial IoT patterns, sensor data is treated as a retrieval source (per-query RAG) rather than accumulated memory/state. If the agent needs to reason about trends, the application must fetch multi-point readings at query time and compose them into context.

6. **Memory extraction is asynchronous and background-executed.** AWS AgentCore, Google Memory Bank, and Azure (implicit) all describe long-term memory generation as asynchronous, decoupled from user interaction. This prevents memory extraction latency from blocking conversation flow.

7. **No platform publicly documents crisis-mode memory changes.** Consistent with L3's finding on orchestration, no platform documents changing the agent's memory scope or retention policy based on crisis/incident classification. Memory isolation and retention are static configuration, not runtime-adapted to urgency.

---

## Primary sources referenced

### AWS Bedrock Agents (Classic and AgentCore)

- [Retain conversational context across multiple sessions using memory - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-memory.html) — session-based memory scoping, memoryId, retention
- [Add memory to your Amazon Bedrock AgentCore agent - Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory.html) — short-term and long-term memory types, use cases, benefits
- [Memory types - Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-types.html) — short-term memory events, long-term memory extraction/consolidation
- [Memory - Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-memory.html) — managed memory defaults, scoping by sessionId and actorId, strategy configuration
- [Get started with AgentCore Memory - Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-get-started.html) — event write/retrieval, session management

### AWS Bedrock Flows (State Externalization)

- [Run Amazon Bedrock flows asynchronously with flow executions - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/flows-create-async.html) — execution ARN, async checkpoint, polling
- [Long-running execution flows now supported in Amazon Bedrock Flows - AWS ML Blog](https://aws.amazon.com/blogs/machine-learning/long-running-execution-flows-now-supported-in-amazon-bedrock-flows-in-public-preview/) — 24-hour async flows, decoupled execution, use cases
- [StartFlowExecution - Amazon Bedrock API Reference](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_StartFlowExecution.html) — flow execution creation, input schema, execution ARN response
- [GetFlowExecution - Amazon Bedrock API Reference](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_GetFlowExecution.html) — status polling, execution trace, error tracking
- [InvokeFlow - Amazon Bedrock API Reference](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_InvokeFlow.html) — synchronous flow invocation, executionId parameter, execution ID header

### AWS Best Practices (Multi-Tenant Isolation)

- [AGENTSEC01-BP01 Implement memory isolation and integrity controls - AWS Well-Architected Lens (Agentic AI)](https://docs.aws.amazon.com/wellarchitected/latest/agentic-ai-lens/agentsec01-bp01.html) — hierarchical namespaces, IAM scoping, integrity checks, shared namespace authorization

### Azure AI Foundry

- [Build with agents, conversations, and responses in Foundry Agent Service - Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/runtime-components) — threads, runs, messages, conversation persistence
- [Threads, Runs, and Messages in the Foundry Agent Service - Microsoft Learn (GitHub)](https://github.com/MicrosoftDocs/azure-ai-docs/blob/main/articles/foundry-classic/agents/concepts/threads-runs-messages.md) — thread lifecycle, message storage, Cosmos DB backing
- [Memory in Microsoft Foundry Agent Service (preview) - Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/what-is-memory) — long-term memory, extraction/consolidation/retrieval phases
- [Create and use memory in Foundry Agent Service (preview) - Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/memory-usage) — memory store creation, scope parameter, retention controls
- [Azure AI Foundry: How to persist memory across threads - Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/5542041/azure-ai-foundry-how-to-persist-memory-across-thre) — community-sourced (2025-09-02); custom persistence pattern via external stores
- [Build a workflow in Microsoft Foundry - Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/workflow) — declarative workflows, variable state passing

### Google Vertex AI / Agent Development Kit

- [Conversational Context: Session, State, and Memory - Agent Development Kit](https://adk.dev/sessions/) — Session, State, Memory concepts; SessionService/MemoryService
- [State: The Session's Scratchpad - ADK (GitHub)](https://github.com/google/adk-docs/blob/main/docs/sessions/state.md) — state key-value pairs, scoping prefixes (none, `user:`, `app:`), SessionService implementations
- [Building Stateful and Personalized Agents with ADK - Google Codelabs](https://codelabs.developers.google.com/codelabs/agent-memory/instructions) — multi-turn memory stack, session → state → memory patterns
- [Agent Platform Memory Bank - Gemini Enterprise Agent Platform](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/memory-bank) — long-term memory generation, extraction, consolidation, TTL, identity scoping
- [Scale your agents - Gemini Enterprise Agent Platform](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/memory-bank/generate-memories) — Agent Runtime, memory bank, sessions, continuous quality improvement

### Microsoft Copilot Studio

- [Memory (preview) - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/agents-experience/memory-overview) — per-user per-agent memory store, capture/store/apply lifecycle, 28-day idle deletion
- [Hand off to ServiceNow - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/customer-copilot-servicenow) — DirectLine relay, conversation activities array, handoff.initiate event
- [Deploy agents in Microsoft Teams - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/deploy-agent-teams) — persistent conversation context in Teams, session lifecycle management

### ServiceNow Now Assist

- [AI Agent Masterclass - Session 4 - ServiceNow Community](https://www.servicenow.com/community/now-assist-articles/ai-agent-masterclass-session-4/ta-p/3469309) — three memory types (short-term, long-term, episodic), context engineering, metrics definition
- [ServiceNow Virtual Agent: Multi-Turn Context in 2026 - CRM Curator](https://crmcurator.com/articles/servicenow/servicenow-virtual-agent-multi-turn-context/) — community blog (third-party, not official); multi-turn context retention, session resume policies, retention windows
- [AI Memory in ServiceNow - The Digital Iceberg](https://www.thedigitaliceberg.com/post/ai-memory-in-servicenow) — community blog (third-party, not official); LTM categories, agent–category mapping, memory table, relevance scoring
- [Long-Term Memory for AI Agents - ServiceNow](https://www.servicenow.com/docs/r/intelligent-experiences/long-term-memory-aia.html) — LTM categories, agent mapping, cross-interaction knowledge retention
- [Long-Term Memory Categories for Now Assist AI Agents - ServiceNow](https://www.servicenow.com/docs/r/intelligent-experiences/aia-ltm-categories.html) — category creation, mapping to agents, citation tracking
- [Transferring Virtual Agent conversations to a live agent - ServiceNow](https://www.servicenow.com/docs/r/conversational-interfaces/virtual-agent/transfer-to-live-agent.html) — vaSystem.connectToAgent(), context passing
- [Solved: Can you pass Live Agent Variables from Virtual Agent to an Incident - ServiceNow Community](https://www.servicenow.com/community/virtual-agent-nlu-forum/can-you-pass-live-agent-variables-contexts-from-virtual-agent-to/m-p/224261) — community forum (third-party, not official); Interaction records, context_table/context_document, LiveAgent_* variables
- [Implement access control in Now Assist AI agents - ServiceNow](https://www.servicenow.com/docs/r/intelligent-experiences/aia-security-implementation.html) — ACL-based memory scoping, user identity, role masking

### Third-Party Memory/State Frameworks and Patterns

- [Session Handoff Pattern - Dakera AI](https://dakera.ai/patterns/session-handoff) — handoff artifact design, context transfer at escalation, importance scoring
- [I stopped treating AI memory as summaries. I now think in handoffs. - DEV Community](https://dev.to/woshiliyana/i-stopped-treating-ai-memory-as-summaries-i-now-think-in-handoffs-1gcb) — handoff vs. summary distinction, framing shifts, boundaries, corrections as memory
- [Handoff Skill: Structured Context Transfer Between Agent Sessions - AgentPatterns.ai](https://agentpatterns.ai/agent-design/handoff-skill-context-transfer/) — handoff document structure, next steps, prior decisions
- [Handoff Debt: The Rediscovery Cost When Coding Agents Take Over Interrupted Tasks - arXiv](https://doi.org/10.48550/arxiv.2606.02875) — handoff context impact on successor agent efficiency
- [Handoff - Encyclopedia of Agentic Coding Patterns](https://aipatternbook.com/handoff) — handoff pattern mechanics, forces, solution structure
- [Agent-to-Human Handoff Patterns: Designing Escalation That Doesn't Break - Zylos Research](https://zylos.ai/research/2026-04-03-agent-to-human-handoff-patterns) — escalation signals, context packaging, failure modes; third-party design guidance
- [How to Build the Context Package for AI-to-Human Handoffs - Chanl Blog](https://www.channel.tel/blog/agent-human-handoff-context-package) — handoff context structure (identity, intent, summary, transcript, sentiment); third-party implementation pattern
- [Human-in-the-Loop Learning in a Production-Ready AI Agent - Luis Beqja](https://luisbeqja.com/human-in-the-loop-learning-in-a-production-ready-agent/) — missing-information acknowledgement pattern, merchant-initiated feedback, knowledge updates from human corrections
- [Building a self-improving agent on a context graph of human disagreement - Arize AI](https://arize.com/blog/self-improving-agent-with-context-graph/) — context graph construction from human override traces, mining disagreements for runtime config improvements
- [Multi-Tenant Primitives - Agentium Documentation](https://docs.agentium.in/features/multi-tenant) — ScopedStorage, AgentFactory, namespace transformation, scope hierarchy
- [Multi-User Isolation - Agentium Documentation](https://xhipment.mintlify.app/memory/isolation) — memory scope hierarchy (user → agent → tenant → global), default safety
- [AI agent memory scope: the namespace key, the five-axis rule file - PIAS](https://fde10x.com/t/ai-agent-memory-scope) — multi-tenant namespace rules, five axes, scope leak probes
- [agent-memory (reaatech GitHub)](https://github.com/reaatech/agent-memory) — long-term memory layer, extraction, decay, forgetting, contradiction resolution
- [Case Study: AI Customer Support Agent - Synthax Codes](https://synthax.codes/case-studies/ai-support-agent/) — feedback loop on responses for continuous improvement, confidence-based routing, human handoff with context

### Industrial IoT and Telemetry Patterns

- [Flash-Fusion: Enabling Expressive, Low-Latency Queries on IoT Sensor Streams with LLMs](https://ar5iv.labs.arxiv.org/html/2511.11885) — edge summarization, query planning, token usage reduction
- [RAG in Industrial IoT: How AI Agents Actually Reason About Telemetry - Cloud Studio IoT](https://cloudstudioiot.com/hub/rag-in-industrial-iot) — four retrieval sources (metadata, latest readings, historical, alerts), per-query retrieval pattern
- [ZeptoDB | Action-Outcome Memory for Physical AI](https://zeptodb.com/) — action-outcome episodes, sensor-evidence joining, context-gated recall for robotics
- [How to Build IoT Telemetry Storage with Dapr State Management - OneUptime](https://oneuptime.com/blog/post/2026-03-31-dapr-iot-telemetry-storage-state/view) — two-tier architecture (hot path state, cold path time-series), TTL on state

