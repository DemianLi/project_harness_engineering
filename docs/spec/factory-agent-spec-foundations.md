# Factory Equipment Monitoring / Crisis-Handling Agent — Spec Foundations

This document is the **decisions log** for turning the six-layer harness research (`docs/research/factory-agent-harness-l1` through `l6`) and the Pydantic AI v2 technical evaluation (`docs/research/pydantic-ai-evaluation.md`, `docs/research/pydantic-ai-l1-l6-implementation-mapping.md`) into an actual product spec. Where the research docs survey *what capabilities exist across platforms*, this document records *the specific choice made for this product*.

Status: living document, updated as each open question below is resolved. Not yet a complete spec — see "Pending prerequisites" for what still blocks full spec sign-off.

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

**5.2 — Incident/crisis classification standard.** No existing SOP was confirmed. Needed: a definition of what sensor-reading combinations or events constitute each severity tier of incident (e.g., routine / elevated / crisis), to ground L5 (evaluation criteria — "did the agent classify severity correctly?") and to refine §4's interim constraint policy into a full hard-constraint list.

**Action**: schedule interviews with plant operations / safety engineering to produce these two documents. Until delivered, L1/L4/L5/L6 design can proceed on the interim decisions above, but final sign-off on those layers is blocked on this workstream.

---

## 6. Open items (not yet discussed — carried forward from the original gap checklist)

The following 🟡/🟢 items from the original gap analysis have not yet been resolved and are carried forward to the next round of clarification:

- Multi-tenant model (single-factory vs. multi-factory SaaS)
- Human-in-the-loop workflow specifics (who is escalated to, response SLA)
- Non-functional requirements (latency budget for crisis responses, availability target, expected concurrent load, cost/token budget — the last is less relevant now given local model deployment, but hardware capacity planning replaces it)
- Domain-specific evaluation metrics (depends on §5.2 landing first)
- Data privacy/governance policy
- UI/UX design
- Testing/red-team strategy
- CI/CD and rollout plan
- Team ownership and ongoing model/framework upgrade responsibility
- Legal/compliance standard applicability (confirm which industrial safety/cybersecurity standards, if any, apply to this deployment and jurisdiction — not yet verified)

---

## Primary sources referenced

- `docs/research/factory-agent-harness-l1-information-boundary.md` through `l6-constraint-validation-recovery.md` — six-layer harness taxonomy research
- `docs/research/pydantic-ai-evaluation.md` — Pydantic AI v2 framework evaluation
- `docs/research/pydantic-ai-l1-l6-implementation-mapping.md` — layer-to-implementation mapping
