# Hybrid AI Orchestration Framework — Architecture Specification

## Philosophy
**AI as a component, not a manager.** The system uses deterministic orchestration (n8n) to enforce hard logic gates and stop points. No autonomous loops. No model holds the stop button — the orchestrator and humans do.

---

## 1. Core Orchestration Layer

- **Engine:** n8n (self-hosted or enterprise)
- **Pattern:** Deterministic orchestration — explicit workflow logic, not autonomous agent loops
- **Stop Gates:** Points where execution halts for:
  - **Human-in-the-Loop (HITL):** Mandatory manual validation before high-stakes or irreversible actions
  - **Data Validation:** Automated schema checks on AI output before proceeding
- **Token ROI Gates:** n8n logic decides whether each task routes to a Frontier Model (complex) or Local Model (high-volume/cheap) based on defined criteria
- **Modular Skilling:** Individual tasks (e.g. "Extract Table", "Check Compliance Rule X") built as independent sub-workflows, updatable without touching the rest of the pipeline

---

## 2. Model Hierarchy

### A. Frontier Models — Cloud Intelligence
- **Providers:** Model-agnostic (Azure OpenAI, Google Vertex AI, AWS Bedrock, Anthropic API)
- **Role:** Complex synthesis, multi-step reasoning, final quality assurance
- **When to use:** High-nuance reasoning, tasks that cannot be decomposed

### B. Local / Cheaper Models — On-Premise Muscle
- **Engine:** Ollama or vLLM (GPU nodes) / LM Studio (dev/lighter use cases) — choice depends on deployment context
- **Role:** High-volume tasks — PII identification, classification, basic summarisation, formatting, bulk text generation
- **Output format:** Caveman Format (see Section 6) — ultra-concise, no filler, technical substance only

### Delegation Trigger — The Heuristic
Route to Local Model only when **all three** conditions are met:
1. Subtask is **self-contained** (no conversation history required)
2. Output is **substantially long** (~300+ tokens, tunable per use case)
3. Pattern is **repetitive** across multiple instances in the same run

> Menial vs. complex is NOT the primary criterion — volume and repetition are.

---

## 3. Orchestration Loop

```
User Task
    │
    ▼
n8n Workflow Engine
    │
    ├─ [Token ROI Gate] ──COMPLEX──► Frontier Model (planner/reasoner)
    │                                       │
    │                                       ├─ Handles directly
    │                                       │
    │                                       └─ [Delegate condition met]
    │                                           └─ tool call: run_local(prompt, context_pages[])
    │                                                   │
    │                                                   ▼
    │                                            Wiki pages injected via MCP
    │                                                   │
    │                                                   ▼
    │                                            Local Model
    │                                            responds in Caveman format
    │                                                   │
    │                                                   ▼
    │                                            Result returned to Frontier ◄─┐
    │                                                                           │
    └─ [Token ROI Gate] ──SIMPLE───► Local Model ──────────────────────────────┘
    │
    ▼
[Irreversible action?] ──YES──► PAUSE → Human confirmation (HITL) → proceed/abort
         │
        NO
         │
         ▼
    [Data Validation Gate] ── schema check on output ──FAIL──► flag / retry
         │
        PASS
         │
         ▼
    Continue / End turn
```

---

## 4. Connectivity Standard — MCP

- **Standard:** Model Context Protocol (MCP)
- **Role:** Standardised interface between Frontier Models and local data sources (file systems, databases, project Wiki)
- **Benefit:** Secure, auditable data access without brittle custom integration code
- **Deployment:** MCP Servers expose local resources to the AI layer in a controlled, queryable way

---

## 4. State Management — The Wiki

- **Medium:** Markdown-based project Wiki (local, Git version-controlled — see Section 4a)
- **Role:** Single Source of Truth for project state — replaces reliance on model context windows or volatile memory
- **Write discipline:** Frontier Model (or orchestrator) writes; Local Models read only
- **Contents:**
  - Glossaries and terminology
  - Audit rulesets and legal constraints
  - Workflow checkpoints and current status
  - Decisions and rationale log
- **Context injection:** n8n/MCP specifies which Wiki pages are injected into each model call — keeps prompts self-contained
- **File watcher:** n8n monitors the Wiki folder for changes and triggers embedding updates automatically when files are saved — ensures the AI's view of the Wiki stays current
- **Context window guard:** As the Wiki grows, full injection becomes impractical. A semantic search / embedding layer must be introduced before that point — only the top 3–5 relevant snippets are sent per call. See Open Questions and Section 4b

## 4a. Wiki Version Control — Git

- **Purpose:** Makes write discipline enforceable and auditable rather than a convention; provides clean rollback without manual reconstruction
- **Trigger:** n8n commits to the Wiki automatically at defined workflow checkpoints — not model-initiated
- **Branching strategy:** Each workflow run operates on its own branch; merge to `main` only after HITL sign-off. Each branch maps to a single task — clean rollback unit
- **Revert as Stop Gate action:** If HITL rejects an output, n8n automatically reverts the Wiki branch to pre-run state before halting
- **Collision protection:** Git branching per run prevents concurrent overwrites. Sufficient for low-concurrency usage — high-concurrency multi-user environments will require a database backend (see Section 4b)
- **Commit attribution:** Currently moot — Frontier Model is the sole writer. When/if multi-model writing is introduced, each agent gets its own Git committer identity so the log shows exactly which model wrote what and when. Architecture is ready for this without redesign

## 4b. Storage Evolution — Phased Approach

Markdown + Git is the starting point. Migration is driven by operational need, not a fixed timeline.

| Phase | Storage | Trigger to move on |
|---|---|---|
| 1 — current | Markdown + Git | Works until concurrency or relational complexity becomes painful |
| 2 — optional bridge | Airtable | Multiple simultaneous users; need to inspect/edit records without SQL knowledge. Adds vendor dependency — skip if possible |
| 3 — target | PostgreSQL | Team comfortable with SQL basics; concurrency is a real problem; relational data (e.g. Client → Regulation → Audit Year) justifies the complexity |

**Hybrid model (Phase 3):** PostgreSQL as the master record store. n8n automatically exports relevant records as Markdown strings before injecting them into model calls — Postgres on the backend, Markdown as the AI-facing interface.

**Redaction Map upgrade (Phase 3):** The local JSON Redaction Map (Section 5, Step 2) moves to a dedicated `anonymization_log` table in Postgres with a TTL (Time To Live). PII-to-token mappings are automatically purged after 24 hours — stronger privacy guarantee, fully auditable, no manual cleanup.

**Semantic search (Phase 3 or later):** PostgreSQL with the `pgvector` extension stores embeddings alongside text records. The AI navigates the knowledge base by semantic similarity rather than keyword match — relevant for large-scale deployments, not current scope.

---

## 5. Security & Privacy — The Clean Room Pipeline

Designed for enterprise environments where PII must never reach cloud infrastructure.

**Pipeline steps:**

1. **Ingestion (local):** Raw document received and processed entirely on-premise
2. **Anonymisation (Local Model + n8n):** Local Model identifies PII entities; n8n generates a **Redaction Map** — a temporary, local-only JSON lookup table mapping unique placeholder tokens (e.g. `{{ENTITY_01}}`) to real values. Sensitive data replaced with tokens before leaving the local environment. The Redaction Map is stored exclusively in a RAM-disk or encrypted temp folder — it never touches persistent storage and is vaporized automatically when the process ends or the machine loses power. No forensic recovery is possible
3. **Cloud Processing (Frontier Model):** Receives anonymised text with tokens only. Treats `{{ENTITY_01}}` as constant variables in its reasoning. Has zero access to underlying identity data
4. **Synthesis returned (Frontier Model → local):** Output contains tokens, not real values
5. **Re-constitution (deterministic script):** A deterministic string replacement script uses the local Redaction Map to swap placeholders back to original values. **No model inference involved** — 100% fidelity guaranteed. Redaction Map remains local-only throughout
6. **HITL sign-off:** Human reviews the reconstituted document before release

> The Brain (Cloud) stays blind to PII. The Muscle (Local/Orchestrator) ensures the final document is accurate and complete.

---

## 6. Optimisation Protocols

### Caveman Format
Local Model returns output in compressed English — no articles, no filler, no pleasantries. Core meaning only.
- **Saving 1:** Local Model generates fewer output tokens
- **Saving 2:** Frontier Model reads fewer input tokens (the expensive side)
- No reasoning penalty — Frontier Model reconstructs meaning natively
- Frontier system prompt must explicitly expect compressed input
- Real-world savings: 14–21% on Frontier Model input tokens (benchmarked on major models)

### Context Caching
Frequently used rulesets (audit standards, glossaries, compliance rules) are cached via API features to avoid re-sending on every call.

### Token ROI Logic
n8n gates evaluate task complexity before routing. Delegation to Local Model only when the volume/repetition heuristic is met (see Section 2).

---

## 7. Safety Architecture

| Action type | Gate |
|---|---|
| Reversible | Proceeds autonomously |
| Irreversible | Hard stop — human confirmation required |

**Critical principle:** Confirmation gates and permission scopes live in **n8n workflow logic and infrastructure**, not in model prompts. Prompts can be lost to context window compaction.

### Documented failure modes this architecture addresses
| Incident | Root cause | Addressed by |
|---|---|---|
| Replit/Lemkin (Jul 2025) — production DB deleted | Agent violated code freeze | n8n Stop Gates |
| Amazon Kiro (Dec 2025) — 13hr AWS outage | Inherited elevated permissions bypassed approval | Scoped permissions at infra level |
| OpenClaw/Yue (Feb 2026) — bulk email deletion | "Confirm before acting" instruction lost to context compaction | Gates in orchestrator, not prompts |

---

## 8. Key System Differentiator

The architecture is fully **model-agnostic** — no dependency on a specific provider or model version. Frontier and Local model slots can be swapped as better or cheaper models emerge without redesigning the pipeline.

Auditability is built in at every layer: n8n logs workflow execution, MCP provides auditable data access, the Wiki maintains decision history, and the Redaction Map ensures PII handling is traceable.

---

## Open Questions (pending concrete use case)
- Token threshold for delegation — 300 tokens is a starting estimate, needs calibration
- Wiki structure for the specific domain
- Does the target Local Model support tool use? (affects complexity of local execution)
- Remote kill switch — how to halt a runaway run from any device
- Redaction Map storage and rotation policy for enterprise compliance (resolved in Phase 3 — see Section 4b)
- Embedding layer / semantic search implementation — which tool, at what Wiki size does it become necessary, how does n8n trigger re-indexing on file change
- Airtable evaluation — determine early whether the team can skip straight to Postgres or needs the intermediate step
