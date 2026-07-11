# Factory Equipment Monitoring Chat Agent: The Information Boundary Layer (L1)

This document investigates the information boundary mechanisms for end-user-facing, non-configurable chat products that serve factory equipment monitoring and crisis-handling use cases. Unlike the developer-facing tools documented in the companion research files, this product architecture has no user-configurable harness (no CLAUDE.md, no AGENTS.md that end users write). Instead, all boundary decisions are made by product builders at deployment time and delivered through a chat-only interface.

Scope: how enterprise chat platforms (Azure AI Foundry, AWS Bedrock, Google Vertex AI, Microsoft Copilot Studio) implement system-level role/scope framing; retrieval/grounding scoping in RAG architectures; multi-tenant data isolation; real-time equipment telemetry injection patterns; and crisis/incident-mode information boundary changes.

---

## Search tooling notes

This pass used the `anysearch` skill for web search, WebSearch fallback, and targeted WebFetch of official documentation. Primary sources include:

- **Azure AI Foundry**: Official documentation on system message design, RAG/retrieval-augmented generation concepts, agentic retrieval, and Azure AI Search indexing
- **AWS Bedrock**: Knowledge Bases API documentation, multi-tenant RAG patterns (via AWS ML blog posts), metadata filtering, managed vs. customer-managed knowledge bases
- **Google Vertex AI**: Agent Search grounding documentation, system instructions for generative answers, ADK (Agent Development Kit) tool configuration
- **Microsoft Copilot Studio**: Agent instructions authoring, system topics, entities and slot filling, custom instructions in generative answers nodes
- **ServiceNow**: Now Assist AI agents access control documentation, ACLs, user identity/role masking, supervised execution modes
- **PagerDuty**: Escalation policies, dynamic routing, SRE Agent documentation
- **Siemens Industrial Copilot**: Press releases and marketing material (technical architecture details not found in verified primary sources)
- **Open-source frameworks**: AegisFlow (GitHub), Jeltz framework, Flash-Fusion research paper (arXiv), IoTAgents framework

Industrial platforms (Siemens, Honeywell Forge, GE Vernova, Schneider Electric EcoStruxure) returned marketing-only results without technical depth on information boundary mechanisms; no vendor-authored technical documentation on crisis-mode boundary changes was located for these platforms.

---

## 1. Hidden system-prompt / role-scoping design for enterprise chat agents

### Azure AI Foundry — system messages and agent instructions

Per [System message design for Azure OpenAI](https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/advanced-prompt-engineering), Azure OpenAI system messages are explicitly designed to "define the assistant's role and boundaries" and "set tone and communication style." The documentation notes that system messages influence the model but "do not guarantee compliance," and recommends "layering system messages with other mitigations (for example, filtering and evaluation)."

In the context of Foundry agents, system messages are authored by the *builder* and baked into the agent definition at deployment; they are not exposed to the end user. Foundry agents are created declaratively via agent configuration, per [Baseline Microsoft Foundry Chat Reference Architecture](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-chat) and [Basic Microsoft Foundry Chat Reference Architecture](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/basic-microsoft-foundry-chat). The agent receives "instructions in its system prompt" which define "what the agent's purpose is and what resources it should use."

### Microsoft Copilot Studio — agent instructions and system topics

Per [Write agent instructions - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/authoring-instructions), instructions are "the central directions and parameters an agent follows" and are authored by the product builder at agent creation or edit time. Instructions are written in plain text and can reference specific resources (tools, topics, knowledge sources) by typing "/" and selecting from a menu. The agent cannot act on instructions for tools or knowledge sources it doesn't have, so instructions must be "grounded in capabilities and knowledge you configured for your agent."

System topics (per [Use system topics - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/authoring-system-topics)) are "built into Copilot Studio and added to an agent automatically" and "help your agent respond to common events like escalation." System topics available include "Conversation Start," "Escalate," "End of Conversation," "Fallback," and others — each with built-in trigger logic that executes in response to defined events, independently of user-authored instructions.

### AWS Bedrock Agents — role definitions at agent creation

In AWS Bedrock Agents, the system-level instructions are not documented as explicitly as in Azure or Copilot Studio, but the [Query a knowledge base and retrieve data - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-retrieve.html) documentation shows that agents are configured with specific tool sets and knowledge base connections. Bedrock agents operate within guardrail configurations (per the Retrieve API documentation), which can include guardrail IDs and versions applied at retrieval time.

### Google Vertex AI — system instructions for generative answers

Per [Generate grounded answers with RAG - Google Cloud Documentation](https://docs.cloud.google.com/generative-ai-app-builder/docs/grounded-gen), "system instruction" is a required input: "A preamble to your prompt that governs the behavior of the model and modifies the output accordingly. For example, you can add a persona to the generated answer or instruct the model to format the output text a certain way." For multi-turn answer generation, "you must provide the system instructions for every turn." System instructions in Vertex AI are authored by the builder and embedded in the request structure; they are not user-modifiable.

---

## 2. Retrieval/grounding scoping (RAG) as the mechanism for information boundaries

The information boundary in enterprise chat products is enforced primarily through retrieval configuration, not system prompts alone. What the agent knows at any moment is determined by what data is retrieved into context before the model is called.

### Azure AI Foundry — agentic retrieval and hybrid search

Per [Retrieval augmented generation (RAG) and indexes in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/concepts/retrieval-augmented-generation), Azure distinguishes between "classic RAG" (single query) and "agentic retrieval" (also called "agentic RAG"):

- **Classic RAG** follows a three-step flow: retrieve relevant content from an index, augment the prompt with that content, and generate a response.
- **Agentic retrieval** uses "a model to break down complex inputs into multiple focused subqueries, run them in parallel, and return structured grounding data that works well with chat completion models." Agentic retrieval provides "context-aware query planning" (using conversation history to understand context) and "parallel execution" (running multiple focused subqueries simultaneously).

Per [Ground Your Agents Faster with Native Azure AI Search Indexing in Foundry](https://devblogs.microsoft.com/foundry/ground-your-agents-faster-native-azure-ai-search-indexing-foundry/) (a Foundry blog post by Farzad Sunavala, Principal Product Manager), Foundry now supports inline index creation directly from data sources (Azure Blob Storage, ADLS Gen2, Microsoft OneLake). The process is: select a data source, authorize, pick containers/paths, select an embedding model, and click "Create index & ingest" — Foundry then "pulls content → chunks documents → generates embeddings → provisions (or reuses) an Azure AI Search index optimized for hybrid queries."

Importantly, each agent is bound to specific indexes or knowledge sources at configuration time. The agent cannot retrieve from data sources that haven't been explicitly connected to it by the builder.

### AWS Bedrock — managed vs. customer-managed knowledge bases and metadata filtering

Per [Retrieve data and generate AI responses with Amazon Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html), Bedrock offers two types of knowledge bases:

- **Managed Knowledge Base** — Amazon Bedrock manages ingestion, indexing, storage, and retrieval. Includes "multi-modal data ingestion," "storage auto-scaling," "agentic retrieval for multi-hop reasoning," and "document-level permission filtering using Access Control Lists (ACLs) at retrieval time" (except for Web Crawler sources).
- **Customer-managed Knowledge Base** — the customer sets up and manages the vector store (Amazon OpenSearch Serverless, Aurora, Neptune) and ingestion pipeline.

For retrieval, per [Use agentic retrieval to query a knowledge base - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-agentic-retrieve.html), the `AgenticRetrieveStream` API breaks down complex queries into sub-queries, executes them against configured retrievers, and evaluates sufficiency of results. Agentic retrieval currently supports only managed knowledge bases.

Per [Query a knowledge base and retrieve data - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-retrieve.html), retrieval can be configured with metadata filtering using `managedSearchConfiguration` or `vectorSearchConfiguration`, allowing filtering of results by tenant or other metadata.

### Google Vertex AI Search & Conversation — data store binding and search scoping

Vertex AI documents two distinct grounding modes as separate features: "Grounding with Google Search" (opens the model to the public web — the opposite of a tight information boundary) and "Grounding with Agent Search" (scopes the model to the builder's own data stores). Only the latter is relevant to an information-boundary design. Per [Grounding with Agent Search](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/grounding/grounding-with-vertex-ai-search): "A maximum of 10 data stores is supported" per grounding configuration, and the documented flow requires the builder to explicitly select a data store ID before grounding can occur — an agent cannot retrieve from a data store that wasn't bound to it this way.

---

## 3. Multi-tenant / customer data isolation

Since a factory monitoring product might serve multiple factories or customers, data isolation is a critical information boundary concern. Real platforms document this explicitly.

### AWS Bedrock — three isolation models (silo, pool, bridge)

Per [Multi-tenant RAG data isolation for SaaS - Boundev](https://www.boundev.ai/blog/multi-tenant-rag-data-isolation-saas) (a vendor blog discussing RAG patterns across platforms), three common layouts emerge:

- **Silo**: one index or namespace per tenant. Strongest isolation (a query physically cannot reach another tenant's vectors), but operationally costly (many indexes, headroom per each).
- **Pool**: one shared index with tenant ID stored as metadata on every vector, filtered on each query. Cheapest and simplest, but every query depends on a correct filter.
- **Bridge**: a hybrid — pool for small tenants, dedicated silo for large or regulated ones.

AWS Bedrock's official documentation on this is at [Multi-tenancy in RAG applications in a single Amazon Bedrock knowledge base with metadata filtering](https://aws.amazon.com/blogs/machine-learning/multi-tenancy-in-rag-applications-in-a-single-amazon-bedrock-knowledge-base-with-metadata-filtering/). The post describes using "S3 folder structures and Amazon Bedrock Knowledge Bases metadata filtering to enable efficient data segmentation within a single knowledge base." The example shows a folder hierarchy: `s3://.../customer-data/customerA/policies/` and `s3://.../customer-data/customerB/policies/`, with metadata filtering applied at retrieval time to ensure tenant A's queries retrieve only from tenant A's folders.

Per [Multi-tenant RAG implementation with Amazon Bedrock and Amazon OpenSearch Service for SaaS using JWT](https://aws.amazon.com/blogs/machine-learning/multi-tenant-rag-implementation-with-amazon-bedrock-and-amazon-opensearch-service-for-saas-using-jwt/), the architecture combines Cognito (authentication), Lambda (request orchestration), DynamoDB (tenant mappings), and OpenSearch Service (vector store with fine-grained access control, FGAC). A pre-token-generation Lambda trigger adds tenant ID as a JWT attribute; that JWT is then used to map the requesting user to a tenant-specific FGAC role in OpenSearch. The post describes three concrete isolation patterns built on this: domain-level (a separate OpenSearch domain per tenant), index-level (shared domain, separate index per tenant), and document-level (shared index, query-time filtering by tenant attribute).

Per [Secure multi-tenant RAG with Amazon Bedrock and Verified Permissions](https://aws.amazon.com/blogs/architecture/secure-multi-tenant-rag-with-amazon-bedrock-and-verified-permissions/), AWS extends metadata filtering with "Amazon Verified Permissions," which externalizes authorization logic into Cedar policies. The pattern allows "a single RAG application to serve many departments while keeping each department's documents isolated, without standing up a knowledge base per team."

Separately, per a third-party vendor blog rather than an AWS source — [Multi-tenant RAG data isolation for SaaS — Boundev](https://www.boundev.ai/blog/multi-tenant-rag-data-isolation-saas) — states the underlying principle plainly, as a section heading: "Enforce the tenant boundary at the data layer, not the prompt," elaborating that "Language models are not an access-control layer" and that "the tenant id is derived server-side from the authenticated session or a signed JWT claim, never from a request body the client can edit." The same post enumerates common leak vectors verbatim: "A background re-embedding job that iterates all documents without re-applying the tenant scope. An evaluation or analytics query written by a different team that forgets the filter. A cache keyed on the question text but not the tenant id, so tenant B gets tenant A's cached answer. An admin or internal tool that reads across tenants for debugging and is left wired into production."

### ServiceNow — ACLs and role-based access control for agents

Per [Implement access control in Now Assist AI agents - ServiceNow](https://www.servicenow.com/docs/r/intelligent-experiences/aia-security-implementation.html), ServiceNow's security model comprises:

- **Access Control Lists (ACLs)** — determine which role(s) a user must have to invoke an agentic workflow or agent. ACLs are configured per workflow/agent/tool.
- **User identity** — determines which user the agent operates as during execution, and therefore what data it can access. Two options: "Dynamic user" (inherits invoking user's roles) or "AI user" (dedicated user with consistent roles).
- **Role masking** — (per [Latest access control security enhancements for AI Agents and Skill Kit - ServiceNow Community](https://www.servicenow.com/community/now-assist-articles/latest-access-control-security-enhancements-for-ai-agents-and/ta-p/3374036)) "minimizes the roles with which AI Agents can execute to a subset of the invoking user's roles." Role masking "applies to data access for agentic workflows, AI Agents, and Skills tool in that order."

The key principle is that the agent runs as a specific user (either the invoker or a dedicated AI user), and that user's roles determine what data the agent can access via ACL checks.

---

## 4. Real-time / telemetry data injection as a distinct information-boundary problem

Equipment monitoring agents need live sensor readings, alarm states, and status information. This is distinct from static knowledge bases and creates unique boundary challenges.

### Flash-Fusion — data reduction for IoT telemetry at the edge

[Flash-Fusion: Enabling Expressive, Low-Latency Queries on IoT Sensor Streams with LLMs](https://ar5iv.labs.arxiv.org/html/2511.11885) (arXiv 2511.11885, Georgia Tech, 2025) addresses the core problem: "Directly feeding all IoT telemetry to LLMs is impractical due to finite context windows, prohibitive token consumption (hundreds of millions of dollars at enterprise scales), and non-interactive latencies."

Flash-Fusion's approach:

1. **Edge-based statistical summarization** — reduces telemetry by 73.5% on a university bus fleet by summarizing sensor streams (averages, percentiles, anomalies) at the edge.
2. **Cloud-based query planning** — the cloud component parses the user's query to "determine the required analytical task, then selects the relevant data slices, and finally chooses the right representation before invoking an LLM."

The result: "95% latency reduction and 98% decrease in token usage and costs while maintaining high-quality responses."

Key observation: raw IoT data is not fed to the LLM at all. Instead, pre-processed, summarized features selected for the specific query are injected into the prompt.

### AegisFlow — anomaly detection triggering context injection

Per the [AegisFlow GitHub repository](https://github.com/bishalbera/aegisflow) (a custom-built example, not an official vendor platform), AegisFlow streams live sensor data from 9 industrial devices through MQTT into an anomaly detector. When anomalies are detected (using Z-score analysis on a sliding window), an AI agent diagnoses the root cause using MCP tools including RAG-based manual lookups and historical incident reports. Critically, "Archestra's LLM Proxy and Tool Policies act as safety guardrails, blocking dangerous commands like emergency shutdowns."

The data flow is: Sensor CSV → MQTT Simulator → Mosquitto Broker → Anomaly Detector → (on breach) → AI Agent with MCP Tools.

### Jeltz framework — declarative device binding via TOML profiles

Per [Jeltz GitHub repository](https://github.com/heath0xFF/jeltz), Jeltz "connects AI assistants to physical devices — sensors, actuators, cameras, and edge ML boards — via the Model Context Protocol (MCP)." Developers write a TOML profile describing their hardware; Jeltz generates an MCP server that exposes device readings as tools. The key insight: "A dashboard shows you numbers. Jeltz lets an LLM *reason about what they mean*" — by exposing sensors as tools (not dumping raw time-series into the prompt), agents can "correlate readings across sensors to identify shared root causes," "interpret anomalies," and "investigate multi-step failure scenarios."

### IoT data connector patterns — proprietary implementations

Per [The-Swarm-Corporation/IoTAgents GitHub](https://github.com/The-Swarm-Corporation/IoTAgents), open-source frameworks abstract IoT data integration by providing "IoT Data Connectors" for MQTT, CoAP, HTTP and other protocols, followed by "Data Normalization" to make disparate sensor formats uniform, and then "LLM Integration" to expose normalized data via tools.

**Not found in verified sources:** no official documentation from Siemens, Honeywell Forge, GE Vernova, or Schneider Electric was located that describes the specific mechanism by which live equipment telemetry is injected into their industrial copilot agents' context. Siemens' marketing material mentions "troubleshoot with Industrial Copilot" and "chat with Siemens Industrial Copilot" but does not detail whether telemetry injection is via RAG retrieval, tool calls, or static prompt inclusion.

---

## 5. Crisis/incident-mode information boundary changes

Is there a documented pattern where the information the agent has access to *changes* when a conversation is identified as a crisis/emergency/incident, as opposed to routine query handling?

### ServiceNow — escalation-triggered access and supervised execution

Per [Use system topics - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/authoring-system-topics), the "Escalate" system topic "Informs customers if they need to speak with a human. Triggers when 'talk to agent' is matched or the Escalate system event is called." This is a *redirection* to a human, not a change in the agent's information boundary.

Per [Latest access control security enhancements for AI Agents and Skill Kit - ServiceNow Community](https://www.servicenow.com/community/now-assist-articles/latest-access-control-security-enhancements-for-ai-agents-and/ta-p/3374036), ServiceNow supports **Supervised Execution mode** on tools: "You can use the Supervised mode to enhance security for agents with the capability to perform sensitive or critical actions. You can set the supervised execution mode when creating a tool in the AI agent guided setup." When a tool is in supervised mode, "human oversight" is required before the tool executes. This is an *approval gate*, not an information boundary change.

### PagerDuty — escalation policies and dynamic routing

Per [Escalation Policy Basics - PagerDuty](https://support.pagerduty.com/main/docs/escalation-policies), escalation policies automate notification of on-call users based on rules (initial responder, escalation timeout, repeat logic). Per [Meet Your Virtual Responder: PagerDuty's SRE Agent for AI-Driven Reliability - PagerDuty](https://www.pagerduty.com/blog/ai/meet-your-virtual-responder-pagerdutys-sre-agent-for-ai-driven-reliability/), the SRE Agent "connects directly to PagerDuty's event intelligence, on-call data, and service context." The agent "summarizes the situation, identifies potential root causes, and recommends next actions." However, the documentation does not describe whether the agent's context window or information boundary *changes* at different escalation levels.

Per [How Pro Services Routes Requests Based on Request Details - PagerDuty](https://www.pagerduty.com/ops-guides/using-pd/pro-services-team/), PagerDuty supports "dynamic escalation policy routing" — routing is "assigned based on the request details." This routes the *request* to different on-call users/teams, but does not explicitly describe the agent's information boundary changing based on escalation level.

### Crisis mode — not found in verified sources

No official documentation was found from Azure AI Foundry, AWS Bedrock, Google Vertex AI, Microsoft Copilot Studio, ServiceNow, PagerDuty, or the industrial platforms (Siemens, Honeywell, GE) describing a documented pattern where the *agent's information boundary* (what data it retrieves, what tools it can call, what system instructions apply) changes based on classification of a conversation as a crisis/emergency. 

Escalation mechanisms exist (routing to different humans, requiring approval, triggering notifications), but these operate at the *orchestration* layer (deciding who responds), not at the *information boundary* layer (what data the agent can see). The absence of this pattern in verified sources suggests either:
- The boundary does not change at crisis mode; instead, the same agent handles both routine and crisis queries with the same information scope, relying on its instructions and approval gates to govern behavior.
- Some platforms implement crisis-mode boundary changes, but this design is not documented in the public primary sources checked.

---

## Comparison: information boundary mechanisms across platforms

| Platform | System prompt / role-scoping | Retrieval scoping | Multi-tenant isolation | Real-time data injection | Crisis-mode boundary |
|---|---|---|---|---|---|
| **Azure AI Foundry** | Builder-authored system messages; agentic instructions per agent | Agentic RAG with parallel subqueries; hybrid search (vector + keyword); indexes bound to agent at config time | Not documented in verified sources checked | Not documented in verified sources; implied via tool calls | Not documented |
| **AWS Bedrock** | System-level guardrails + agent config | Managed/customer-managed knowledge bases; metadata filtering; agentic retrieval (managed only) | Silo/Pool/Bridge patterns; tenant ID enforcement at data layer (server-side JWT, not client params); metadata filters on every query | Not documented in verified sources | Not documented |
| **Google Vertex AI** | System instructions per request (required); persona/formatting guidance | Agent Search data stores (up to 10 per agent); dynamic retrieval option for Google Search | Not documented in verified sources checked | Not documented in verified sources | Not documented |
| **Microsoft Copilot Studio** | Agent instructions (plain text); system topics for common events | Knowledge sources connected to agent at creation; generative answers node with custom instructions | Not documented in verified sources checked | Not documented in verified sources | Escalate system topic redirects to human; no documented boundary change |
| **ServiceNow Now Assist** | Agent instructions via authoring UI | Not documented (ITSM focused) | ACL-based access control; user identity (dynamic or AI user); role masking | Not documented in verified sources | Supervised execution mode (approval gate); ACL-based field-level access (e.g., hiding sensitive fields from AI agents) |
| **Industrial platforms (Siemens, Honeywell, GE)** | Marketing material only; technical details not found in public sources | Marketing material only | Not found | Not found | Not found |

---

## Primary sources referenced

### Enterprise Chat Platforms

- [System message design for Azure OpenAI - Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/advanced-prompt-engineering) — role definition, scope boundaries, safety constraints
- [Baseline Microsoft Foundry Chat Reference Architecture](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-chat) — agent system prompt, tool orchestration, reference architecture
- [Basic Microsoft Foundry Chat Reference Architecture](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/basic-microsoft-foundry-chat) — agent configuration, instructions, tool usage
- [Retrieval augmented generation (RAG) and indexes in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/concepts/retrieval-augmented-generation) — classic vs. agentic RAG, context-aware query planning, parallel execution
- [Ground Your Agents Faster with Native Azure AI Search Indexing in Foundry](https://devblogs.microsoft.com/foundry/ground-your-agents-faster-native-azure-ai-search-indexing-foundry/) — inline index creation, data source binding, chunking and embedding
- [Connect an Azure AI Search index to Foundry agents](https://github.com/MicrosoftDocs/azure-ai-docs/blob/main/articles/foundry/agents/how-to/tools/ai-search.md) — tool connection, citations, source attribution
- [Write agent instructions - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/authoring-instructions) — instruction authoring, resource references, capability grounding
- [Use system topics - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/authoring-system-topics) — built-in topics, escalation, fallback, end-of-conversation
- [Use entities and slot filling in agents - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/advanced-entities-slot-filling) — entity recognition, custom entities, slot filling for domain-specific information
- [Optimize prompts and topic configuration - Microsoft Copilot Studio](https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/optimize-prompts-topic-configuration) — custom instructions, knowledge base configuration, content moderation levels
- [Retrieve data and generate AI responses with Amazon Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html) — managed vs. customer-managed KB, multi-modal ingestion, permission filtering, smart parsing
- [Query a knowledge base and retrieve data - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-retrieve.html) — Retrieve API, metadata filtering, managed search configuration, reranking
- [Use agentic retrieval to query a knowledge base - Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-agentic-retrieve.html) — AgenticRetrieveStream API, query decomposition, multi-hop reasoning, result evaluation
- [Grounding with Agent Search - Gemini Enterprise Agent Platform](https://cloud.google.com/vertex-ai/generative-ai/docs/grounding/grounding-with-vertex-ai-search) — data store binding, 10-datasource limit, grounding mechanics
- [Generate grounded answers with RAG - Google Cloud Documentation](https://docs.cloud.google.com/generative-ai-app-builder/docs/grounded-gen) — system instructions, role/text specification, multi-turn generation
- [Grounding with Search - Agent Development Kit (ADK)](https://adk.dev/grounding/grounding_with_search/) — Vertex AI Search tool configuration, data store ID format, agent definition patterns

### Multi-Tenant Isolation & Access Control

- [Multi-tenancy in RAG applications in a single Amazon Bedrock knowledge base with metadata filtering](https://aws.amazon.com/blogs/machine-learning/multi-tenancy-in-rag-applications-in-a-single-amazon-bedrock-knowledge-base-with-metadata-filtering/) — S3 folder structures, metadata filtering, tenant data segregation
- [Multi-tenant RAG with Amazon Bedrock Knowledge Bases](https://aws.amazon.com/blogs/machine-learning/multi-tenant-rag-with-amazon-bedrock-knowledge-bases/) — tenant isolation patterns, data sources, ingestion pipelines, vector databases, access controls
- [Multi-tenant RAG implementation with Amazon Bedrock and Amazon OpenSearch Service for SaaS using JWT](https://aws.amazon.com/blogs/machine-learning/multi-tenant-rag-implementation-with-amazon-bedrock-and-amazon-opensearch-service-for-saas-using-jwt/) — tenant boundary enforcement at data layer, server-side JWT, fine-grained access control
- [Secure multi-tenant RAG with Amazon Bedrock and Verified Permissions](https://aws.amazon.com/blogs/architecture/secure-multi-tenant-rag-with-amazon-bedrock-and-verified-permissions/) — Cedar policies, runtime-evaluated authorization, defense-in-depth pattern
- [Multi-tenant RAG data isolation for SaaS - Boundev](https://www.boundev.ai/blog/multi-tenant-rag-data-isolation-saas) — silo/pool/bridge isolation models, leak test vectors, cache-based leaks, cross-tenant filter vulnerabilities
- [Implement access control in Now Assist AI agents - ServiceNow](https://www.servicenow.com/docs/r/intelligent-experiences/aia-security-implementation.html) — ACLs, user identity, role masking, dynamic vs. AI user execution
- [Latest access control security enhancements for AI Agents and Skill Kit - ServiceNow Community](https://www.servicenow.com/community/now-assist-articles/latest-access-control-security-enhancements-for-ai-agents-and/ta-p/3374036) — role-based access, supervised execution mode, role masking per layer, field-level access restrictions

### Real-Time Telemetry & IoT

- [Flash-Fusion: Enabling Expressive, Low-Latency Queries on IoT Sensor Streams with LLMs](https://ar5iv.labs.arxiv.org/html/2511.11885) — edge-based statistical summarization, cloud-based query planning, 73.5% data reduction, 95% latency improvement
- [AegisFlow GitHub Repository](https://github.com/bishalbera/aegisflow) — MQTT sensor streams, Z-score anomaly detection, MCP tools, LLM Proxy safety guardrails, tool policy enforcement
- [Jeltz Framework GitHub Repository](https://github.com/heath0xFF/jeltz) — TOML profiles for device description, MCP server generation, cross-device reasoning, anomaly interpretation
- [The-Swarm-Corporation/IoTAgents GitHub Repository](https://github.com/The-Swarm-Corporation/IoTAgents) — IoT data connectors (MQTT, CoAP, HTTP), data normalization, LLM integration, real-time processing
- [water-ai-industrial-automation GitHub Repository](https://github.com/chiragmandal/water-ai-industrial-automation) — pump telemetry monitoring, anomaly detection via ML model, MCP tool exposure, live dashboard, FastAPI streaming

### Incident Escalation & Crisis Handling

- [Escalation Policy Basics - PagerDuty](https://support.pagerduty.com/main/docs/escalation-policies) — automatic notification, escalation timeout, repeat policy configuration, dynamic assignment
- [Meet Your Virtual Responder: PagerDuty's SRE Agent for AI-Driven Reliability](https://www.pagerduty.com/blog/ai/meet-your-virtual-responder-pagerdutys-sre-agent-for-ai-driven-reliability/) — SRE Agent integration with event intelligence, on-call data, service context, context-aware incident summarization
- [How Pro Services Routes Requests Based on Request Details - PagerDuty](https://www.pagerduty.com/ops-guides/using-pd/pro-services-team/) — dynamic escalation policy routing, orchestration rules, service routes
- [Exercise 4: Restrict AI Agent Read Access to the Incident Caller Field - ServiceNow Knowledge 2026](https://servicenow-events-or-lab-guidebo.gitbook.io/knowledge-2026/knowledge-2026/lab6217-k26/exercise-4-restrict-ai-agent-read-access-to-the-incident-caller-field) — ACL field-level access control, "Deny Unless" decision type, AI Agent security attributes
- [Incident Access Restriction Using ACL and Custom Roles - ServiceNow Community](https://www.servicenow.com/community/developer-forum/incident-access-restriction-using-acl-and-custom-roles/m-p/3521613) — table-level ACLs, field-level ACLs, role-based incident visibility

### Industrial Platforms

- [Siemens introduces AI agents for industrial automation - Siemens Press Release](https://press.siemens.com/global/en/pressrelease/siemens-introduces-ai-agents-industrial-automation) — AI agents for automation, industrial use cases (marketing overview only; technical architecture not detailed)
- [Siemens Industrial Copilot - Siemens Insights](https://www.siemens.com/en-us/company/insights/generative-ai-industrial-copilot/) — troubleshooting, automation code generation, shop-floor support (marketing material; technical information boundary mechanisms not documented)

---

## Key gaps: areas where primary-source material was not found

- **Siemens, Honeywell Forge, GE Vernova, Schneider Electric EcoStruxure** — Only marketing and press material was available. No official technical documentation was found describing how these platforms scope agent information boundaries, manage real-time telemetry injection, or handle multi-tenant isolation.
- **Crisis-mode information boundary changes** — No platform documented a pattern where the agent's accessible information *changes* based on crisis/emergency/incident classification. Escalation exists (routing, approval gates), but not documented information scope changes.
- **Azure AI Foundry multi-tenant isolation** — The documentation checked does not explicitly address multi-tenant patterns (though Azure's broader ecosystem supports this via role-based access control and resource isolation).
- **Google Vertex AI multi-tenant isolation** — Not found in verified sources.
- **Real-time telemetry injection mechanisms in Siemens/Honeywell/GE** — No vendor documentation specifies how live equipment data enters the agent's context.

---

## Observations

1. **Retrieval is the primary information boundary mechanism**, not system prompts. What an agent knows is determined first by what data sources it's bound to, second by what metadata filtering applies at retrieval time, and third by system instructions on how to interpret retrieved content.

2. **Multi-tenant isolation is enforced at the data layer**, not the model layer. Tenant IDs must be derived server-side (from authenticated session or JWT), never from client-editable request parameters. Every retrieval call, cache key, and logging path must include the tenant filter.

3. **Real-time telemetry is not dumped into the prompt**. Instead, it is pre-processed at the edge (statistical summarization, anomaly detection), and only the relevant features/events are injected into the query context or exposed as tools.

4. **Supervised execution and approval gates** exist for high-risk actions, but these are distinct from information boundary controls. An approval gate slows down a dangerous action; a boundary control prevents certain data from being visible at all.

5. **Industrial platforms (Siemens, Honeywell, GE) claim AI-assisted troubleshooting and equipment monitoring**, but public technical documentation on their harness engineering is unavailable. This is characteristic of the industrial software market, where detailed architecture is often proprietary.

