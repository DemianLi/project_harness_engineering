# Building an LLM Agent Harness in Python

This repo has no existing research-notes convention, so this file lives under `docs/research/` — a natural home for grounded investigation notes that inform future engineering work without being product documentation themselves.

Scope: how to build the kind of framework Claude Code, Codex CLI, and similar coding agents run on — the tool-use loop, context management, permission/safety gating, sandboxing, subagent orchestration, and observability — grounded in primary sources (official docs, SDK source code, specs).

## Search tooling notes

The repo's `anysearch` skill (`.claude/skills/anysearch/scripts/anysearch_cli.py`) was tried first, as instructed. Both the initial query and one retry failed with `Connection Error: Unable to reach the API endpoint.` — consistent with the documented caveat that the outbound proxy blocks `api.anysearch.com` in some sandboxed environments. No further retries were spent on it. All research below was done with `WebSearch`/`WebFetch`, plus `git clone --depth 1` of two primary-source repositories (`anthropics/claude-agent-sdk-python` and `openai/openai-agents-python`) into the scratchpad so that architecture claims about the Python SDKs could be checked against actual source rather than documentation prose. A handful of pages (`docs.claude.com` initial hit before its 302 redirect, `anthropic.com/engineering/...`, `openai.github.io/openai-agents-python/...`) returned HTTP 403 to `WebFetch`; where that happened, the equivalent information was sourced from the cloned repository's own `docs/*.md` files or from `WebSearch` result summaries, and is noted inline.

---

## 1. Core agentic loop architecture

The universal pattern across every framework surveyed is the same state machine: **call the model → inspect the response for tool-use requests → execute those tools → feed results back as a new message → call the model again → repeat until the model produces a final answer with no tool calls (or a turn/iteration limit is hit).** What differs between frameworks is *where* that loop is implemented and how much of it is exposed to the developer.

### Raw Anthropic Messages API — the loop is yours to write

At the lowest level, the Anthropic Python SDK's Messages API has no concept of a "loop" at all: each call returns a `Message` whose `content` may contain `tool_use` blocks, and the caller is responsible for executing them and sending a follow-up `messages.create()` call whose `messages` array appends the assistant turn plus a `user` turn containing `tool_result` blocks. This manual pattern, including exact formatting rules (`tool_result` blocks must immediately follow their `tool_use` counterparts; `tool_result` blocks must come before any text in the same user turn) is documented at [Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls).

### Anthropic's `tool_runner` (Python/TS/etc. SDK beta) — the loop lives in the SDK, in-process

The Python SDK ships a beta `client.beta.messages.tool_runner()` helper that owns the loop for you. Per [Tool Runner (SDK)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner), each iteration works like this:

1. The runner sends a request to the Messages API with its current state.
2. It yields the response `Message` to your loop body.
3. Your loop body runs (you may inspect/mutate state).
4. If you did not touch message history, and the message contains tool calls, the runner appends the assistant message plus tool results and continues; if no tool calls, the loop exits. If you did touch state (e.g. called `runner.append_messages(...)`), the runner defers to you entirely.

Tools are defined with a `@beta_tool` decorator that derives a JSON Schema from the Python function's type hints and docstring:

```python
@beta_tool
def get_weather(location: str, unit: str = "fahrenheit") -> str:
    """Get the current weather in a given location.

    Args:
        location: The city and state, e.g. San Francisco, CA
        unit: Temperature unit, either 'celsius' or 'fahrenheit'
    """
    return json.dumps({"temperature": "20°C", "condition": "Sunny"})

runner = client.beta.messages.tool_runner(
    model="claude-opus-4-8", max_tokens=1024,
    tools=[get_weather], messages=[{"role": "user", "content": "Weather in Paris?"}],
)
for message in runner:
    print(message)
```

`runner.until_done()` collects only the final message; `max_iterations` bounds the loop; a thrown tool exception is caught and converted to a `tool_result` with `is_error: true` automatically, with the Python SDK additionally logging the full traceback via the standard `logging` module. Client-side automatic **compaction** (summarizing history when token usage crosses a threshold) is supported by the Python, TypeScript, and Ruby runners but is deprecated in favor of server-side context editing (see §3).

### Claude Agent SDK (`claude-agent-sdk` for Python) — a thin RPC client around the CLI, not an in-process loop

This is an important architectural distinction the docs don't foreground: the Claude Agent SDK for Python is **not** a Python implementation of the agent loop. Reading the actual source (cloned from [`anthropics/claude-agent-sdk-python`](https://github.com/anthropics/claude-agent-sdk-python)) shows that `query()` (`src/claude_agent_sdk/query.py`) and `ClaudeSDKClient` (`src/claude_agent_sdk/client.py`) both delegate to an `InternalClient`, which in turn drives `SubprocessCLITransport` (`src/claude_agent_sdk/_internal/transport/subprocess_cli.py`). That transport locates and spawns the compiled `claude` CLI binary as a subprocess (`cmd = [self._cli_path, "--output-format", "stream-json", "--verbose", ...]`, `subprocess_cli.py:286`) and communicates over stdio using newline-delimited JSON (`--input-format stream-json`). The actual tool-use loop — calling the model, executing built-in tools (Read/Write/Bash/etc.), applying permission decisions — runs **inside that Node.js-based CLI process**, not in the Python code. The Python SDK's job is to serialize `ClaudeAgentOptions` into CLI flags/config, stream prompts in, and parse `stream-json` events back into typed `Message` objects (`_internal/message_parser.py`). `README.md` confirms this directly: "The Claude Code CLI is automatically bundled with the package." Python-side hooks (`can_use_tool`, `hooks=`) are Python callables that get bridged across that subprocess boundary (`client.py:82-234`) — i.e. the CLI calls back into your Python process mid-loop for permission decisions. This is architecturally very different from `tool_runner`, which runs entirely in-process.

### OpenAI Agents SDK (`openai-agents-python`) — `Runner.run()` loop, in-process

Cloned from [`openai/openai-agents-python`](https://github.com/openai/openai-agents-python); loop description quoted verbatim from `docs/running_agents.md`:

> 1. We call the LLM for the current agent, with the current input.
> 2. The LLM produces its output. (1) If the LLM returns a `final_output`, the loop ends. (2) If the LLM does a handoff, we update the current agent and input, and re-run the loop. (3) If the LLM produces tool calls, we run those tool calls, append the results, and re-run the loop.
> 3. If we exceed `max_turns`, we raise `MaxTurnsExceeded`. Pass `max_turns=None` to disable the turn limit.

Three entry points: `Runner.run()` (async), `Runner.run_sync()`, `Runner.run_streamed()`. "Final output" is defined precisely as: text output of the desired type with **no** tool calls present. Unresolved model behavior (malformed tool JSON, an unmatched tool name) raises `ModelBehaviorError` by default, or can be downgraded to a model-visible error via `RunConfig(tool_not_found_behavior="return_error_to_model")`. Handoffs (an agent redirecting the conversation to a different `Agent` object) and multi-agent orchestration are first-class parts of this same loop, unlike Anthropic's tool_runner/Claude Agent SDK where subagents are invoked as a tool.

### LangGraph — the loop as a graph, not a `while` statement

LangGraph's `create_react_agent` (in `langgraph.prebuilt`) builds a small cyclic graph rather than a Python loop: an `agent` node calls the LLM; if the resulting `AIMessage` has `tool_calls`, a conditional edge routes to a `ToolNode`, which executes each call and appends `ToolMessage` objects; control returns to `agent`; the cycle repeats until a response has no `tool_calls`. Source: [`create_react_agent` reference](https://reference.langchain.com/python/langgraph.prebuilt/chat_agent_executor/create_react_agent) and [`langgraph-ai/react-agent`](https://github.com/langchain-ai/react-agent). The practical difference from a bespoke `while` loop is that state transitions are checkpointable/resumable graph nodes, which is what LangGraph's durable-execution and human-in-the-loop features build on.

### Comparison

| Framework | Where the loop runs | Turn limit mechanism | Tool result format |
|---|---|---|---|
| Raw Messages API | Your code | You write it | `tool_result` content block in a `user` message |
| Anthropic `tool_runner` | In-process (Python/TS SDK) | `max_iterations` | Same as above, auto-appended |
| Claude Agent SDK (Python) | **Out-of-process**, inside the `claude` CLI subprocess | CLI-side (`max_turns` option) | Streamed back as parsed `Message` objects over stdio |
| OpenAI Agents SDK | In-process (`Runner`) | `max_turns` → `MaxTurnsExceeded` | `function_call_output` items, Responses API format |
| LangGraph | In-process, as a graph cycle | Graph recursion limit | `ToolMessage` in `messages` state list |

---

## 2. Tool/function calling implementation

**Schema definition.** Anthropic's raw API takes a JSON Schema per tool (`name`, `description`, `input_schema`); the Python SDK's `@beta_tool` decorator generates that schema from a typed function signature and Google-style docstring ([Tool Runner](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner)). `strict: true` tool definitions enable grammar-constrained sampling so inputs are guaranteed schema-valid (referenced in the tool-runner docs' Java example and [Strict tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use)). OpenAI's Agents SDK derives schemas from Python function signatures via `@function_tool` (`src/agents/function_schema.py` in the cloned repo).

**Parsing requests.** A Claude response with `stop_reason: "tool_use"` contains one or more `tool_use` blocks, each with `id`, `name`, `input` — [Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls) gives the exact JSON shape and the strict ordering/adjacency rules for the reply (`tool_result` blocks must immediately follow their `tool_use` counterpart and must precede any trailing text in the same turn, or the API returns a 400).

**Execution and error handling.** The consistent pattern across SDKs is: run the tool, catch exceptions, and turn failures into a `tool_result`/`function_call_output` with an error flag (`is_error: true` for Anthropic) rather than raising out of the loop — this lets the model see the failure and retry or recover. Anthropic's docs explicitly recommend *instructive* error text ("Rate limit exceeded. Retry after 60 seconds." rather than "failed") and note that on an invalid tool call (e.g. missing parameter), "Claude will retry 2-3 times with corrections before apologizing to the user" ([Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)). The tool_runner exposes explicit interception points (`generate_tool_call_response()` in Python) so callers can inspect a tool result's `is_error` flag before it reaches the model, e.g. to abort the loop instead of letting the model see the error. OpenAI's SDK has an analogous `tool_error_formatter` hook and a `ToolTimeoutError` raised when a tool exceeds a configured timeout (`docs/running_agents.md`).

**Retry strategy specifically for malformed tool calls** is handled by the *model*, not the harness, in Anthropic's design: an invalid call gets a `tool_result` describing the error and the conversation continues, relying on the model's own self-correction rather than harness-side retry logic. OpenAI's SDK instead gives the harness author a knob (`tool_not_found_behavior="return_error_to_model"`) to choose between raising or feeding the error back.

---

## 3. Context/memory engineering

### Prompt caching (Anthropic)

Primary source: [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching). Mechanics:

- A `cache_control: {"type": "ephemeral"}` block marks a **breakpoint**; everything from the start of the prompt (tools → system → messages, in that fixed order) up to and including that block is cached.
- Two TTLs: 5-minute default (refreshed free on every cache hit) or a 1-hour option at `{"type": "ephemeral", "ttl": "1h"}`.
- Pricing multipliers relative to base input tokens: 5-min cache write ×1.25, 1-hour cache write ×2.0, cache read (hit) ×0.1.
- Minimum cacheable prefix length varies by model (e.g. 1,024 tokens for Sonnet/Opus-class models, 2,048–4,096 for some Haiku variants); prompts below the minimum are silently sent uncached.
- Up to 4 explicit breakpoints per request; automatic caching (a single top-level `cache_control` field) uses one slot and auto-finds the longest matching prefix.
- A 20-block lookback window: the system searches backward from a breakpoint for a matching prefix hash; place the breakpoint on the *last unchanging block*, not on content that varies per-request (e.g. timestamps), or cache hits silently stop occurring.
- Changing tool definitions invalidates every cache level (tools/system/messages); changing only `tool_choice` or adding images invalidates system+messages but not tools.
- The `usage` response field distinguishes `cache_read_input_tokens`, `cache_creation_input_tokens`, and `input_tokens` (tokens strictly after the last breakpoint) so cost/latency can be attributed precisely.
- Tool results can also be marked with `cache_control` — the tool_runner docs show mutating the generated `tool_result` block to cache large tool outputs (e.g. document search results) before they're re-sent on the next turn.

### Server-side context editing / compaction (Anthropic)

Primary source: [Context editing](https://platform.claude.com/docs/en/build-with-claude/context-editing) (`clear_tool_uses_20250919` strategy, beta header `context-management-2025-06-27`). This runs server-side, before the prompt reaches the model, and the client keeps the full unmodified history:

```json
{
  "context_management": {
    "edits": [{
      "type": "clear_tool_uses_20250919",
      "trigger": {"type": "input_tokens", "value": 30000},
      "keep": {"type": "tool_uses", "value": 3},
      "clear_at_least": {"type": "input_tokens", "value": 5000},
      "exclude_tools": ["web_search"],
      "clear_tool_inputs": false
    }]
  }
}
```

Default trigger is 100,000 input tokens; default `keep` is the 3 most recent tool-use/result pairs. By default only *tool results* are cleared and replaced with a placeholder (tool call parameters remain visible), unless `clear_tool_inputs: true`. This is explicitly positioned as the successor to the SDK's client-side "compaction" (full-history summarization) — the tool_runner docs state that "the Python, TypeScript, and Ruby tool runners support automatic compaction... All three SDKs have deprecated this client-side option in favor of server-side context editing."

### System prompt / harness design principles

Cross-cutting principles visible across both vendors' docs: keep tool descriptions specific enough to avoid ambiguous invocations (Anthropic notes vague descriptions are the main cause of "invalid tool call" retries); treat all tool-returned content as untrusted and keep it inside `tool_result` blocks rather than the system prompt or plain text (see §4); and use `call_model_input_filter` (OpenAI) to trim/edit history immediately before each model call as a lower-level alternative to session-level compaction (`docs/running_agents.md`, "Hooks and customization" section).

---

## 4. Permission and safety engineering

### Sandboxing (Claude Code / Claude Agent SDK)

Primary source: [Configure the sandboxed Bash tool](https://code.claude.com/docs/en/sandboxing) and the companion engineering post (title indexed via search: "Making Claude Code more secure and autonomous with sandboxing," `anthropic.com/engineering/claude-code-sandboxing` — the page 403'd to `WebFetch`, so this section relies on the sandboxing doc page, which was fetched successfully and is the more detailed primary source anyway). Key facts, verified against the doc:

- **OS-level enforcement mechanisms differ by platform**: macOS uses the built-in **Seatbelt** framework (`sandbox-exec`); Linux and WSL2 use **bubblewrap** (`bwrap`), an unprivileged namespacing tool, plus `socat` to relay network traffic through a sandbox proxy. Native Windows is unsupported (WSL2 required).
- **Filesystem isolation**: by default a sandboxed Bash command can read almost anything but can only *write* to the working directory and a session temp dir; `sandbox.filesystem.allowWrite`/`denyRead`/`denyWrite`/`allowRead` configure exceptions, enforced by the OS (bubblewrap/Seatbelt), so they bind on all child processes, not just the direct command.
- **Network isolation**: no domains are pre-allowed; the first request to a new domain prompts once, after which it's approved for the session (or pre-declared via `allowedDomains`). The proxy makes allow/deny decisions on the **hostname only** — by default it does not terminate/inspect TLS, which the docs explicitly flag as a domain-fronting exfiltration risk for broad allowlist entries like `github.com`.
- **Credential protection** is a separate, opt-in layer (`sandbox.credentials`) that can `deny` (unset) or `mask` (swap for a per-session sentinel, substituted back only at the proxy for declared `injectHosts`) specific files/env vars — the sandbox's default read policy otherwise still permits reading `~/.aws/credentials` or `~/.ssh/`.
- **Escape hatch**: a command that fails purely because of sandbox restrictions can be retried with `dangerouslyDisableSandbox`, which routes it through the normal (non-sandboxed) permission flow instead — this can be disabled organization-wide via `allowUnsandboxedCommands: false`.
- Sandboxing is explicitly scoped to the Bash tool; Read/Edit/Write use the ordinary permission system, and computer-use runs on the real desktop with per-app OS prompts, not the sandbox.

### Permission gating layers

Three distinct, stackable mechanisms, all documented at [Permission modes](https://code.claude.com/docs/en/permission-modes):

1. **Permission modes** (`default`/manual, `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions`) set a baseline for which tool categories prompt at all.
2. **Auto mode's classifier**: a separate server-configured model reviews each pending action against a fixed, versioned rule list (things like `curl | bash`, force-push, `terraform destroy`, secret-manager writes, or "launching an autonomous agent loop... started with `--dangerously-skip-permissions`" are blocked by default) *before* it runs. The classifier only sees "user messages, tool calls, and CLAUDE.md content" — tool results are explicitly stripped from what the classifier sees, specifically so that hostile content embedded in a file or web page can't manipulate the classifier directly; a separate server-side probe scans tool results themselves for injected instructions. If the classifier blocks an action 3 times consecutively or 20 times total in a session, auto mode disables itself and falls back to interactive prompting.
3. **Hooks** (`PreToolUse`, `PostToolUse`, `Notification`, etc.) — deterministic, user-written shell commands invoked at lifecycle points, documented at [Automate actions with hooks](https://code.claude.com/docs/en/hooks-guide). A `PreToolUse` hook receives the tool call as JSON on stdin and can return a `permissionDecision` of `allow`/`deny`/`ask` (a `deny` from a hook overrides even `bypassPermissions`/`--dangerously-skip-permissions`); a non-JSON hook script can instead simply `exit 2` to block, with stderr surfaced back to the model as the denial reason (shown concretely in the doc's `protect-files.sh` example, which pattern-matches `.env`/`package-lock.json`/`.git/` and exits 2 to block edits).

`Protected paths` (e.g. `.git`, `~/.bashrc`, `.claude/`) are a fourth, orthogonal layer: writes to these are never auto-approved in any mode except `bypassPermissions`, regardless of allow rules.

OpenAI's Agents SDK has a structurally similar but simpler mechanism: **tool guardrails** run before/after a function-tool call and can block or rewrite it, while `needs_approval=True` (or a per-call async predicate) on a `@function_tool` pauses the run and surfaces a `ToolApprovalItem` in `RunResult.interruptions`, which the caller resolves via `state.approve(...)`/`state.reject(...)` before resuming from a serializable `RunState` (`docs/human_in_the_loop.md`, `docs/guardrails.md`).

### Prompt injection defenses

Two layers found in Anthropic's public material: (1) model-level — RL training that exposes Claude to simulated prompt injections, which Anthropic states "reduced attack success rates for browser agents from double digits to approximately 1% with Opus 4.5" (from `anthropic.com/research/prompt-injection-defenses`, retrieved via `WebSearch` summary since direct `WebFetch` 403'd — flagged as secondary-sourced); and (2) framework-level — the pattern, stated directly in Anthropic's own tool-use docs, of keeping all externally-sourced content ("web pages, inbound email, user uploads, third-party APIs") inside `tool_result` blocks rather than `system` or plain `text` blocks, specifically because the model is trained to treat instructions in those positions with lower trust ([Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls), warning callout). Claude Code's auto-mode classifier (above) is a concrete, harness-level instance of this same idea — it evaluates only user-authored content, not tool results, when deciding whether to approve an action.

---

## 5. Subagent orchestration

Primary source: [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents). Subagents are defined programmatically via an `agents={}` dict of `AgentDefinition` objects passed to `ClaudeAgentOptions`:

```python
agents={
    "code-reviewer": AgentDefinition(
        description="Expert code review specialist. Use for quality, security, and maintainability reviews.",
        prompt="You are a code review specialist...",
        tools=["Read", "Grep", "Glob"],   # tool restriction
        model="sonnet",                  # per-agent model override
    ),
}
```

Key mechanics from the doc:

- **Context isolation**: a subagent's context window starts empty; the *only* channel from parent to subagent is the `Agent` tool's prompt string. It does not see the parent's conversation history, tool results, or system prompt — only its own `AgentDefinition.prompt`, project `CLAUDE.md`, and its declared tool set.
- **Invocation**: subagents are invoked through a tool literally named `Agent` (renamed from `Task` in Claude Code v2.1.63; both names still appear depending on SDK version). Claude decides when to delegate based on each subagent's `description` field, or can be told explicitly ("Use the code-reviewer agent to...").
- **Parallelism**: multiple subagents can run concurrently; independent subtasks finish "in the time of the slowest one rather than the sum."
- **Background execution**: as of Claude Code v2.1.198, subagents run in the background *by default* — an `Agent` tool call that omits `run_in_background` launches non-blocking, and Claude explicitly sets `run_in_background: false` when it needs the result before continuing.
- **Tool restriction** via `tools`/`disallowedTools` is the primary blast-radius control (e.g. a `doc-reviewer` agent given only `Read`/`Grep`/`Glob` structurally cannot write files).
- **Nesting**: subagents can spawn their own subagents up to 5 levels deep (as of v2.1.172); a subagent can be prevented from spawning further ones by omitting `Agent` from its own `tools`.
- **Resumability**: a subagent can be resumed with its full transcript intact by capturing `session_id` + the `agentId:` trailer returned in the Agent tool result text, then re-issuing `query(resume=session_id, prompt=f"Resume agent {agent_id}...")`.
- **Scaling beyond a handful of agents**: for orchestrating "dozens to hundreds" of agents, the SDK recommends the separate `Workflow` tool, which "moves the orchestration into a script the runtime executes outside the conversation context" rather than turn-by-turn delegation.
- Auto mode's classifier (§4) explicitly extends into subagents: it evaluates the delegated task description *before* the subagent starts, checks each of the subagent's own actions during execution with the same rules as the parent, and reviews the subagent's full action history on completion, prepending a warning to the returned result if anything looks off.

OpenAI's Agents SDK takes a different default shape: multi-agent orchestration is built into the *same* run loop as a **handoff** (the LLM's response can redirect control to a different `Agent` object, updating "the current agent and input" and re-running the same loop — `docs/running_agents.md`) rather than a separate nested-conversation "subagent" call. `Agent.as_tool()` is the closer analog to Claude's subagent pattern — wrapping an agent so another agent can call it as a tool, including its own `needs_approval` gate — and its approvals still surface on the *outer* run's `RunState`, i.e. nested agent-as-tool calls don't get their own isolated approval flow.

LangGraph's equivalent is the general **multi-agent graph** pattern (agents as graph nodes, sometimes wrapped in a supervisor node) rather than a dedicated subagent primitive — this is architecturally consistent with LangGraph modeling the whole harness as a graph rather than a call stack, but wasn't a focus of this pass given the volume of Claude/OpenAI primary-source material available.

---

## 6. Testing and observability

### Evals

Anthropic's guidance is at [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (this page returned HTTP 403 to `WebFetch`; the summary below is sourced from `WebSearch`'s indexed excerpt of that page, not a direct read, and should be treated as lower-confidence than the rest of this document). It frames agent evaluation as harder than single-turn LLM evals because agents have "many possible trajectories to success or failure," and Anthropic has separately published (in the Opus 4.6 system card, per search results) that harness configuration alone — independent of the underlying model — can move SWE-bench-style scores by several points, illustrating why eval numbers are not comparable across differently-built harnesses. The open-source **SWE-bench** harness (maintained by Princeton) is the closest thing to a standard for coding-agent evaluation and is used by Anthropic, Cursor, and OpenAI's Codex CLI teams to report comparable(-ish) numbers, per search-indexed reporting.

### Tracing/observability

The most concretely documented implementation is OpenAI's Agents SDK, read directly from `docs/tracing.md` in the cloned repo:

- Tracing is **on by default**; disable via `OPENAI_AGENTS_DISABLE_TRACING=1`, `set_tracing_disabled(True)`, or per-run `RunConfig.tracing_disabled`.
- A **Trace** represents one end-to-end `Runner.run()`/`run_sync()`/`run_streamed()` call (or a larger block if you wrap several runs in a `with trace("name"):` context); it's composed of **Spans**, each with `started_at`/`ended_at`, a `parent_id`, and typed `span_data` (`AgentSpanData`, `GenerationSpanData`, etc.). By default, agent runs, LLM generations, function-tool calls, guardrails, and handoffs are each auto-wrapped in their own span type (`agent_span()`, `generation_span()`, `function_span()`, `guardrail_span()`, `handoff_span()`).
- The current trace/span is tracked via a Python `contextvar`, so it composes correctly under `asyncio` concurrency without explicit propagation.
- Architecture: a global `TraceProvider` is configured with a `BatchTraceProcessor` that exports to a `BackendSpanExporter` (OpenAI's dashboard) in batches; `add_trace_processor()` adds an additional sink (e.g. Datadog, Arize, Langfuse per the SDK's third-party integration list) without replacing the default, while `set_trace_processors()` replaces it entirely.
- Sensitive-data handling is explicit and opt-out: `generation_span()`/`function_span()` capture full LLM and tool call inputs/outputs by default; `RunConfig.trace_include_sensitive_data` (or the `OPENAI_AGENTS_TRACE_INCLUDE_SENSITIVE_DATA` env var) turns that off.
- For short-lived worker processes (Celery/RQ/FastAPI background tasks), the doc recommends an explicit `flush_traces()` after the `trace()` context exits, since the default batch exporter runs on a background timer that may not fire before process exit.

Claude Code's equivalent is OpenTelemetry-based rather than a bespoke tracing UI: set `CLAUDE_CODE_ENABLE_TELEMETRY=1` plus `OTEL_METRICS_EXPORTER`/`OTEL_LOGS_EXPORTER` (`otlp`, `prometheus`, or `console`) to export metrics (token usage/cost by user or model, cache hit rates, tool accept/reject rates, active-time-excluding-idle) and events/logs via the standard OTel protocols to any OTel-compatible backend (Grafana, SigNoz, etc.) — [Monitoring](https://code.claude.com/docs/en/monitoring-usage) (confirmed via `WebSearch` summary of the official doc; not independently `WebFetch`-verified in this pass). This is complementary to, not a replacement for, the hooks mechanism from §4 — hooks fire deterministically on tool events and can themselves log/forward data to a separate observability pipeline.

---

## Primary sources referenced

- [Agent SDK reference — Python](https://platform.claude.com/docs/en/agent-sdk/python) / [`anthropics/claude-agent-sdk-python`](https://github.com/anthropics/claude-agent-sdk-python) (cloned and read: `query.py`, `client.py`, `_internal/transport/subprocess_cli.py`)
- [Tool Runner (SDK)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner)
- [Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)
- [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Context editing](https://platform.claude.com/docs/en/build-with-claude/context-editing)
- [Configure the sandboxed Bash tool](https://code.claude.com/docs/en/sandboxing)
- [Choose a permission mode](https://code.claude.com/docs/en/permission-modes)
- [Automate actions with hooks](https://code.claude.com/docs/en/hooks-guide)
- [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- [Mitigating the risk of prompt injections in browser use](https://www.anthropic.com/research/prompt-injection-defenses) (search-indexed excerpt; direct fetch 403'd)
- [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (search-indexed excerpt; direct fetch 403'd)
- [Monitoring — Claude Code Docs](https://code.claude.com/docs/en/monitoring-usage) (search-indexed excerpt; direct fetch not attempted after initial 403 pattern)
- [`openai/openai-agents-python`](https://github.com/openai/openai-agents-python) (cloned and read: `docs/running_agents.md`, `docs/tracing.md`, `docs/guardrails.md`, `docs/human_in_the_loop.md`)
- [`create_react_agent` reference — LangGraph](https://reference.langchain.com/python/langgraph.prebuilt/chat_agent_executor/create_react_agent) / [`langchain-ai/react-agent`](https://github.com/langchain-ai/react-agent)
