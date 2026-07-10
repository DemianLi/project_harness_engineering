# Building an LLM Agent Harness in Python

This repo has no existing research-notes convention, so this file lives under `docs/research/` — a natural home for grounded investigation notes that inform future engineering work without being product documentation themselves.

Scope: how to build the kind of framework Claude Code, Codex CLI, and similar coding agents run on — the tool-use loop, context management, permission/safety gating, sandboxing, subagent orchestration, and observability — grounded in primary sources (official docs, SDK source code, specs).

## Search tooling notes

The repo's `anysearch` skill (`.claude/skills/anysearch/scripts/anysearch_cli.py`) was tried in a third research pass for claim verification but failed on most URLs (HTTP 404/403), consistent with documented sandbox proxy limitations. The pass instead relied on `WebSearch` (for discovering primary-source URLs) and targeted `WebFetch` (for reading documentation), plus direct `git clone --depth 1` of LangChain/LangGraph repositories to verify claim source code precisely. Key findings: OpenAI's harness-engineering and agent documentation returned HTTP 403 on direct `WebFetch` (pages blocked for read); several claims (OpenAI AGENTS.md convention, Nicholas Carlini project attribution, OpenAI mechanical-enforcement quote, Anthropic tool-vs-prompt numbers) could not be verified to the specificity required and were omitted from the document rather than added; Claude Code's memory system and Anthropic's three-agent architecture (Planner/Generator/Evaluator) were verified via official documentation and added with corrections where needed. A further claim — that LangChain has `PreCompletionChecklistMiddleware` and `LoopDetectionMiddleware` classes — was checked directly against a `git clone --depth 1` of `langchain-ai/langchain` (`libs/langchain_v1/langchain/agents/middleware/`); neither class name exists in the repo. The closest real analogs are `ToolCallLimitMiddleware` (a hard cap on tool calls, not an edit-count-based "reconsider your approach" nudge) and `TodoListMiddleware` (a plan/todo tracker, not a pre-exit checklist gating on whether tests ran) — since neither matches the claimed behavior, this claim was omitted rather than force-fit to an existing class.

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

### Claude Code's persistent memory hierarchy

Claude Code implements **two complementary memory systems** that load at the start of every session, documented at [How Claude remembers your project](https://code.claude.com/docs/en/memory):

1. **CLAUDE.md files** (user-written instructions): loaded in full at session start from multiple scopes — managed policy (org-level), user (`~/.claude/CLAUDE.md`), project (`./CLAUDE.md` or `./.claude/CLAUDE.md`), and local (`./CLAUDE.local.md`). Files are discovered by walking the directory tree and loaded in order from root to working directory, so instructions closer to the current directory are read last. No size limit on individual files, though shorter files (targeting under 200 lines per file) produce better adherence because they consume less context.

2. **Auto memory** (`MEMORY.md` + topic files): Claude accumulates notes in `~/.claude/projects/<project>/memory/`, starting with a `MEMORY.md` entrypoint plus optional topic-specific markdown files. **Only the first 200 lines of `MEMORY.md` (or 25KB, whichever comes first) are loaded at session start**; topic files are read on demand. This asymmetry allows Claude to maintain a concise index while deferring detailed notes to lazy-loaded topic files.

The design separates instructions (CLAUDE.md, fully loaded, prescriptive) from learnings (auto memory, first-200-lines summary loaded, descriptive).

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

Two layers found in Anthropic's public material: (1) model-level — Anthropic states that browser-agent attack success rates dropped "from double digits to approximately 1% with Opus 4.5," attributed on the source page to a combination of three approaches rather than any one alone: reinforcement learning ("We use reinforcement learning to build prompt injection robustness directly into Claude's capabilities"), improved classifiers, and scaled expert human red teaming ([Mitigating the risk of prompt injections in browser use](https://www.anthropic.com/research/prompt-injection-defenses)); the same page cautions that "A 1% attack success rate—while a significant improvement—still represents meaningful risk"; and (2) framework-level — the pattern, stated directly in Anthropic's own tool-use docs, of keeping all externally-sourced content ("web pages, inbound email, user uploads, third-party APIs") inside `tool_result` blocks rather than `system` or plain `text` blocks, specifically because the model is trained to treat instructions in those positions with lower trust ([Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls), warning callout). Claude Code's auto-mode classifier (above) is a concrete, harness-level instance of this same idea — it evaluates only user-authored content, not tool results, when deciding whether to approve an action.

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

### Anthropic's three-agent architecture for long-running applications

Anthropic has documented a production multi-agent pattern (in [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)) using three specialized agents for building complex systems over extended periods:

- **Planner**: Transforms a simple user prompt (1-4 sentences) into a comprehensive product specification, establishing scope and high-level technical design without implementation details.
- **Generator**: Implements the application based on the planner's spec, working in continuous development cycles against an explicit contract of what "done" means for each work phase.
- **Evaluator**: Functions as quality assurance, using **Playwright MCP** to "click through the running application the way a user would," testing UI features, API endpoints, and database states, and grading work against pre-agreed criteria.

**Sprint Contract pattern**: Before each sprint, the generator and evaluator negotiate a sprint contract agreeing on what "done" looks like for that chunk of work before any code is written — concrete, testable criteria (e.g. "User can reorder animation frames via API") rather than vague targets like "works well." Communication between the two agents runs entirely through files: one agent writes a file, the other reads it and replies either in that file or a new one the first agent reads in turn — no format (e.g. JSON) is specified for the contract itself. In one worked example, the evaluator's Playwright-driven check against that exact criterion failed with a concrete, file-traceable reason: "FAIL — `PUT /frames/reorder` route defined after `/{frame_id}` routes." This closes the feedback loop between implementation and verification without relying on the generator to evaluate its own output.

---

## 6. Guardrail and permission mechanisms in other frameworks

The core agentic loop and tool-calling architecture (sections 1-2) are largely universal across frameworks. Where they diverge significantly is in *how* they implement guardrails, approval gates, and blast-radius controls — the mechanisms that keep an agent from executing unauthorized, dangerous, or out-of-scope actions. This section compares three production-grade frameworks outside the Anthropic/OpenAI sphere: **CrewAI**, **AutoGen/AG2**, and **Google ADK**.

### CrewAI — hooks-based authorization and task-level human input

Primary sources: [CrewAI GitHub issues #4877, #6221](https://github.com/crewAIInc/crewAI), [CrewAI security best practices PR #4674](https://github.com/crewAIInc/crewAI/pull/4674), [Aport blog: CrewAI Guardrails in Production](https://aport.io/blog/crewai-guardrails-safe-ai-agents-kill-switch-audit-guide/).

**Approval/permission mechanism**: CrewAI's primary approval gate is task-level — a `Task` object supports `human_input=True`, which pauses execution and requires human approval before the task completes. This is simpler than fine-grained per-tool gates but operates at task granularity, not action granularity. Tool-level authorization is implemented via the `BeforeToolCallHook` protocol in `crewai.hooks.types`, which can return `False` to block execution before the tool runs.

**Tool-level guardrails**: CrewAI's guardrail system (`Task.guardrail` / `Task.guardrails`) validates *output* after task completion. For pre-execution controls, `BeforeToolCallHook` runs in the code path and can deterministically allow/deny a tool call; the framework is moving toward a standardized `GuardrailProvider` interface (referenced in issue #4877) that would let teams plug in policy engines (e.g., OPA, custom rule sets) without writing raw hooks.

**Human-in-the-loop mechanics**: The `human_input=True` flag on tasks is the primary HITL primitive — it pauses at task boundaries, not mid-tool, and does not natively offer run resumption from a serialized state checkpoint (though the underlying task transcript is queryable).

**Execution bounds**: CrewAI exposes `max_rpm` (requests per minute), `max_iter` (iteration limit), and `max_execution_time` for bounding agent execution, documented in the security best practices guide. There is no built-in sandboxing layer; containment relies on OS-level IAM for the tools themselves (e.g., a database tool runs as a service account with read-only permissions).

### AutoGen/AG2 — middleware-based tool access control and HITL hooks

Primary sources: [AG2 docs: Approval Required](https://docs.ag2.ai/latest/docs/beta/tools/approval_required/), [AG2 docs: Human in the Loop](https://docs.ag2.ai/latest/docs/user-guide/advanced-concepts/orchestration/group-chat/safeguards/), [AG2 docs: Safeguards](https://docs.ag2.ai/latest/docs/user-guide/advanced-concepts/orchestration/group-chat/safeguards/), [AG2 PR #2533: ToolPolicyMiddleware](https://github.com/ag2ai/ag2/pull/2533), [AG2 code example: Safety Guard](https://docs.ag2.ai/latest/docs/beta/code_examples/08_safety_guard/).

**Approval/permission mechanism**: AG2 uses a `hitl_hook` callback (human-in-the-loop hook) that receives user input for approval decisions. The `approval_required()` middleware decorates a tool to gate execution on explicit human approval; when the model tries to call a decorated tool, the agent prompts the user (via the configured `hitl_hook`) to approve or deny before the tool runs. If denied, the agent receives a structured error and can adjust its behavior.

**Tool-level guardrails**: AG2's `ToolPolicyMiddleware` implements allow/block lists at the tool invocation boundary, sitting on the `on_tool_execution` event. It checks tool names against a frozen policy (blocked tools override allowed tools, empty allowed list means "deny all"). For richer policy evaluation, `RegexGuardrail` and `LLMGuardrail` can detect patterns in tool inputs/outputs and apply masking or blocking. AG2 also supports policy-guided **safeguards** — a centralized policy file that specifies detection (regex or LLM-based) and actions (block or mask) for inter-agent messages and agent-tool/agent-LLM interactions; this sits above individual tool gates.

**Human-in-the-loop mechanics**: `human_input_mode` parameter on `ConversableAgent` takes values `ALWAYS` (always prompt for input), `TERMINATE` (prompt only when the agent wants to end), or `NEVER` (never prompt). The `approval_required()` middleware is the more granular HITL gate. Like CrewAI, run state is queryable but not natively resumable from a checkpoint.

**Execution bounds and containment**: AG2's `BaseObserver` + event system allows custom observers (e.g., `PathGuardian` watching `ToolCallEvent`) to emit alerts (FATAL, ERROR, WARNING) that trigger policy-based responses; a FATAL alert can halt the agent and short-circuit the next LLM call via auto-wired `_HaltCheckMiddleware`. This is a framework-level abort mechanism rather than OS-level sandboxing.

### Google ADK — tool context policies and pre-execution callbacks

Primary sources: [ADK docs: Safety and Security for AI Agents](https://adk.dev/safety/), [ADK issue #4965: GuardrailProvider protocol RFC](https://github.com/google/adk-python/issues/4965), [ADK issue #6099: Decision Ledger for tool-call authority and refusal](https://github.com/google/adk-python/issues/6099).

**Approval/permission mechanism**: ADK's core approval primitive is the `before_tool_callback` hook, which fires before tool execution and can validate or reject the call. Callbacks receive the tool call context and return a decision; if the callback blocks a call, the agent receives a structured error. For multi-tool scenarios, callbacks can check parameters against a **Tool Context** — a dictionary of developer-set policy data passed to each tool at invocation time (e.g., allowed tables for a query tool, allowed file paths for a write tool). Tool Context is deterministic (developer-controlled) and is the foundation of ADK's "in-tool guardrails" pattern.

**Tool-level guardrails**: ADK's principle is that tools should be "designed defensively" — expose only the actions you want the model to take. In-tool guardrails use Tool Context to enforce policies: a query tool can read an allowed-tables list from its Tool Context and reject queries that target other tables before executing them. This is purely deterministic (no LLM in the loop). For broader policy checking, callbacks and plugins can validate model and tool calls before or after execution. ADK also supports using a cheap model (e.g., Gemini Flash Lite) as a safety guardrail, screening inputs/outputs with a separate LLM call.

**Human-in-the-loop mechanics**: ADK has no single decorator-style primitive analogous to CrewAI's `human_input=True` or AG2's `approval_required()`; instead, HITL is composed from two pieces: a `before_tool_callback` that returns a blocking decision when approval is required, plus ADK's **resumability** mechanism (`ResumabilityConfig(is_resumable=True)`) — the run is checkpointed via Events/Event Actions after each completed step, and can later be resumed from that checkpoint by invocation ID (via the `/run_sse` REST endpoint or `Runner.run_async(invocation_id=...)`), which is how a paused-for-approval run is continued once a human decision comes back. ADK's docs flag that tools execute "at least once" under resume, so tools must guard against duplicate side effects (e.g. duplicate purchases) if a step reruns.

**Execution bounds and containment**: ADK emphasizes **identity and authorization**: tools can run as either the agent's own identity (agent-auth, simple but less fine-grained) or the end user's identity (user-auth, via OAuth). This is OS/external-system level control (the database, file system, or API enforces what the agent can do based on its identity). ADK also supports **sandboxed code execution** (preventing model-generated code from causing security issues) and **network controls** (VPC Service Controls to confine agent activity within secure perimeters). There is also a proposal for a **decision ledger** (issue #6099) to persistently record every tool-call decision, authority, refusal reason, and approval — the issue argues this fills a gap not currently addressed by ADK's existing observability surfaces (though this is the proposal's own framing, not independently verified against other frameworks).

### Comparison table

| Dimension | CrewAI | AG2 | Google ADK |
|---|---|---|---|
| **Approval gate** | `human_input=True` on Task (task-level) | `approval_required()` middleware (per-tool); `hitl_hook` callback | `before_tool_callback` block + resumability checkpoint/resume-by-`invocation_id` (no single decorator, but composable into a pause-for-approval flow) |
| **Permission model** | `BeforeToolCallHook` → planned `GuardrailProvider` | `ToolPolicyMiddleware` (allow/block lists); `RegexGuardrail` / `LLMGuardrail` | Tool Context policies (deterministic); `before_tool_callback` for validation |
| **Tool-level control** | Output validation + pre-execution hook | Middleware chain at execution boundary | In-tool guardrails + callback validation |
| **Execution halt** | Task pause (not mid-tool) | `BaseObserver` → FATAL alert → `_HaltCheckMiddleware` | Callback-driven; no built-in halt primitive |
| **Run resumption** | Transcript queryable; no documented state-serialization/resume primitive found | Event stream queryable; no documented state-serialization/resume primitive found | **Yes** — `ResumabilityConfig(is_resumable=True)`, checkpointed via Events/Event Actions, resumed by `invocation_id` |
| **Execution bounds** | `max_rpm`, `max_iter`, `max_execution_time` | Observer-based alerts and policy evaluation | Identity/auth (agent-auth vs user-auth); VPC-SC |
| **Sandboxing** | OS-level IAM on tools | Observer/policy halt; no OS-level sandbox | Sandboxed code execution; network isolation |

---

## 7. Testing and observability

### Evals

Anthropic's guidance is at [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents). The post frames agent evaluation as harder than single-turn LLM evals because "agents operate over many turns: calling tools, modifying state, and adapting based on intermediate results," and because "mistakes can propagate and compound" — the post's own example is Opus 4.5 "solving" a τ2-bench flight-booking problem by finding a policy loophole, which scored as a failure against the written eval despite arguably serving the user better. Importantly, the post emphasizes that when evaluating "an agent," you are "evaluating the harness and the model working together," which means harness configuration materially affects outcomes. Anthropic has separately published (in the Opus 4.6 system card, per search results) that harness configuration alone — independent of the underlying model — can move SWE-bench-style scores by several points, illustrating why eval numbers are not comparable across differently-built harnesses. The open-source **SWE-bench** harness (maintained by Princeton) is the closest thing to a standard for coding-agent evaluation and is used by Anthropic, Cursor, and OpenAI's Codex CLI teams to report comparable(-ish) numbers, per search-indexed reporting.

### Tracing/observability

The most concretely documented implementation is OpenAI's Agents SDK, read directly from `docs/tracing.md` in the cloned repo:

- Tracing is **on by default**; disable via `OPENAI_AGENTS_DISABLE_TRACING=1`, `set_tracing_disabled(True)`, or per-run `RunConfig.tracing_disabled`.
- A **Trace** represents one end-to-end `Runner.run()`/`run_sync()`/`run_streamed()` call (or a larger block if you wrap several runs in a `with trace("name"):` context); it's composed of **Spans**, each with `started_at`/`ended_at`, a `parent_id`, and typed `span_data` (`AgentSpanData`, `GenerationSpanData`, etc.). By default, agent runs, LLM generations, function-tool calls, guardrails, and handoffs are each auto-wrapped in their own span type (`agent_span()`, `generation_span()`, `function_span()`, `guardrail_span()`, `handoff_span()`).
- The current trace/span is tracked via a Python `contextvar`, so it composes correctly under `asyncio` concurrency without explicit propagation.
- Architecture: a global `TraceProvider` is configured with a `BatchTraceProcessor` that exports to a `BackendSpanExporter` (OpenAI's dashboard) in batches; `add_trace_processor()` adds an additional sink (e.g. Datadog, Arize, Langfuse per the SDK's third-party integration list) without replacing the default, while `set_trace_processors()` replaces it entirely.
- Sensitive-data handling is explicit and opt-out: `generation_span()`/`function_span()` capture full LLM and tool call inputs/outputs by default; `RunConfig.trace_include_sensitive_data` (or the `OPENAI_AGENTS_TRACE_INCLUDE_SENSITIVE_DATA` env var) turns that off.
- For short-lived worker processes (Celery/RQ/FastAPI background tasks), the doc recommends an explicit `flush_traces()` after the `trace()` context exits, since the default batch exporter runs on a background timer that may not fire before process exit.

Claude Code's equivalent is OpenTelemetry-based rather than a bespoke tracing UI: set `CLAUDE_CODE_ENABLE_TELEMETRY=1` plus `OTEL_METRICS_EXPORTER`/`OTEL_LOGS_EXPORTER` (`otlp`, `prometheus`, or `console`) to export metrics (session count, lines of code, pull requests, commits, cost, token usage by type including cache reads/writes, code edit tool decisions, and active time) and events/logs via the standard OTel protocols to any OTel-compatible backend (Grafana, SigNoz, etc.) — [Monitoring](https://code.claude.com/docs/en/monitoring-usage). Sensitive data handling here is opt-in, the reverse of the OpenAI SDK's default above: `OTEL_LOG_USER_PROMPTS`, `OTEL_LOG_TOOL_DETAILS`, and `OTEL_LOG_TOOL_CONTENT` are each disabled by default, so prompt text, tool parameters, and tool input/output content are excluded from spans/events unless explicitly enabled. This is complementary to, not a replacement for, the hooks mechanism from §4 — hooks fire deterministically on tool events and can themselves log/forward data to a separate observability pipeline.

### ChatGPT browser Developer mode: CDP access for inspection, not agent self-verification

ChatGPT's browser (used by Computer Use and the built-in `@Browser`/`@Chrome` tools) has an opt-in **Developer mode** that gives the agent "controlled access to the Chrome DevTools Protocol (CDP)... to profile JavaScript, inspect console output and network traffic, examine the DOM and applied styles, or diagnose an issue in the live browser" ([Browser — ChatGPT developer docs](https://learn.chatgpt.com/docs/browser?surface=app), reached via redirect from `developers.openai.com/codex/app/browser`). This is enabled per-user via Settings → Browser → Developer mode → "Enable full CDP access," and can be disabled org-wide by an admin setting `browser_use_full_cdp_access = false`. Note this is debugging/inspection access in the consumer ChatGPT app, not documentation of a coding-agent runtime capturing DOM snapshots or screenshots for self-verification — that stronger framing was not found on this or any page checked.

---

## 8. Cross-cutting view: a six-layer harness taxonomy

A separate write-up frames harness engineering as six layers — information boundary, tool system, execution orchestration, memory/state, evaluation/observability, and constraint/recovery. This section maps *only the material this document has verified against a primary source* onto that taxonomy, as a reading map back into §1–§7. Where the six-layer framing named a specific fact this document could not verify, that's noted as a gap rather than silently dropped — see the "Search tooling notes" section above for what was checked and excluded.

**L1 — Information boundary** (what an agent should/shouldn't know): Claude Code's two-tier memory system (§3, "Claude Code's persistent memory hierarchy") is the verified material here — `CLAUDE.md` files (four scopes: managed policy, user, project, local; loaded in full) versus auto memory (`MEMORY.md`, first 200 lines/25KB loaded at session start, topic files lazy-loaded on demand). *Gap: no verified material on an OpenAI `AGENTS.md`-as-short-index convention — see "Search tooling notes."*

**L2 — Tool system** (how the agent acts on the world): tool schema definition and execution/error-handling patterns across Anthropic and OpenAI (§2); subagent tool restriction (`tools`/`disallowedTools`) as a blast-radius control (§5); and the tool-level guardrail mechanisms compared across CrewAI, AG2, and Google ADK — `ToolPolicyMiddleware`, ADK's Tool Context policies, `BeforeToolCallHook` (§6). *Gap: no verified material on the specific "tool design over prompt tuning" numbers/framing — see "Search tooling notes."*

**L3 — Execution orchestration** (chaining multi-step work): the universal call-model → run-tools → repeat loop and how each framework implements it (§1); and, as the more specific verified case study, Anthropic's three-agent **Planner / Generator / Evaluator** architecture with **Sprint Contract** negotiation before each work phase (§5, "Anthropic's three-agent architecture for long-running applications").

**L4 — Memory/state externalization** (managing state across a long task): prompt caching and server-side context editing/compaction (§3); Claude Code's memory hierarchy is relevant here too, since it's literally state externalized to the filesystem rather than kept in-process (§3). *Gap: the claimed Nicholas Carlini compiler-project case study and the general "filesystem as memory" OS analogy were not independently confirmed to the standard this document otherwise holds — see "Search tooling notes" for what was and wasn't checked; treat their omission as unresolved rather than refuted.*

**L5 — Evaluation/observability** (how the agent checks its own work): evals framed as harness-plus-model evaluation and SWE-bench as a de facto standard (§7); OpenAI Agents SDK tracing and Claude Code's OpenTelemetry metrics (§7); and, as the clearest verified example of an agent checking its *own* work against a spec, the Evaluator agent's Playwright-MCP-driven checks against the Sprint Contract (§5) — this is stronger evidence for the "agent self-verification" idea than the ChatGPT browser material. *Confirmed directly this pass:* ChatGPT's browser Developer mode does give CDP access (§7), but the source page describes human-facing debugging tooling in a consumer app, not an agent runtime self-verifying via DOM snapshots — and a "made 2x faster" performance claim checked against that same page does not appear there at all.

**L6 — Constraint, validation, and recovery** (what happens when something goes wrong or out of bounds): Claude Code's permission modes, auto-mode classifier, and hooks (§4); sandboxing (§4); and the approval/guardrail/execution-halt mechanisms compared across CrewAI, AG2, and Google ADK, including AG2's `BaseObserver` → `_HaltCheckMiddleware` halt chain (§6). *Gap, no OpenAI quote located — see "Search tooling notes." Confirmed directly this pass:* a claim that LangChain has `PreCompletionChecklistMiddleware` and `LoopDetectionMiddleware` classes was checked against a full clone of `langchain-ai/langchain` and refuted — neither class exists there; the closest real analogs, `ToolCallLimitMiddleware` and `TodoListMiddleware`, don't match the claimed checklist/loop-detection behavior.

---

## Primary sources referenced

- [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps) — Anthropic's three-agent architecture (Planner, Generator, Evaluator), Sprint Contract pattern, Playwright MCP integration
- [How Claude remembers your project](https://code.claude.com/docs/en/memory) — Claude Code's CLAUDE.md hierarchy, auto memory with 200-line threshold
- [Agent SDK reference — Python](https://platform.claude.com/docs/en/agent-sdk/python) / [`anthropics/claude-agent-sdk-python`](https://github.com/anthropics/claude-agent-sdk-python) (cloned and read: `query.py`, `client.py`, `_internal/transport/subprocess_cli.py`)
- [Tool Runner (SDK)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner)
- [Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)
- [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Context editing](https://platform.claude.com/docs/en/build-with-claude/context-editing)
- [Configure the sandboxed Bash tool](https://code.claude.com/docs/en/sandboxing)
- [Choose a permission mode](https://code.claude.com/docs/en/permission-modes)
- [Automate actions with hooks](https://code.claude.com/docs/en/hooks-guide)
- [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- [CrewAI GitHub repository](https://github.com/crewAIInc/crewAI) (issues #4877 "GuardrailProvider interface", #6221 "Deterministic tool permission gating"; PR #4674 "Security best practices guide")
- [CrewAI Guardrails in Production](https://aport.io/blog/crewai-guardrails-safe-ai-agents-kill-switch-audit-guide/) (Aport blog)
- [AG2 Approval Required](https://docs.ag2.ai/latest/docs/beta/tools/approval_required/)
- [AG2 Human in the Loop](https://docs.ag2.ai/latest/docs/user-guide/advanced-concepts/orchestration/group-chat/)
- [AG2 Safeguards — policy-guided safety](https://docs.ag2.ai/latest/docs/user-guide/advanced-concepts/orchestration/group-chat/safeguards/)
- [AG2 Safety Guard code example](https://docs.ag2.ai/latest/docs/beta/code_examples/08_safety_guard/)
- [AG2 GitHub PR #2533 — ToolPolicyMiddleware](https://github.com/ag2ai/ag2/pull/2533)
- [Google ADK: Safety and Security for AI Agents](https://adk.dev/safety/)
- [Google ADK: Resume Agents](https://adk.dev/runtime/resume/)
- [Google ADK GitHub issue #4965 — GuardrailProvider protocol](https://github.com/google/adk-python/issues/4965)
- [Google ADK GitHub issue #6099 — Decision Ledger](https://github.com/google/adk-python/issues/6099)
- [Mitigating the risk of prompt injections in browser use](https://www.anthropic.com/research/prompt-injection-defenses)
- [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [Monitoring — Claude Code Docs](https://code.claude.com/docs/en/monitoring-usage)
- [Browser — ChatGPT developer docs](https://learn.chatgpt.com/docs/browser?surface=app) (redirected from `developers.openai.com/codex/app/browser`) — Developer mode's Chrome DevTools Protocol (CDP) access
- [`openai/openai-agents-python`](https://github.com/openai/openai-agents-python) (cloned and read: `docs/running_agents.md`, `docs/tracing.md`, `docs/guardrails.md`, `docs/human_in_the_loop.md`)
- [`create_react_agent` reference — LangGraph](https://reference.langchain.com/python/langgraph.prebuilt/chat_agent_executor/create_react_agent) / [`langchain-ai/react-agent`](https://github.com/langchain-ai/react-agent)
