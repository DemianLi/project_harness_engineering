# Factory Equipment Monitoring Chat Agent: The Tool System Layer (L2)

This document investigates the tool/action system layer for end-user-facing factory equipment monitoring and crisis-handling chat agents. Unlike developer-configurable agent harnesses, this product architecture delivers all tool definitions and permissions as immutable builder-time decisions. The L2 layer covers how enterprises define what tools an agent can invoke, enforce parameter safety and type correctness, distinguish read from write operations, handle tool failures and retries safely, and scope tool access by user/role.

Scope: tool definition schema mechanics (OpenAPI, JSON Schema, function details); parameter typing and validation enforcement; tool-level access control (per-agent tool binding, role-based visibility); dangerous-action marking and approval gates; idempotency design for write operations; and structured parameter validation as an injection defense.

---

## Search tooling notes

This pass used `anysearch` and `WebSearch` to discover primary-source URLs, then targeted `WebFetch` of official documentation for AWS Bedrock Agents, Azure AI Foundry, Google Vertex AI ADK, Microsoft Copilot Studio, and ServiceNow. Primary sources include:

- **AWS Bedrock Agents**: OpenAPI schema definitions, function details API, action group configuration
- **Azure AI Foundry**: Function calling with strict mode, FunctionToolDefinition class reference
- **Google Vertex AI ADK**: Function tool definition, type hint inspection, require_confirmation mechanism
- **Microsoft Copilot Studio**: Agent instruction authoring, knowledge source binding
- **ServiceNow Now Assist**: ACL-based tool access control, user identity configuration, supervised execution mode
- **Anthropic Claude**: Strict tool use documentation, JSON Schema enforcement via grammar-constrained sampling
- **Stripe, AEP, IETF drafts**: Idempotency key patterns and HTTP standards
- **Enterprise security and integration architecture**: JSON Schema injection defenses, parameter validation patterns

Industrial platforms (Siemens, Honeywell, GE) and Microsoft Copilot Studio did not yield vendor-authored technical documentation on tool system mechanics beyond marketing material; those gaps are noted explicitly. Claims about tool-system mechanics for idempotency and parameter validation are grounded in enterprise API design standards and verified against multiple implementations.

---

## 1. Tool/action definition mechanics: schema formats and required fields

### AWS Bedrock Agents: OpenAPI or function-detail approaches

Per [Define OpenAPI schemas for your agent's action groups in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-api-schema.html), Bedrock agents can define action groups in two ways:

1. **OpenAPI schema** — a JSON or YAML file conforming to OpenAPI 3.0 (a subset, with documented limitations). Key fields in the schema:
   - `openapi` (required) — must be `"3.0.0"`
   - `paths` (required) — relative paths to endpoints, each starting with `/`
   - `method` (required) — HTTP method (GET, POST, PUT, DELETE, PATCH)
   - `description` (required per method) — describes when to call this operation
   - `operationId` (required for tool-use models) — unique alphanumeric string with hyphens/underscores only, identifying the operation
   - `parameters` (optional) — query/path/header parameters
   - `requestBody` (optional) — request body schema
   - `responses` (required) — expected response structure
   - `x-requireConfirmation` (optional) — ENABLED or DISABLED; defaults to DISABLED if omitted

   Per the documentation, the `enum` field for parameter constraints is **not supported**; allowed values must be described in the parameter's `description` field instead.

2. **Function details** — a simpler configuration where the builder specifies parameters the agent must elicit from the user. Per [Define function details for your agent's action groups in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-action-function.md), parameters are defined with name, type, and required/optional marking; the agent then requests these from the user and passes them to the builder's Lambda handler.

### Azure AI Foundry: function calling with `strict: true`

Per [Use function calling with Microsoft Foundry agents](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/function-calling) and the Python SDK reference (`FunctionToolDefinition`), Foundry agents use JSON Schema to define function tools. The schema includes:

- `name` (required) — function name
- `parameters` (required) — JSON Schema object describing the input
- `description` (required) — what the function does
- `strict` (optional) — boolean flag; when `strict: true`, the schema compliance is enforced via the underlying model

The documentation notes that `strict: true` requires `additionalProperties: false` in the schema and uses the model's schema-compliance guarantees to ensure inputs are type-valid.

### Google Vertex AI ADK: function tools with type hints and `require_confirmation`

Per [Function tools](https://adk.dev/tools-custom/function-tools/) and the Python source (`google/adk/tools/function_tool.py`), ADK derives tool definitions from Python function signatures. The mechanism is:

1. **Name and description** — derived from function name and docstring
2. **Parameters** — derived from type hints and parameter docstrings. A parameter is **required** if it has a type hint but no default value.
3. **Parameter description** — extracted from the function's docstring (e.g., `Args: city (str): The city name.`)
4. **require_confirmation** (optional) — a boolean or callable that determines whether user approval is needed before invoking the tool

The `require_confirmation` callable receives the function's arguments and returns `True` if approval is required for that invocation.

### ServiceNow: tool definition in AI Agent Studio

Per [Building AI Agents Guide - ServiceNow SDK](https://servicenow.github.io/sdk/guides/building-ai-agents-guide) and the [ServiceNow AiAgent API reference](https://servicenow.github.io/sdk/4.7.1/api/aiagent-api), tools are configured as part of an `AiAgent` definition via the Fluent API. Each tool specifies:

- Tool type (Script, CRUD, RAG, Web Search, etc.)
- Name and description
- Parameters (if applicable)
- Execution mode (Supervised or Unsupervised)

The documentation does not expose a publicly visible JSON Schema or parameter schema definition format; tool definitions are programmatic via the SDK.

---

## 2. Minimal-privilege tool exposure: scoping which tools an agent can invoke

The information boundary (L1) determines what data an agent can retrieve; the tool system layer (L2) determines which actions an agent can take with that data.

### AWS Bedrock: per-agent action group binding

Per [Retrieve data and generate AI responses with Amazon Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html), agents are created with an explicit list of action groups. Once an agent is defined, only the action groups attached to it are available; the agent cannot invoke action groups it was not configured with.

### Azure AI Foundry: tool definitions per agent

Per the Foundry agents documentation, tools are registered at agent creation time. An agent instance has a specific set of tools available to it.

### Google Vertex AI ADK: `require_confirmation` per tool instance

Per the ADK function tools documentation, `require_confirmation` is set per tool at definition time, but it operates at the invocation level — a tool can be available to multiple agents, and approval behavior is consistent across them.

### ServiceNow: ACLs at tool level

Per [Implement access control in Now Assist AI agents - ServiceNow](https://www.servicenow.com/docs/r/intelligent-experiences/aia-security-implementation.html), ServiceNow implements access control via:

- **Agent-level ACLs** — determine which users can invoke the agent
- **Tool-level ACLs** — determine which users can invoke specific tools within an agent (per the [ServiceNow Community post on access control enhancements](https://www.servicenow.com/community/now-assist-articles/latest-access-control-security-enhancements-for-ai-agents-and/ta-p/3374036))

The ACL checks apply to the user identity (either dynamic user or dedicated AI user) that the agent runs as.

---

## 3. Read vs. write / dangerous-action tool distinction

Enterprise systems (payments, equipment control, incident creation) require marking tools by the risk of their side effects. This is distinct from general permission gates — it is part of the tool's definition itself.

### Tool approval requirement as a built-in property

Multiple frameworks document an `approval` or `needsApproval` property on tool definitions:

- **Convex Agents** (`needsApproval`): Per [Tool Approval - Convex](https://docs.convex.dev/agents/tool-approval.md), a tool can specify `needsApproval: true` (always requires approval) or `needsApproval: async (ctx, input) => boolean` (approval depends on input). When approval is required, the agent returns a `tool-approval-request` instead of executing.

- **AI SDK (Vercel)** (`toolApproval`): Per [Tool Approvals - Vercel AI SDK](https://ai-sdk.dev/v7/docs/agents/tool-approvals), `toolApproval` can be per-tool (returning 'approved', 'denied', or 'user-approval') or a generic function checking the full tool call. Example:
  ```
  toolApproval: async ({ amount }, { runtimeContext }) => {
    if (runtimeContext.role !== 'admin') return { type: 'denied', reason: 'Only admins can send payments' };
    return amount > 1000 ? 'user-approval' : undefined;
  }
  ```

- **AG2** (`approval_required`): Per [Approval Required - AG2](https://docs.ag2.ai/latest/docs/beta/tools/approval_required/), a middleware decorator `@tool(middleware=[approval_required()])` gates a tool on human approval. The decorator accepts custom messages for the approval prompt and denial message.

- **Claude Agent SDK** (tool restrictions via blast radius): Per [Configure permissions - Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/permissions.md), subagents can be granted only specific tools, preventing them from invoking tool categories they should not access.

### Tier-based approval in Oh-My-Pi

Per [Tool approval mode - oh-my-pi](https://github.com/can1357/oh-my-pi/blob/d7383294/docs/approval-mode.md), tools can declare an `approval` tier: `read` (reads data or updates UI-only metadata), `write` (mutates state but not arbitrary code), or `exec` (executes code, shells out, drives browsers). Tools without an explicit tier default to `exec` — the docs call this "the safe default for unknown custom tools." The system then enforces approval based on the declared tier and the active approval *mode* — but the mode itself has the opposite default: `tools.approvalMode` defaults to `yolo`, which auto-approves `read`, `write`, and `exec` tiers alike with no prompts at all, unless a project or user explicitly configures a stricter mode (`always-ask` or `write`). So per-tool tier classification defaults conservative, but the overall system ships permissive by default at the mode level — a factory-agent deployment relying on this pattern would need to explicitly set a non-`yolo` mode to get any prompting at all.

### Anthropic's `x-requireConfirmation` in OpenAPI

Per AWS Bedrock's OpenAPI documentation above, the `x-requireConfirmation` field on an operation specifies whether user confirmation is required before invoking the action. This is documented as a defense against malicious prompt injections.

### Real-world industrial example: AegisFlow

Per the L1 factory harness document's description of [AegisFlow](https://github.com/bishalbera/aegisflow), Archestra's Tool Policies distinguish actions that are safe (e.g., `reduce_load`) from dangerous ones (e.g., `emergency_shutdown`). The policy engine blocks dangerous actions even if the agent requests them, treating the tool definition's safety classification as the authoritative boundary.

---

## 4. Idempotency design for action tools

Equipment-control and incident-creation actions must survive network retries and LLM retry loops without creating duplicate side effects (double-charged payments, two tickets for one incident, two alarm acknowledgments).

### Idempotency-key pattern: definition and deployment

Per [Designing robust and predictable APIs with idempotency - Stripe](https://stripe.com/blog/idempotency), idempotency is implemented by having the client generate a unique identifier for each intent and attaching it to the request. The server stores the result the first time it sees that key; on retry with the same key, the server returns the cached result instead of reprocessing.

Per the [IETF draft on the Idempotency-Key HTTP header](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header-07), the `Idempotency-Key` is an HTTP request header carrying a unique value generated by the client. The server ensures requests with the same key are processed at most once.

### Idempotency key generation for agent-invoked tools

Two independent sources converge on the same underlying rule — the key must be stable across every retry of the same logical action — via two different concrete techniques:

- [Chanl](https://www.channel.tel/blog/idempotent-tool-calls-agent-retry-safety) derives the key by hashing a payload of conversation ID, tool-call ID, tool name, and arguments (shown below) — so any change in what's being called produces a different key, while a pure retry with identical inputs reproduces the same one.
- [how2](https://how2.sh/posts/how-to-implement-agent-retry-and-idempotency-for-tool-calls/) instead generates stable IDs once "at the start of the run and attach[es] them to every tool call" as an explicit field, rather than deriving a key by hashing.

Both agree on the same negative rule: do **not** include timestamps or random values in the key, since those would change on retry and defeat deduplication.

Actual example from the Chanl blog (TypeScript), quoted verbatim rather than translated:
```typescript
const payload = JSON.stringify({
  conversationId,
  toolCallId,
  toolName,
  args,
});
return createHash("sha256").update(payload).digest("hex").slice(0, 32);
```
The same post states the rule directly: "Don't include timestamps or random values in the key. If you do, a retry generates a different key, which defeats the entire purpose."

### Deduplication window and storage

Per [The API Idempotency Threat Model - Secure Patterns](https://newsletter.securepatterns.dev/p/designing-api-idempotency-keys-to-prevent-duplicate-writes) and [AEP-155 — Idempotency](https://aep.dev/aep-2026/155/) — a vendor-neutral API standard adapted from, but not officially part of, Google's own API Improvement Proposals (AIP) — the idempotency key and its result must be stored durably for at least as long as a client might retry. AEP-155 states plainly: "APIs should honor idempotency keys for at least an hour."

Storage options:
- **Database-backed** (preferred for safety-critical operations) — key and result stored in the same transaction as the business logic, ensuring atomicity
- **Cache-backed** (performance trade-off) — key and result stored in a separate cache (Redis, DynamoDB) with TTL management; accepts higher risk of duplicate during failover/replication lag

### Retry policy constraints

Per how2's guidance, retries must preserve:
- Same tool allowlist (do not expand to new tools on retry)
- Same or lower budget (do not increase iteration limit)
- Same idempotency key for write operations

---

## 5. Structured parameter validation as injection defense

Strict parameter schema enforcement prevents an agent from being manipulated into invoking a tool with attacker-injected or hallucinated parameters.

### Grammar-constrained sampling: Anthropic's strict tool use

Per [Strict tool use - Anthropic](https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use), setting `strict: true` on a tool definition guarantees Claude's tool inputs match the JSON Schema by constraining the model's token sampling to only valid outputs (grammar-constrained sampling).

Key mechanics:
- Tool inputs **must** conform to the schema structure (correct types, required fields present, no extra fields)
- Enums, patterns, and property constraints are enforced at generation time
- Without strict mode, the model may emit `"2"` (string) where the schema specifies `integer`, or omit required fields

This is an architectural defense: the constraint operates at the token level, before the tool is executed.

### JSON Schema injection vulnerabilities

Per [LLM Structured Output Security - System Hardening](https://www.systemshardening.com/articles/ai-landscape/llm-structured-output-security/), strict schema compliance does not prevent parameter value injection attacks. The threat model includes:

1. **Hallucinated parameter values** — the model invents a value (e.g., a nonexistent user ID) that matches the schema type but is factually wrong
2. **Semantically wrong values** — the schema-valid values exist but refer to the wrong resource (e.g., a transfer between two accounts is executed with the accounts swapped)
3. **Nested JSON injection** — the model includes attacker-provided JSON structure verbatim in a string field
4. **Schema description injection** — a schema field description is constructed from user input, allowing an attacker to inject instructions into the schema itself

The cure for these is **application-level validation after tool dispatch**:
- Verify that hallucinated IDs actually exist in your system
- Verify semantic correctness (the recipient account is what the user intended, not swapped)
- Validate nested structures are expected
- Never construct schema descriptions from user-controlled input

### Prompt injection via retrieved content: the dispatch-layer defense

Per [Prompt Injection in Tool-Calling Agents - CyberDevTech](https://www.cyberdevtech.com/articles/prompt-injection-tool-calling-agents-pydantic-fix), the real injection vector for agents is not direct user input but data the agent legitimately retrieves and re-reads: a support ticket body, an API response, a PDF chunk from RAG, an email.

The defense belongs in the dispatch layer, not the system prompt:
- After the model returns a tool call, validate the tool name against a whitelist of callable tools
- For each parameter, validate it against the tool's schema using a strict validator (Pydantic, AJV, etc.)
- Only pass validated parameters to the tool implementation
- Log both the raw model output and the validation decision for audit purposes

Example from CyberDevTech: instead of passing `block.input` directly to the function, use Pydantic to parse and validate:
```python
from pydantic import BaseModel, Field
class DeleteFileArgs(BaseModel):
    path: str = Field(pattern=r"^[/a-zA-Z0-9._-]+$")  # whitelist safe paths
    
validated_args = DeleteFileArgs(**block.input)
delete_file(validated_args.path)  # safe because path is validated
```

### JSON Schema validation: built-in vs. application-level

Per [AJV in ServiceNow - Medium](https://medium.com/@gsandeep2010/ajv-in-servicenow-the-secret-weapon-for-bulletproof-integrations-b52ea91b0016), even after strict schema compliance from the model, application-level validation is necessary. ServiceNow developers use AJV (Another JSON Validator) in Scripted REST APIs to validate incoming payloads before business logic processes them — the same pattern applies to tool-call dispatch:

```
Request validation gate → JSON Schema validation → business logic → database
```

---

## 6. Comparison: tool system mechanisms across platforms

| Platform | Schema format | Parameter typing | Tool-level access control | Approval gate | Idempotency guidance | Parameter validation |
|---|---|---|---|---|---|---|
| **AWS Bedrock Agents** | OpenAPI 3.0 (subset) or function details | JSON type system; no enum support (describe in description field) | Per-agent action group binding | `x-requireConfirmation` in OpenAPI; optional | Not documented in verified sources checked | Not documented |
| **Azure AI Foundry** | JSON Schema with `strict: true` option | Type-safe via model schema compliance | Tool instance per agent | Not documented | Not documented | `strict: true` enforces schema conformance via model sampling |
| **Google Vertex AI ADK** | Function signature + type hints + docstring | Type hints (required = no default, optional = has default) | Tool availability per agent (not granular per user) | `require_confirmation` callable per tool | Not documented | Type hint inspection + docstring parsing by framework |
| **Microsoft Copilot Studio** | Not documented in verified sources | Not documented | Tool binding to agent at creation | Not documented | Not documented | Not documented |
| **ServiceNow Now Assist** | Programmatic via SDK (no schema exposed) | Not documented | ACL-based per tool; role-masked execution | Supervised execution mode (approval gate) | Not documented | AJV recommended for REST API payloads; not built-in to tool dispatch |
| **Anthropic Claude** | JSON Schema | Type system + `strict: true` for grammar-constrained sampling | Per-subagent tool allowlist/disallowlist | `require_confirmation` (manual hook-based) | Not explicitly documented; clients implement via idempotency headers on tool result blocks | `strict: true` for input conformance; application layer validates semantics |

---

## 7. Key gaps: areas where primary-source material was not found

- **Microsoft Copilot Studio tool system details** — only agent instruction authoring and knowledge source binding were documented in the sources checked; tool definitions, parameter schemas, and access control specifics were not found.
- **ServiceNow built-in idempotency support** — AJV and JSON validation are used by developers; no official ServiceNow documentation was found describing automatic idempotency key generation or deduplication for tool calls.
- **Industrial platforms (Siemens, Honeywell, GE)** — only marketing material mentioning "AI-assisted troubleshooting" was available; technical tool-system documentation not found in public sources.
- **Azure AI Foundry multi-tenant tool scoping** — tool per-user or per-role filtering is not documented in the verified sources; Azure's broader RBAC applies, but agent-specific tool-access control mechanics were not found.
- **Google Vertex AI crisis-mode tool restrictions** — per the L1 factory document, no documented pattern was found where an agent's tool set changes based on crisis/emergency detection.

---

## Observations

1. **Schema enforcement is structural, not semantic.** Strict mode (AWS, Azure, Anthropic, Google) guarantees the JSON shape is valid. It does not guarantee the values are true (an ID exists, an account is the intended recipient, a user has permission). Application-level validation after dispatch is mandatory for safety-critical operations.

2. **Idempotency requires stable keys and durable storage.** The key must survive the full retry loop (client retry, LLM re-planning, orchestrator fallback) — it cannot include timestamps or random values. Storage must persist for at least one hour and survive failovers. Database-backed storage is safer than cache-backed for high-risk operations.

3. **Tool-level approval and access control are orthogonal.** Approval gates (require approval before executing) and access control (this user/agent cannot invoke this tool at all) solve different problems and are both necessary. An approval gate slows a dangerous action; an access control denies it entirely.

4. **Parameter validation must happen at dispatch time, not in the schema.** Strict schema conformance is a prerequisite but not sufficient. The dispatch layer must re-validate that parameters are semantically correct (IDs exist, values have expected semantics) before passing them to implementation code.

5. **Industrial platforms claim equipment monitoring but document little.** Siemens, Honeywell, and GE market AI-assisted troubleshooting for factory environments, but public technical documentation on their tool-system architecture is unavailable. This is typical of proprietary industrial software, leaving a gap in verified best practices for that vertical.

---

## Primary sources referenced

### Tool/Action Definition Mechanics

- [Define OpenAPI schemas for your agent's action groups in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-api-schema.html) — OpenAPI 3.0 subset, required fields, operationId, x-requireConfirmation
- [Define function details for your agent's action groups in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-action-function.md) — function-detail alternative to OpenAPI
- [Use function calling with Microsoft Foundry agents](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/function-calling) — function calling with strict mode
- [FunctionToolDefinition class (Azure.AI.Agents.Persistent)](https://learn.microsoft.com/en-us/dotnet/api/azure.ai.agents.persistent.functiontooldefinition) — C# SDK reference for tool definition
- [Function tools — Google ADK](https://adk.dev/tools-custom/function-tools/) — type hint inspection, require_confirmation mechanism
- [src/google/adk/tools/function_tool.py](https://github.com/google/adk-python/blob/main/src/google/adk/tools/function_tool.py) — Python source verification of FunctionTool schema derivation

### Tool-Level Access Control

- [Retrieve data and generate AI responses with Amazon Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html) — per-agent action group binding
- [Implement access control in Now Assist AI agents - ServiceNow](https://www.servicenow.com/docs/r/intelligent-experiences/aia-security-implementation.html) — ACLs, user identity, role masking
- [Latest access control security enhancements for AI Agents and Skill Kit - ServiceNow Community](https://www.servicenow.com/community/now-assist-articles/latest-access-control-security-enhancements-for-ai-agents-and/ta-p/3374036) — tool-level ACLs, supervised execution mode, role masking
- [Building AI Agents Guide - ServiceNow SDK](https://servicenow.github.io/sdk/guides/building-ai-agents-guide) — agent and tool configuration
- [AiAgent (sn_aia_agent) - ServiceNow SDK](https://servicenow.github.io/sdk/4.7.1/api/aiagent-api) — AiAgent API reference

### Approval Gates and Dangerous-Action Marking

- [Tool Approval - Convex](https://docs.convex.dev/agents/tool-approval.md) — needsApproval boolean and conditional approval
- [Tool Approvals - Vercel AI SDK](https://ai-sdk.dev/v7/docs/agents/tool-approvals) — per-tool and generic toolApproval functions
- [Approval Required - AG2](https://docs.ag2.ai/latest/docs/beta/tools/approval_required/) — approval_required() middleware decorator
- [Tool approval mode - oh-my-pi](https://github.com/can1357/oh-my-pi/blob/d7383294/docs/approval-mode.md) — read/write/exec approval tiers
- [Configure permissions - Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/permissions.md) — subagent tool restriction as blast-radius control

### Idempotency Design

- [Designing robust and predictable APIs with idempotency - Stripe](https://stripe.com/blog/idempotency) — core idempotency-key pattern
- [The API Idempotency Threat Model - Secure Patterns](https://newsletter.securepatterns.dev/p/designing-api-idempotency-keys-to-prevent-duplicate-writes) — database-backed vs. cache-backed storage, golden path
- [AEP-155 — Idempotency](https://aep.dev/aep-2026/155/) — vendor-neutral API standard (adapted from, not part of, Google's own AIP); idempotency_key field spec, minimum one-hour retention
- [draft-ietf-httpapi-idempotency-key-header-07](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header/) — IETF HTTP header specification (October 2025)
- [How to Build Idempotent Tool Calls for AI Agents - Chanl](https://www.channel.tel/blog/idempotent-tool-calls-agent-retry-safety) — key generation for agent calls, three layers of retry risk
- [How to Implement Agent Retry and Idempotency for Tool Calls - how2](https://how2.sh/posts/how-to-implement-agent-retry-and-idempotency-for-tool-calls/) — run ID + call ID + tool name + args; when not to retry
- [Idempotent Tool Calls - OnceOnly](https://docs.onceonly.tech/how-to/idempotent-tools/) — patterns for database-record, resource-operation, transaction-ID idempotency keys
- [agentidemp-py](https://github.com/MukundaKatta/agentidemp-py) — SHA256 hex, UUIDv5, scoped idempotency key helpers for LLM agent calls

### Structured Parameter Validation and Injection Defenses

- [Strict tool use - Anthropic](https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use) — grammar-constrained sampling, JSON Schema enforcement
- [LLM Structured Output Security: JSON Schema Injection, Type Confusion, and Schema Enforcement](https://www.systemshardening.com/articles/ai-landscape/llm-structured-output-security/) — threat model: hallucinated values, semantic wrongness, nested injection, schema description injection
- [Why Strict JSON Mode Doesn't Stop Hallucinated Tool Calls - DEV Community](https://dev.to/gabrielanhaia/why-strict-json-mode-doesnt-stop-hallucinated-tool-calls-4na4) — schema compliance vs. semantic correctness; four shapes of failure; application-level validation necessity
- [Prompt Injection in Tool-Calling Agents: Python Fix - CyberDevTech](https://www.cyberdevtech.com/articles/prompt-injection-tool-calling-agents-pydantic-fix) — indirect injection via retrieved content; dispatch-layer Pydantic validation example
- [Data-Structure Injection (DSI) in LLM Agents](https://github.com/Trivulzianus/Data-Structure-Injection) — completion attacks via semi-populated JSON/YAML structures; tool-hijack and tool-hack taxonomy
- [AJV in ServiceNow: The Secret Weapon for Bulletproof Integrations - Medium](https://medium.com/@gsandeep2010/ajv-in-servicenow-the-secret-weapon-for-bulletproof-integrations-b52ea91b0016) — JSON Schema validation pattern in Scripted REST APIs; application-level schema checking

### Related/Foundational

- [Factory Equipment Monitoring Chat Agent: The Information Boundary Layer (L1)](./factory-agent-harness-l1-information-boundary.md) — L1 foundation for multi-tenant isolation, RAG scoping, AegisFlow example
- [Building an LLM Agent Harness in Python](./python-agent-harness-engineering.md) — §2 (tool/function calling), §4 (permission and safety), §6 (CrewAI/AG2/Google ADK guardrails)
