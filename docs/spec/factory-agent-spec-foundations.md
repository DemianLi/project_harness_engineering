# Factory Equipment Monitoring / Crisis-Handling Agent — Spec Foundations

This document is the **decisions log** for turning the six-layer harness research (`docs/research/factory-agent-harness-l1` through `l6`) and the Pydantic AI v2 technical evaluation (`docs/research/pydantic-ai-evaluation.md`, `docs/research/pydantic-ai-l1-l6-implementation-mapping.md`) into an actual product spec. Where the research docs survey *what capabilities exist across platforms*, this document records *the specific choice made for this product*.

Status: living document. All items from the original gap checklist have now been walked through and resolved, **except** the three prerequisites in §5 (5.1/5.2/5.3), which require input from stakeholders outside the engineering team (plant/safety engineering, legal/compliance) and cannot be resolved by further clarification alone. Those three items are what block full spec sign-off.

---

## 1. Scope statement

**AGEL-Comp (the neuro-symbolic, self-modifying agent architecture researched and evaluated earlier) is explicitly out of scope for v1.**

v1 is built on lightweight, already-proven harness patterns (Reflexion-style generate→critique→improve, Blueprint/ReWoo-style plan→verify→execute, strict tool-call validation) implemented on Pydantic AI v2. AGEL-Comp remains a candidate for a separate, parallel research track, to be revisited only if a specific, recurring failure mode is observed in production that these lighter patterns cannot address (i.e., the agent repeatedly fails at the same class of causal reasoning because it has no mechanism to retain learned domain rules across episodes).

This statement exists specifically so that engineering staff reading this spec do not assume the self-modifying neuro-symbolic architecture is part of the v1 deliverable.

---

## 2. Deployment environment

**Decision: air-gapped / strictly isolated network.** The factory's internal network cannot reach external cloud LLM APIs.

Implications:
- The agent must run on a **locally deployed, open-weights model**, not a cloud-hosted model (OpenAI/Anthropic/Google APIs are not reachable).
- Pydantic AI v2 remains the correct framework choice — it natively supports local model serving via Ollama/LiteLLM (verified in `docs/research/pydantic-ai-evaluation.md`, §5).
- Cloud-dependent components identified in the Pydantic AI mapping must be replaced with self-hosted equivalents:
  - Pydantic Logfire (cloud observability) → self-hosted OpenTelemetry collector/backend.
  - Any cloud vector store for RAG → on-prem/local vector store.
- Local open-weights models generally have lower reasoning capability than top cloud models (Claude/GPT/Gemini-class). This sets an upper bound on diagnostic reliability that must be reflected in the L5 evaluation criteria (still pending — see §6) and in how much autonomy the agent is given (reinforces the conservative default in §4).
- Hardware requirement: on-site GPU capacity sufficient to serve the chosen local model. Specific model choice and hardware sizing are a follow-up technical decision, not yet made.

---

## 3. Data source integration

**Decision: an existing SCADA/historian database is available on the factory's internal network and will be integrated with.**

- The agent's real-time telemetry and equipment-state queries will be implemented as tool functions (Pydantic AI `@agent.tool`) that query this existing database — no new data pipeline needs to be built from scratch for the core telemetry path.
- Specific database vendor, schema, and connection protocol are a follow-up integration detail, not yet specified — does not block the spec at this level, since "an integratable source exists" is the load-bearing fact for L1 (retrieval scoping) and L4 (state) design.

---

## 4. Constraint policy (interim, pending full hard-constraint list)

**Decision (blanket rule, in effect until the full hard-constraint list from §6's prerequisite is delivered):**

> Any tool/action that would change equipment state (shutdown, valve open/close, parameter adjustment, etc.) **requires human approval before execution**. Any tool/action that is read-only (status query, historical data lookup, diagnostic analysis) **may run fully autonomously**.

- Maps directly to Pydantic AI's `requires_approval=True` parameter (on `@agent.tool`, `Tool()`, or `FunctionToolset`) plus `HandleDeferredToolCalls` for the approval-handling flow (per `docs/research/pydantic-ai-l1-l6-implementation-mapping.md`, L2/L6 sections).
- This is a **read/write classification**, not a full severity-based hard-constraint list. It is a safe, conservative interim rule that lets tool-layer (L2) design proceed now, without waiting for the full incident-classification work in §6. It will likely be refined (e.g., some read-only actions may still warrant approval in certain crisis classifications) once that work lands — this rule is a floor, not the final policy.

---

## 5. Pending prerequisites (blocking — require non-engineering input)

These items cannot be resolved by the engineering/harness-design team alone. They require input from plant operations and safety engineering staff, and are being tracked as a single combined workstream since they share the same stakeholders and source material.

**5.1 — Equipment/sensor taxonomy and asset register.** No existing asset register was confirmed. Needed: a structured list of equipment types, sensor types, and what each sensor's readings mean, to ground L1 (information scoping) and L4 (state modeling).

**5.2 — Incident/crisis classification standard.** No existing SOP was confirmed. Needed: a definition of what sensor-reading combinations or events constitute each severity tier of incident (e.g., routine / elevated / crisis), to ground L5 (evaluation criteria — "did the agent classify severity correctly?") and to refine §4's interim constraint policy into a full hard-constraint list. **Directional principle established now, ahead of the SOP**: the classification standard and any resulting evaluation metrics should be built with a **conservative bias — prefer over-flagging (false positives on severity) over under-flagging (missing a real crisis)**. This principle should be used to break ties when the SOP work defines the actual thresholds.

**5.3 — AI conversation log retention/access policy.** The factory's existing data governance rules are assumed to cover sensor/operational data generally (see §7 below), but AI conversation logs are a new data type those rules likely don't address. Needed: how long conversation logs (including approval decisions made through the chat) are retained, and who is authorized to review them. Flagged for later resolution; not yet defined.

**Action**: schedule interviews with plant operations / safety engineering to produce the 5.1/5.2 documents, and separately define the 5.3 policy (does not require the same stakeholders as 5.1/5.2). Until delivered, L1/L4/L5/L6 design can proceed on the interim decisions above, but final sign-off on those layers is blocked on this workstream.

---

## 6. Product/operational decisions (resolved)

**6.1 — Multi-tenancy.** Single-factory deployment. No multi-tenant isolation architecture (silo/pool/bridge) is needed for L1/L4 — each factory (if this is ever deployed elsewhere) would run its own fully separate instance, consistent with the air-gapped, single-site nature of §2.

**6.2 — Human-in-the-loop model.** Approval requests are **not** routed to a separate on-duty operator or external notification channel. The end user is already in an active chat session with the agent; when a tool call requires approval (per §4's policy), the approval request surfaces back into that same chat session and the same user approves or denies it directly. This maps to Pydantic AI's `requires_approval=True` + `HandleDeferredToolCalls` flow entirely within a single session — no separate escalation/handoff infrastructure (of the kind researched in L3 for bot→human-agent handoff) is needed for this approval path. Because the approving user is already engaged in the conversation, no separate response-time SLA needs to be designed. Identity/authorization of who is allowed to open the chat and approve actions is handled by the **existing system-level account/permission design** — the agent must read identity/role from that existing system rather than implement its own authentication or authorization layer (consistent with the L1 finding that tenant/identity derivation should happen server-side via the existing system, not be reinvented in the harness).

**6.3 — Non-functional requirements.**
- Latency: up to **30 seconds** is an acceptable response time for a crisis-context message, given the constraint of local-model inference on-site hardware (§2).
- Concurrency: approximately **10 concurrent users** expected, consistent with a single-factory deployment (§6.1).
- Availability target: not yet set a specific number; interim approach is to benchmark against the existing SCADA/historian system's own availability standard (§3) rather than invent a new target — revisit once actual on-site hardware and local-model latency are benchmarked.

**6.4 — Domain-specific evaluation metrics.** Merged into the §5.2 prerequisite (cannot be made concrete before the incident classification SOP exists), but the conservative-bias principle from §5.2 applies here directly: any metric design should penalize false negatives (missed/under-classified crises) more heavily than false positives.

**6.5 — Data governance.** General factory data governance policy is assumed to apply to sensor/operational data accessed by the agent (no new policy needed there). AI conversation log retention/access is a new data type without existing coverage — tracked separately as §5.3.

---

## 7. Delivery and operational decisions (resolved)

**7.1 — UI/UX.** The chat interface will be **embedded into the factory's existing operational dashboard**, not built as a standalone app. Reduces context-switching cost for operators and keeps the agent inside the environment they already work in, especially important during a crisis.

**7.2 — Testing / red-team strategy.** Split into two tracks:
- The approval mechanism (§4's read/write rule) is pure engineering logic and can be unit-tested now, independent of the §5 prerequisites.
- Validating crisis-diagnosis logic against realistic scenarios is merged into the §5.1/5.2 prerequisite workstream: when interviewing plant/safety engineering for the asset register and incident classification standard, also collect 3-5 representative real past incidents (spanning different severity levels) to use as a simulated-replay test set.

**7.3 — CI/CD and rollout plan.** Phased rollout:
1. **Phase 1**: ship read-only capabilities only (status queries, historical data lookups) — does not depend on §5's prerequisites, can ship first.
2. **Phase 2**: enable crisis-diagnosis suggestions once §5.1/5.2 land, but every action remains subject to the human-approval rule in §4.
3. **Phase 3**: after a production track record accumulates sufficient accuracy data, revisit whether any low-risk actions can be granted autonomous execution.

**7.4 — Team ownership.** Initial candidates for harness engineering, Pydantic AI upgrade decisions, and crisis-logic safety sign-off have been identified (not detailed in this doc). Safety sign-off should be held by the same plant/safety engineering stakeholders consulted in §5.1/5.2, to avoid the engineering team reviewing its own implementation against a standard it also wrote.

**7.5 — Legal/compliance standard applicability.** Not resolved by this team — flagged as a reminder; the organization's legal/compliance team will determine which industrial safety/cybersecurity standards apply to this deployment's industry and jurisdiction.

**7.6 — Local model and hardware.** Two open-weights candidates were checked against the tool-calling/reasoning needs of this use case (diagnostic reasoning combined with tool calls to query telemetry and gate approvals, not pure tool-calling alone):
- Llama-3-Groq-8B-Tool-Use (Groq/Glaive, built on Meta Llama 3 8B) — purpose-built for tool calling (89.06% on BFCL, best among open 8B models at time of publishing), runs in 6-8GB VRAM at 4-bit, but narrowly optimized for tool use at the expense of general reasoning depth.
- **Mistral NeMo 12B (Mistral AI) — selected.** Native function-calling support with more balanced general reasoning capability, ~12GB VRAM. Chosen over the 8B tool-use-specialized option because this use case needs diagnostic reasoning alongside tool calls, not tool-calling accuracy alone.
- Note: Gemma 4 12B (considered earlier) was found to have comparatively weaker tool-calling support versus Llama/Mistral-family models — a relevant factor given this harness's reliance on tool calls for telemetry access and approval gating.
- Exact hardware sizing (GPU model/count) for serving Mistral NeMo 12B at the required latency/concurrency (§6.3) is a follow-up integration detail, not yet benchmarked on-site.

---

## 8. Remaining pending work

Everything from the original gap checklist has been resolved except:

- **§5.1 / §5.2 / §5.3** — the three prerequisites requiring plant/safety engineering and legal/compliance input (see §5). These block final sign-off on L1/L4/L5/L6 design and on the Phase 2/3 rollout in §7.3.
- **SCADA/historian vendor, schema, and connection protocol specifics** (follow-up to §3) — does not block spec-level decisions, resolve during integration.
- **On-site hardware sizing for Mistral NeMo 12B** (follow-up to §7.6) — benchmark once hardware is available.

---

## Primary sources referenced

- `docs/research/factory-agent-harness-l1-information-boundary.md` through `l6-constraint-validation-recovery.md` — six-layer harness taxonomy research
- `docs/research/pydantic-ai-evaluation.md` — Pydantic AI v2 framework evaluation
- `docs/research/pydantic-ai-l1-l6-implementation-mapping.md` — layer-to-implementation mapping
