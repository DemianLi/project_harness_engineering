# LLM Coding-Agent Harness: The Information Boundary Layer (L1)

This document investigates how different coding-agent tools decide what an agent should and shouldn't know at any given moment — via role/goal framing, context trimming/pruning strategy, and persistent instruction file organization. It complements the harness engineering overview in `python-agent-harness-engineering.md` (§3, §5), focusing specifically on the information boundary mechanisms available across tools.

Scope: instruction-file conventions (CLAUDE.md, AGENTS.md, .cursorrules, .windsurfrules, .aiderignore, .github/copilot-instructions.md), directory-level scoping and nesting, explicit size/scope guidance from official documentation, and runtime context-selection mechanisms (semantic retrieval, indexing, file filtering).

## Search tooling notes

This pass used `WebSearch` to discover primary-source URLs, then `WebFetch` to read official docs (agents.md spec site, Cursor's codebase indexing and engineering blog, Windsurf/Cascade's documentation, GitHub Copilot custom instructions, Aider's usage/repo-map/config guides), and `gh cli`/`git clone` to directly fetch GitHub repository contents (`openai/codex/AGENTS.md`, 322 lines verbatim; `cline/cline`'s source tree for its file-context-tracking implementation). Where an official source did not state a detail (e.g. Cursor's vector-database backend, Windsurf's index storage/refresh mechanics), that is noted as unconfirmed in the relevant section rather than filled in from a secondary source. No claims were included that could not be grounded in a fetched primary source.

---

## 1. Claude Code's persistent memory hierarchy

Per the companion document's §3 ("Claude Code's persistent memory hierarchy"), Claude Code implements two complementary memory systems loaded at session start:

1. **CLAUDE.md files** (user-written instructions): Four scopes (managed policy, user, project, local) are discovered by walking the directory tree and loaded in order from root to working directory, so instructions closer to the current directory are read last. No size limit on individual files, though shorter files targeting under 200 lines produce better adherence.

2. **Auto memory** (MEMORY.md + topic files): Only the first 200 lines of `MEMORY.md` (or 25KB, whichever comes first) are loaded at session start; topic files are read on-demand.

The design separates instructions (CLAUDE.md, fully loaded, prescriptive) from learnings (auto memory, first-200-lines summary loaded, descriptive).

**For full details, depth, and rationale, see [python-agent-harness-engineering.md §3](./python-agent-harness-engineering.md#3-contextmemory-engineering).**

---

## 2. The AGENTS.md cross-tool standard

### What is AGENTS.md?

AGENTS.md is a cross-vendor, open Markdown standard for persisting agent instructions at the codebase level. Per the spec site, it "is now stewarded by the [Agentic AI Foundation](https://aaif.io) under the Linux Foundation" — the site does not give a transfer date, so none is asserted here. Per [agents.md spec site](https://agents.md/):

> AGENTS.md is a README for agents—a dedicated location offering context and instructions that complement human-focused documentation.

### Adoption and tool support

The spec site lists 23 supporting tools/platforms by name: Codex (OpenAI), Jules (Google), Factory, Aider, goose, opencode, Zed, Warp, VS Code, Devin (Cognition), UiPath Autopilot & Coded Agents, Junie (JetBrains), Amp, Cursor, RooCode, Gemini CLI (Google), Kilo Code, Phoenix, Semgrep, GitHub Copilot Coding Agent, Ona, Windsurf (Cognition), and Augment Code. Adoption is measured at "used by over 60k open-source projects" per the spec site, which links to a GitHub code search for `path:AGENTS.md` as its source for that figure.

### Directory-level nesting and monorepo support

The specification explicitly supports nested AGENTS.md files in monorepos. Per the spec site: "The closest AGENTS.md to the edited file wins; explicit user chat prompts override everything." This enables proximity-based precedence — if Windsurf is editing code in `packages/ui/`, it loads `.../packages/ui/AGENTS.md` before walking up to the repository root. OpenAI's own monorepo reportedly contains 88 such nested files at the time the spec was published.

A Codex-specific detail: Codex reads `AGENTS.override.md` if it exists (for temporary overrides), otherwise `AGENTS.md`, then falls back to `project_doc_fallback_filenames` if configured. It stops adding files once combined size reaches `project_doc_max_bytes` (32 KiB by default).

### Size conventions

No hard maximum is codified in the spec itself. However, OpenAI's own Codex repository AGENTS.md file is **322 lines long** — a substantial, project-specific instruction file that guides development of the Codex CLI itself (per GitHub direct fetch). This contradicts any assumption that the cross-tool convention favors "short ~100-line index" files; the actual OpenAI example is detailed and extensive.

---

## 3. Instruction file conventions in other coding-agent tools

### Cursor: .cursor/rules/ directory with .mdc files

**File location and structure**: Project rules live in `.cursor/rules/` as `.mdc` (Markdown Cursor) files. Rules can be organized in subdirectories, e.g. `.cursor/rules/react-patterns.mdc` or `.cursor/rules/frontend/components.mdc`. Plain `.md` files in `.cursor/rules/` are ignored if they lack frontmatter.

**Format and scoping**: Each `.mdc` file requires frontmatter specifying `description`, `globs` (file patterns the rule applies to), and `alwaysApply` (whether to load regardless of context). This allows rules to be scoped to specific file types or directories — for example, a rule for `**/*.ts` applies only to TypeScript files.

**Size guidance** (per [Cursor's official docs](https://cursor.com/docs/rules)): Keep individual rules **under 500 lines**. The documentation explicitly discourages bloated rules: *"Split large rules into multiple, composable rules"* and *"Point to canonical examples instead of copying code—this keeps rules short and prevents them from becoming stale as code changes."*

### Windsurf: workspace/global rule directories with activation modes

Per official documentation ([Memories & Rules](https://docs.windsurf.com/windsurf/cascade/memories), which redirects to `docs.devin.ai` following Cognition's acquisition of Windsurf):

**File format, location, and scopes**: The single-file `.windsurfrules` at the workspace root still works but is documented as "legacy." The current mechanism is a directory of rule files — `.devin/rules/*.md` or `.windsurf/rules/*.md` (workspace scope) — plus a **global** scope at `~/.codeium/windsurf/memories/global_rules.md` (a single file applied across all workspaces). `AGENTS.md` files anywhere in the workspace are also recognized as a rule source, with no documented size limit.

**Size guidance**: unlike the other tools surveyed here, Windsurf gives an exact character limit rather than a line-count target — **global rules are limited to 6,000 characters; workspace rule files are limited to 12,000 characters per file.**

**Four activation modes** (this is Windsurf's actual mechanism for deciding what's "in view" at any moment, richer than a single load-everything model): **Always On** rules have their full content injected into the system prompt on every message; **Model Decision** rules expose only their `description` in the system prompt, and Cascade reads the full file only if it decides that description is relevant to the current turn; **Glob** rules activate when Cascade reads or edits a file matching the rule's `globs` pattern; **Manual** rules are excluded from the system prompt entirely and are only pulled in when the user explicitly types `@rule-name`.

**Auto-generated memories**: separately from user-authored Rules, "Cascade can automatically generate and store memories if it encounters context that it believes is useful to remember" during a conversation — analogous in spirit to Claude Code's auto memory, though the docs give no character/line limit for these and note they live only on the local machine (the docs recommend writing anything durable as a Rule instead).

### Aider: .aiderignore and repository map, no dedicated instruction file

Aider does **not** have a persistent instruction file in the manner of CLAUDE.md or .cursorrules. Instead, Aider's context management relies on:

**Repository map**: A compressed token-efficient representation of the codebase (file names, function signatures, class definitions). Per [Aider's repo-map documentation](https://aider.chat/docs/repomap.html): "The token budget is influenced by the `--map-tokens` switch, which defaults to 1k tokens," and "Aider adjusts the size of the repo map dynamically based on the state of the chat."

**File filtering via .aiderignore**: Users can create a `.aiderignore` file (conforming to `.gitignore` syntax) to exclude parts of the repository not relevant to the task. This is a runtime context-trimming mechanism rather than persistent instructions.

**Convention files**: Aider supports reading context via `--read` option for `.aiderCode.md` or similar convention files, but this is manual per-invocation, not automatic like CLAUDE.md or .cursorrules.

### GitHub Copilot: .github/copilot-instructions.md and path-specific instructions

**File locations and hierarchy**:

1. **Repository-wide instructions**: `.github/copilot-instructions.md` applies to all requests in the repository.
2. **Path-specific instructions**: Files named `*.instructions.md` in `.github/instructions/` or subdirectories, using glob patterns (e.g., `applyTo: "**/*.ts,**/*.tsx"`) to target specific files or directories.
3. **Alternate formats supported**: AGENTS.md files anywhere in the repository, or root-level `CLAUDE.md` / `GEMINI.md`, are also recognized.

**Scoping and priority**: Per [GitHub's official documentation](https://docs.github.com/en/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot), "Personal instructions take the highest priority. Repository instructions come next, and then organization instructions are prioritized last." When both path-specific and repository-wide instructions match a file, instructions from both are used.

**Size guidance** (per GitHub docs): no general line-count limit is stated for authoring instruction files by hand. The one concrete size constraint found is narrower in scope than a general authoring guideline — it's a limitation baked into the prompt template GitHub gives to its own cloud coding agent when *generating* a `copilot-instructions.md` file on a user's behalf: "Instructions must be no longer than 2 pages."

---

## 4. Runtime context-selection mechanisms: codebase indexing and pruning

Beyond static instruction files, several tools implement runtime mechanisms to decide what code/file context the agent sees on each turn. This is distinct from session-level configuration and addresses the question: "Of all the code in this repository, which parts should be in the prompt right now?"

### Cursor: semantic retrieval via a self-trained embedding model

**Mechanism**: Cursor's own engineering blog confirms embeddings-based semantic search over the codebase: "To support semantic search, we've trained our own embedding model and built indexing pipelines for fast retrieval" ([Cursor blog — semantic search](https://cursor.com/blog/semsearch)); the post describes training that embedding model on agent session traces so retrieval ranking aligns with what an LLM actually finds relevant. No official Cursor source checked names the underlying vector-database/retrieval backend.

Per [Cursor's codebase indexing documentation](https://cursor.com/help/customization/indexing):

- **Automatic scanning**: "Cursor scans and indexes your source files" when a project opens, to enhance the Agent's ability to locate relevant code.
- **Periodic sync**: "The index syncs periodically (roughly every five minutes) to pick up changes."
- **Manual reindex**: Users can trigger a complete rebuild via command palette (`Cmd+Shift+P` / `Ctrl+Shift+P` → reindex option).
- **Performance**: Large repositories can improve speed by adding generated files to `.cursorignore` and excluding `node_modules`, `dist` (ignored by default if listed in `.gitignore`).

**Context control**: Users can use the `@` symbol system (`@codebase`, `@file`, `@folder`, `@docs`, `@web`) to precisely control what context the AI agent receives on a given turn, allowing fine-grained context scoping beyond what the index provides.

### Windsurf: RAG-based retrieval over a local codebase index

**Mechanism** (per [Context Awareness overview](https://docs.devin.ai/desktop/context-awareness/overview)): the documentation describes this as retrieval-augmented generation — "techniques to construct highly relevant, context-rich prompts to elicit accurate answers from an LLM" — implemented via proprietary "M-Query" retrieval techniques, stated directly as: "It also uses LLMs to perform retrieval-augmented generation (RAG) on your codebase using our own M-Query techniques." The docs confirm "the entire local codebase is... indexed (including files that are not open), and relevant code snippets are sourced by [the] retrieval engine as you write code, ask questions, or invoke commands."

**What isn't confirmed**: the official docs checked do not state whether the underlying representation is embeddings-based, whether the index is stored locally or remotely, or the exact refresh cadence (incremental-on-save vs. periodic rescan). `.windsurfignore` exists as a filtering mechanism (analogous to Cursor's `.cursorignore`) per [`windsurf-ignore`](https://docs.devin.ai/desktop/context-awareness/windsurf-ignore), and remote/large-codebase indexing is handled separately per [`remote-indexing`](https://docs.devin.ai/desktop/context-awareness/remote-indexing).

### Aider: repository map with token budget and .aiderignore filtering

**Repository map**: Aider creates a compressed, token-efficient representation of the codebase that fits within the context window. Per [Aider's usage documentation](https://aider.chat/docs/usage.html):

> You'll get the best results if you think about which files need to be edited. Add **just** those files to the chat. Aider will automatically pull in content from related files so that it can understand the rest of your code base.

**Selective file inclusion**: Users manually add files they want edited (`aider file1.py file2.py`), and Aider automatically includes related context via the repository map. The documentation's own reasoning, quoted directly rather than paraphrased: *"If you add too many files, the LLM can get overwhelmed and confused (and it costs more tokens)."*

**Filtering via .aiderignore**: A `.aiderignore` file (conforming to `.gitignore` syntax) tells Aider to exclude repository sections from the map and available context.

**Large codebase handling**: For large repositories, Aider supports `--subtree-only` to ignore code outside a specified subdirectory, focusing context only on the relevant subproject.

### Cline: file provenance tracking and on-demand documentation

**File tracking, verified directly against source** (`cline/cline`, `apps/vscode/src/core/context/context-tracking/`): the class responsible is `FileContextTracker`. It maintains a `FileMetadataEntry` per file with a `record_source` field typed as exactly `"read_tool" | "user_edited" | "cline_edited" | "file_mentioned"`, plus separate `cline_read_date`, `cline_edit_date`, and `user_edit_date` timestamps — confirmed verbatim in `ContextTrackerTypes.ts`. This is what lets Cline detect when a file changed out from under it (e.g. the user edited it outside the session) between reads.

**On-demand MCP documentation and caching/truncation tradeoff** (per [Cline's own engineering blog](https://cline.bot/blog/inside-clines-framework-for-optimizing-context-maintaining-narrative-integrity-and-enabling-smarter-ai) — a first-party source, though the specific class implementing this was not independently located in the source tree): the blog states documentation is retrieved "only when explicitly required for MCP-related tasks," citing "~8,000 tokens" as the size of documentation pulled in on demand rather than kept resident. On the tension between truncation and caching, the blog states plainly: "aggressively truncating older messages can break the cache, potentially increasing costs on subsequent turns" — i.e. Cline's context-trimming has to weigh context-window savings against invalidating a prompt cache.

---

## 5. Comparison: file format, size, and scoping across tools

| Tool | Instruction file | Format | Directory scoping | Size guidance | Runtime context mechanism |
|---|---|---|---|---|---|
| **Claude Code** | CLAUDE.md (4 scopes) | Markdown | Yes, 4-level hierarchy; proximity-based | Under 200 lines recommended | Auto memory (200-line MEMORY.md summary + lazy-loaded topic files) |
| **Cursor** | .cursor/rules/*.mdc | Markdown with frontmatter (`globs`, `alwaysApply`) | Yes, nested .mdc files with glob patterns per-file | Under 500 lines per file | Own-trained embedding model + indexing pipeline (~5 min sync); backend datastore not disclosed in official sources checked |
| **Windsurf** | `.devin/rules/` or `.windsurf/rules/` (+ global); legacy `.windsurfrules` still read | Markdown, 4 activation modes (Always On/Model Decision/Glob/Manual) | Yes, workspace + global scope | 6,000 chars (global); 12,000 chars/file (workspace) | RAG retrieval ("M-Query") over full local codebase index; storage/refresh mechanics not documented |
| **Aider** | .aiderignore (filtering only) | .gitignore syntax | Yes, via .aiderignore exclusions | N/A (no persistent instruction file) | Repository map (compressed codebase summary) + token budget |
| **GitHub Copilot** | .github/copilot-instructions.md + .github/instructions/*.instructions.md | Markdown (path-specific with `applyTo` glob frontmatter) | Yes, path-specific via globs | No general limit stated; "no longer than 2 pages" only as a constraint on GitHub's own auto-generation prompt template | Not documented; likely similar to GitHub's standard code-search mechanisms |
| **AGENTS.md standard** | AGENTS.md (any directory) | Plain Markdown | Yes, nested per directory; proximity-based precedence | 322 lines in OpenAI's own example; no hard limit in spec | Tool-dependent (Cursor uses embeddings; Codex uses file proximity rules) |

---

## Primary sources referenced

- [AGENTS.md Specification](https://agents.md/) — Cross-tool standard, scoping rules, monorepo support, adoption figures
- [agents.md Custom Instructions Guide](https://developers.openai.com/codex/guides/agents-md) — ChatGPT Learn documentation on AGENTS.md for Codex CLI
- [`openai/codex` GitHub repository](https://github.com/openai/codex) — Direct fetch of AGENTS.md file (322 lines, project-specific instruction example)
- [Cursor Docs — Rules](https://cursor.com/docs/rules) — Official documentation on .cursor/rules/ directory, .mdc format, 500-line guidance, frontmatter requirements
- [Cursor Docs — Codebase Indexing](https://cursor.com/help/customization/indexing) — Scanning, periodic sync, reindexing, .cursorignore performance tuning
- [Cursor engineering blog — semantic search](https://cursor.com/blog/semsearch) — confirms own-trained embedding model for codebase retrieval; does not name a vector-database backend
- [Windsurf/Cascade — Memories & Rules](https://docs.windsurf.com/windsurf/cascade/memories) (redirects to `docs.devin.ai`, post-acquisition) — official rule scopes, activation modes, character limits, auto-generated memories
- [Windsurf/Cascade — Context Awareness overview](https://docs.devin.ai/desktop/context-awareness/overview) — RAG/"M-Query" retrieval description
- [Aider Documentation — Usage](https://aider.chat/docs/usage.html) — File selection strategy, .aiderignore filtering
- [Aider Documentation — Repo Map](https://aider.chat/docs/repomap.html) — token-budget mechanics for the repository map (`--map-tokens`, dynamic sizing)
- [Aider Documentation — Configuration](https://aider.chat/docs/config.html) — confirms `--subtree-only` option; overview confirms no persistent instruction file
- [GitHub Docs — Adding Repository Custom Instructions for Copilot](https://docs.github.com/en/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot) — File locations, scoping, priority hierarchy; the only size limit found (2 pages) applies to GitHub's own instruction-generation prompt template, not general authoring
- [`cline/cline` GitHub repository](https://github.com/cline/cline) — direct source read of `apps/vscode/src/core/context/context-tracking/ContextTrackerTypes.ts` (`FileContextTracker`'s `FileMetadataEntry`/`record_source` fields, verified verbatim)
- [Cline engineering blog — context optimization](https://cline.bot/blog/inside-clines-framework-for-optimizing-context-maintaining-narrative-integrity-and-enabling-smarter-ai) — on-demand MCP documentation token figure, caching/truncation tradeoff
- [How Claude Remembers Your Project](https://code.claude.com/docs/en/memory) — Claude Code's persistent memory hierarchy (referenced from companion document §3)
- [Building an LLM Agent Harness in Python — §3 and §5](./python-agent-harness-engineering.md) — Companion document detailing Claude Code's memory system and subagent context isolation
