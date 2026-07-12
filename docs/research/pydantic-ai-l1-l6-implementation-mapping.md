# Pydantic AI v2 (v2.9.0): Factory Equipment Monitoring Harness Implementation Mapping

**Research Date**: 2026-07-12  
**Pydantic AI Version Evaluated**: v2.9.0 (released 2026-07-10)  
**Context**: Maps six-layer factory equipment monitoring agent harness concerns (L1–L6) to specific Pydantic AI v2 classes, decorators, and parameters; identifies gaps requiring custom implementation.

---

## L1: Information Boundary (Data/Tool/System-Instructions Scoping)

| Concern | Pydantic AI Implementation | Citation |
|---------|--------------------------|----------|
| **Dependency injection for per-tenant data scoping** | `RunContext[DepsT]` parameterized type; pass `deps_type=MyDataclass` to Agent constructor. At runtime, `@agent.system_prompt` and `@agent.tool` functions accept `ctx: RunContext[MyDataclass]`, then access dependencies via `ctx.deps`. Different agent instances can receive different `deps` at run time via `agent.run(deps=deps_instance)`, enabling per-tenant isolation. | [Dependencies | Pydantic Docs](https://pydantic.dev/docs/ai/core-concepts/dependencies/) |
| **Dynamic/per-call system prompt injection** | `@agent.system_prompt` decorator: async function returning `str`, evaluated at runtime with access to `RunContext[DepsT]` for dependency-driven prompts. Unlike static instructions, system prompts are preserved in message history. Alternative: `@agent.instructions` decorator for run-specific instructions that exclude prior history when `message_history` provided. | [Agents \| Pydantic Docs](https://pydantic.dev/docs/ai/core-concepts/agent/) |
| **RAG-scoping / retrieval-boundary mechanism** | **No native support.** Pydantic AI provides an `Embedder` interface for multiple embedding providers (OpenAI, Google, Cohere, Bedrock, local models), but RAG architecture is builder's responsibility. Recommended pattern: build a `retrieve()` tool that queries an embedding store and passes results to the LLM. No built-in retrieval-scope enforcement or data-source binding per agent. | [Embeddings \| Pydantic Docs](https://pydantic.dev/docs/ai/guides/embeddings/) |

---

## L2: Tool System (Tool Definitions, Validation, Approval Gates, Minimal-Privilege Exposure)

| Concern | Pydantic AI Implementation | Citation |
|---------|--------------------------|----------|
| **Tool definition with schema validation** | Tools defined via `@agent.tool` or `@agent.tool_plain` decorators; schema extracted from function signature and docstring (Google, NumPy, Sphinx formats). Parameters extracted as non-`RunContext` function arguments; all parameter types must be JSON-serializable. Pydantic validates parameters before tool execution against the schema. | [Function Tools \| Pydantic Docs](https://pydantic.dev/docs/ai/tools-toolsets/tools/) |
| **Parameter validation and retry-on-failure** | Pydantic validates tool parameters before execution. On validation failure, `ValidationError` is converted to `RetryPromptPart` and sent to model for retry (automatic, no explicit exception needed). Per-tool retry budget via `Tool(max_retries=N)`, per-toolset via `FunctionToolset(max_retries=N)`, or agent-wide via `Agent(retries={'tools': N})`. Default: 1 retry if not specified. | [Advanced Tool Features \| Pydantic Docs](https://pydantic.dev/docs/ai/tools-toolsets/tools-advanced/) |
| **Minimal-privilege toolsets: per-agent tool binding** | `FunctionToolset` class collects functions as tools; different agents can receive different toolsets at construction (`Agent(toolsets=[...])`) or runtime (`agent.run(..., toolsets=[...])`). Toolsets can be composed via `CombinedToolset`, filtered via `.filtered(predicate)`, renamed via `.renamed(mapping)`, or prefixed via `.prefixed(prefix)`. Each agent instance isolates its toolset. | [Toolsets \| Pydantic Docs](https://pydantic.dev/docs/ai/tools-toolsets/toolsets/) |
| **Dangerous-action approval gate: requires_approval parameter** | Decorator parameter `requires_approval=True` on `@agent.tool`, `@agent.tool_plain`, `Tool()`, `FunctionToolset.tool`, or `FunctionToolset.add_function()`. Tool function can also check `ctx.tool_call_approved` (boolean) or raise `ApprovalRequired()` exception for conditional approval. When a tool requires approval, the model outputs `DeferredToolRequests` instead of executing; approval is handled via `HandleDeferredToolCalls` capability. | [Deferred Tools \| Pydantic Docs](https://pydantic.dev/docs/ai/tools-toolsets/deferred-tools/) |
| **Human-in-the-loop approval handling** | `HandleDeferredToolCalls` capability: registered with a handler function that receives `RunContext` and `DeferredToolRequests`, returns `DeferredToolResults` to approve/deny specific calls, or `None` to defer to next capability. Framework chains multiple handlers in order; first non-None response wins. | [Capabilities \| Pydantic Docs](https://pydantic.dev/docs/ai/core-concepts/capabilities/) |

---

## L3: Execution Orchestration (Multi-Turn Dialog, Slot-Filling, Human Handoff, Plan-Then-Execute)

| Concern | Pydantic AI Implementation | Citation |
|---------|--------------------------|----------|
| **Multi-step/multi-turn orchestration** | `pydantic-graph` library bundled with Pydantic AI: `GraphBuilder[StateT, DepsT, InputT, OutputT]` defines nodes and edges; nodes run concurrently by default. Nodes access state via `ctx.state`, dependencies via `ctx.deps`. State types are generic and type-safe. Execution controller via graph builder's edge definitions (sequential, parallel, conditional). | [Graph Overview \| Pydantic Docs](https://pydantic.dev/docs/ai/graph/graph/); [Graph Builder \| Pydantic Docs](https://pydantic.dev/docs/ai/graph/builder/) |
| **Multi-agent delegation** | Agents can be invoked as tools by other agents (via toolsets). No first-class multi-agent orchestration primitive; instead, use the graph layer to coordinate agent outputs or manually build multi-turn loops with `message_history`. | [pydantic-ai-evaluation.md](./pydantic-ai-evaluation.md), §4 |
| **Plan-then-execute pattern (Blueprint/ReWoo)** | **No native support.** Thinking capability enables chain-of-thought reasoning before action, but explicit plan-generation followed by plan-verification is not a built-in primitive. Hand-build via multi-turn loops: first turn generates plan, subsequent turns execute and verify. | [pydantic-ai-evaluation.md](./pydantic-ai-evaluation.md), §3 |
| **Slot-filling / clarifying-question orchestration** | **No native slot-filling engine.** Use multi-turn conversation with message history and explicit Question nodes in custom orchestration. For deterministic slot collection, build custom state machine in graph layer or via hooks (`@agent.instructions` can emit clarifying questions; model must recognize and respond). | Not found in verified sources |

---

## L4: Memory/State Externalization (Conversation History, Cross-Session Persistence, State Injection)

| Concern | Pydantic AI Implementation | Citation |
|---------|--------------------------|----------|
| **Session/conversation message history** | `result.all_messages()` returns all messages including prior runs; `result.new_messages()` returns only current-run messages. JSON variants: `all_messages_json()`, `new_messages_json()`. Messages are `ModelRequest` and `ModelResponse` objects with timestamps, run_id, conversation_id, and metadata. | [Messages and Chat History \| Pydantic Docs](https://pydantic.dev/docs/ai/core-concepts/message-history/) |
| **Message persistence and serialization** | `ModelMessagesTypeAdapter = TypeAdapter(list[ModelMessage])` exported by framework. Pattern: (1) `result.all_messages()`, (2) `.model_dump_json()` or `TypeAdapter.dump_json()`, (3) store to DB, (4) on resume, load and `TypeAdapter.validate_python()`, (5) pass to `agent.run(..., message_history=validated_messages)`. Security: use `sanitize_messages()` on untrusted sources to strip system prompts and unresolved tool calls. | [Messages and Chat History \| Pydantic Docs](https://pydantic.dev/docs/ai/core-concepts/message-history/) |
| **Cross-session / long-term memory** | **No native cross-session memory service.** Builder responsible for: (1) extracting insights from conversations manually or via LLM, (2) serializing to persistent store, (3) retrieving relevant memories on new conversations, (4) injecting into system prompt or dependencies. No built-in memory consolidation, TTL, or identity-scoped retrieval like AWS Bedrock AgentCore or Google Vertex AI Memory Bank. | Not found in verified sources |
| **State injection via dependencies** | `RunContext[DepsT]` provides access to dependency instances; common pattern is to inject a state/memory store object via `deps` and access it in tools or system prompts. Type-safe via generics. Mutable state shared across all tools/prompts in a run. | [Dependencies \| Pydantic Docs](https://pydantic.dev/docs/ai/core-concepts/dependencies/) |

---

## L5: Evaluation/Observability (Tracing, Evaluation Frameworks, Human Feedback Loops, Observability)

| Concern | Pydantic AI Implementation | Citation |
|---------|--------------------------|----------|
| **Evals framework: dataset-based evaluation** | `Dataset` class collects test cases ("a collection of test Cases designed for the evaluation of a specific task or function"). `Case` represents a single test scenario ("Task inputs, with optional expected outputs, metadata, and case-specific evaluators"). `Evaluator` base class; the official docs' own example implements a **synchronous** `def evaluate(self, ctx: EvaluatorContext[str, str]) -> float:` method (the framework's design also permits async evaluators, but the documented example pattern is sync — do not assume `async def` is the required signature). Built-in evaluators: `IsInstance` (checks type), and others in Built-in Evaluators reference. Add evaluators to dataset via `.add_evaluator()` or constructor `evaluators=[...]`. Run via `evaluate_sync`, which returns an `EvaluationReport` aggregating per-case and average scores. | [Evals \| Pydantic Docs](https://pydantic.dev/docs/ai/evals/evals/); [Built-in Evaluators \| Pydantic Docs](https://ai.pydantic.dev/evals/evaluators/built-in/) |
| **Custom evaluators and LLM-based judging** | Inherit `Evaluator`, implement `evaluate(ctx: EvaluatorContext[InputT, OutputT]) -> float` (sync per the documented example; async is reportedly also supported by the framework's design). Docs reference implementing custom LLM-judge evaluators but do not provide built-in judge class; builder constructs calls to LLM inside evaluator. | [Evals \| Pydantic Docs](https://pydantic.dev/docs/ai/evals/evals/) |
| **Tracing/telemetry with Logfire** | `logfire.configure()` (finds token from `.logfire` directory) and `logfire.instrument_pydantic_ai()` enable automatic tracing. Auto-emits: (1) one trace per agent run, (2) spans for model calls, (3) spans for tool execution, (4) metrics (token usage, cost, time-to-first-chunk). No code changes needed for this tracing. Optional: `logfire.instrument_httpx(capture_all=True)` to capture raw HTTP requests to model providers. | [Debugging & Monitoring with Pydantic Logfire \| Pydantic Docs](https://pydantic.dev/docs/ai/integrations/logfire/) |
| **OpenTelemetry compatibility** | Logfire is an opinionated wrapper around OpenTelemetry; Pydantic AI uses OTel internally. Builder can export to any OTEL backend by configuring OTEL_EXPORTER_OTLP_ENDPOINT and headers. Standard OTEL instrumentation (spans, metrics, logs) is available. | [Observability with Logfire and OpenTelemetry Collector \| Pydantic Docs](https://pydantic.dev/articles/logfire-opentelemetry-collector) |
| **Human feedback loops** | **No native feedback mechanism.** Builder implements: (1) expose feedback UI post-agent-response, (2) capture user feedback with trace_id linking to message history, (3) store feedback with metadata, (4) manually review and retrain/fine-tune. No built-in annotation, closed-loop improvement, or automated retraining pipeline. | Not found in verified sources |

---

## L6: Constraint Validation/Recovery (Output Guardrails, Input Injection Defense, Hard vs. Soft Constraints, Failure Recovery, Rollback)

| Concern | Pydantic AI Implementation | Citation |
|---------|--------------------------|----------|
| **Output validation and guardrails** | `@agent.output_validator` decorator: async function `async def validate(ctx: RunContext, output: OutputType) -> OutputType` runs after structured output validation but before final result. Raise `ModelRetry(message)` to send error back to model for retry (consumes one retry from output budget). Can check `ctx.partial_output` (bool) to detect streaming intermediate vs. final output. Set output retry budget via `Agent(retries={'output': N})`. | [Output \| Pydantic Docs](https://pydantic.dev/docs/ai/core-concepts/output/) |
| **Input validation and injection defense** | Tool parameter validation via Pydantic before execution (not after); invalid parameters trigger validation error sent to model for retry. For user input injection: use `sanitize_messages()` on untrusted message history (strips system prompts, non-HTTP file schemes, unresolved tool calls). No built-in content moderation or prompt-attack filtering; that is a gap. | [Advanced Tool Features \| Pydantic Docs](https://pydantic.dev/docs/ai/tools-toolsets/tools-advanced/); [Messages and Chat History \| Pydantic Docs](https://pydantic.dev/docs/ai/core-concepts/message-history/) |
| **Tool-output validation / indirect injection defense** | Tool functions should return JSON-serializable data (or Pydantic models). Custom validation: use `Tool(args_validator=validator_func)` to add validation after Pydantic but before execution. No built-in tool-output sanitization; builder responsible for parsing and validating tool results before passing to model. | [Advanced Tool Features \| Pydantic Docs](https://pydantic.dev/docs/ai/tools-toolsets/tools-advanced/) |
| **Error recovery and retry orchestration** | Hooks system: `@hooks.on.run_error`, `@hooks.on.model_request_error`, `@hooks.on.tool_validate_error`, `@hooks.on.tool_execute_error`, `@hooks.on.output_validate_error`. Each hook supports "raise-to-propagate, return-to-recover" semantics: raise original error to propagate unchanged, raise different exception to transform, or return a value to suppress error (recovery). Wrap entire run via `@hooks.on.run(wrapping_handler)` for transaction-like recovery. | [Hooks \| Pydantic Docs](https://pydantic.dev/docs/ai/core-concepts/hooks/) |
| **Idempotency / preventing duplicate side effects** | **No built-in idempotent-key support.** Optional: integrate Prefect 3.0 by wrapping the agent in `PrefectAgent(agent, tool_task_config=TaskConfig(...), tool_task_config_by_name={...})` — these are constructor parameters, not a decorator — which wraps tools as cached Prefect tasks; per the docs, "Tasks with identical inputs will not re-execute if their results are already cached." Otherwise, builder manually implements idempotency at tool level (e.g., check database before writing, use upsert instead of insert). | [Prefect \| Pydantic Docs](https://pydantic.dev/docs/ai/integrations/durable_execution/prefect/) |
| **Compensating actions / rollback for failed operations** | **No native support.** Durable execution integration via Restate: Restate records every step in a journal and replays on crash, skipping completed steps. This provides crash-safety but not semantic rollback. For true compensating actions, builder must implement: (1) tool that undoes previous action, (2) hook or orchestration layer that calls undo on recovery. | Not found in verified sources |

---

## Key Gaps: Native Support Not Found

### Gaps Requiring Custom Implementation (Per Layer)

**L1 (Information Boundary)**
- RAG-scoping / retrieval data-source binding: Builder must construct `retrieve()` tool and manage retrieval boundaries in tool logic.
- System-prompt templating for multi-tenant role masking: Use `@agent.instructions` with `RunContext[DepsT]` to inject tenant-specific instructions, but no built-in tenant-isolation directive.

**L4 (Memory/State)**
- Cross-session long-term memory service: No automatic memory extraction, consolidation, or TTL. Builder serializes messages manually and implements extraction/retrieval logic.
- Memory identity-scoping: Builder responsible for scoping memory retrieval to user/tenant via custom state injection.

**L5 (Evaluation/Observability)**
- Domain-specific evaluation templates for equipment-monitoring: Builder designs evaluators for task-specific metrics (e.g., "did agent diagnose root cause correctly?").
- Closed-loop human-feedback improvement: No built-in annotation UI, feedback routing, or model retraining pipeline.

**L6 (Constraint Validation/Recovery)**
- Content moderation / PII redaction: No built-in content filters (unlike AWS Bedrock Guardrails or Azure AI Foundry Guardrails). Builder layers custom filters in `@agent.output_validator` or hooks.
- Idempotency-key enforcement: No per-tool idempotent-key parameter. Optional: use Prefect for input-based caching, or builder manually checks idempotency in tool functions.
- Compensating transactions / rollback: No built-in saga orchestration. Builder constructs undo tools and error-recovery hooks for transactional cleanup.

---

## Summary: Strengths and Trade-offs

### Strengths (Aligned with L1–L6 Harness Concerns)

1. **L1 Dependency Injection**: `RunContext[DepsT]` is flexible for per-tenant data/tool/instruction scoping at the framework level.
2. **L2 Tool Validation**: Pydantic's automatic parameter validation + model-driven retry is robust for parameter safety.
3. **L2 Approval Gates**: `requires_approval=True` + `HandleDeferredToolCalls` capability provides explicit human-in-the-loop flow.
4. **L3 Orchestration**: `pydantic-graph` handles multi-step workflows natively with type-safe state and dependency injection.
5. **L4 Message History**: Built-in serialization and resumption via `message_history` parameter simplifies multi-turn conversations.
6. **L5 Evals**: Code-first evaluation framework with Logfire integration for observability.
7. **L6 Output Validation**: `@agent.output_validator` with `ModelRetry` enables semantic guardrails via custom logic.
8. **L6 Hooks**: Comprehensive error-recovery hooks at every lifecycle point for fine-grained control.

### Trade-offs (Gaps Requiring Custom Code)

1. **L1 RAG**: No retrieval-scope enforcement; builder responsible for data-boundary logic in tools.
2. **L4 Cross-Session Memory**: No managed memory service; builder serializes and persists messages manually.
3. **L5 Feedback Loop**: No closed-loop human annotation or retraining orchestration.
4. **L6 Content Filtering**: No built-in content moderation, PII redaction, or hallucination detection guardrails.
5. **L6 Idempotency**: No per-tool idempotent-key support; builder uses Prefect or manual checks.
6. **L6 Rollback**: No saga pattern or compensating-transaction framework; builder hand-codes undo flows.

---

## Primary sources referenced

- [Pydantic AI official documentation](https://pydantic.dev/docs/ai/overview/) — overview, core concepts, API reference
- [Dependencies | Pydantic Docs](https://pydantic.dev/docs/ai/core-concepts/dependencies/) — RunContext, deps_type, dependency injection pattern
- [Agents | Pydantic Docs](https://pydantic.dev/docs/ai/core-concepts/agent/) — @agent.system_prompt, @agent.instructions, Agent class
- [Function Tools | Pydantic Docs](https://pydantic.dev/docs/ai/tools-toolsets/tools/) — tool definition, schema extraction, parameter types
- [Advanced Tool Features | Pydantic Docs](https://pydantic.dev/docs/ai/tools-toolsets/tools-advanced/) — parameter validation, args_validator, retry logic
- [Toolsets | Pydantic Docs](https://pydantic.dev/docs/ai/tools-toolsets/toolsets/) — FunctionToolset, CombinedToolset, composing toolsets, per-agent binding
- [Deferred Tools | Pydantic Docs](https://pydantic.dev/docs/ai/tools-toolsets/deferred-tools/) — requires_approval parameter, DeferredToolRequests, ApprovalRequired exception
- [Capabilities | Pydantic Docs](https://pydantic.dev/docs/ai/core-concepts/capabilities/) — HandleDeferredToolCalls capability, other native capabilities
- [Graph Overview | Pydantic Docs](https://pydantic.dev/docs/ai/graph/graph/) — pydantic-graph library, graph-based orchestration
- [Graph Builder | Pydantic Docs](https://pydantic.dev/docs/ai/graph/builder/) — GraphBuilder API, nodes, edges, state management
- [Embeddings | Pydantic Docs](https://pydantic.dev/docs/ai/guides/embeddings/) — Embedder interface, embedding providers, RAG pattern (builder-implemented)
- [Messages and Chat History | Pydantic Docs](https://pydantic.dev/docs/ai/core-concepts/message-history/) — all_messages(), new_messages(), ModelMessagesTypeAdapter, sanitize_messages()
- [Evals | Pydantic Docs](https://pydantic.dev/docs/ai/evals/evals/) — Dataset, Case, Evaluator base class, EvaluationReport
- [Built-in Evaluators | Pydantic Docs](https://ai.pydantic.dev/evals/evaluators/built-in/) — IsInstance and other built-in evaluator types
- [Debugging & Monitoring with Pydantic Logfire | Pydantic Docs](https://pydantic.dev/docs/ai/integrations/logfire/) — logfire.configure(), logfire.instrument_pydantic_ai(), auto-traced spans and metrics
- [Observability with Logfire and OpenTelemetry Collector | Pydantic Docs](https://pydantic.dev/articles/logfire-opentelemetry-collector) — OTEL compatibility, export configuration
- [Output | Pydantic Docs](https://pydantic.dev/docs/ai/core-concepts/output/) — @agent.output_validator, ModelRetry, output retry budget
- [Hooks | Pydantic Docs](https://pydantic.dev/docs/ai/core-concepts/hooks/) — @hooks.on.run_error, @hooks.on.model_request_error, error recovery hooks
- [Prefect | Pydantic Docs](https://pydantic.dev/docs/ai/integrations/durable_execution/prefect/) — Prefect 3.0 integration, idempotency via task caching
- [pydantic-ai-evaluation.md](./pydantic-ai-evaluation.md) — prior research on maturity, tool validation, retry loops, graph orchestration, model provider support
