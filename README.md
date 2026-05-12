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
- **Modular Skilling:** Individual tasks (e.g. "Extract Table", "Apply Business Rule X") built as independent sub-workflows, updatable without touching the rest of the pipeline

---

## 2. Model Hierarchy

### A. Frontier Models — Cloud Intelligence
- **Providers:** Model-agnostic (Azure OpenAI, Google Vertex AI, AWS Bedrock, Anthropic API)
- **Role:** Complex synthesis, multi-step reasoning, final quality assurance
- **When to use:** High-nuance reasoning, tasks that cannot be decomposed

### B. Local / Cheaper Models — On-Premise Muscle
- **Engine:** Ollama or vLLM (GPU nodes) / LM Studio (dev/lighter use cases) — choice depends on deployment context
- **Role:** High-volume tasks — PII identification, classification, basic summarisation, formatting, bulk text generation
- **Output format:** Caveman Format (see Section 8) — ultra-concise, no filler, technical substance only

### Delegation Trigger — The Heuristic
Route to Local Model only when **all three** conditions are met:
1. Subtask is **self-contained** (no conversation history required)
2. Output is **substantially long** (~300+ tokens, tunable per use case)
3. Pattern is **repetitive** across multiple instances in the same run

> **Menial vs. complex** is not the primary criterion — volume and repetition are.

---

## 3. Orchestration Loop

```
User Task
    │
    ▼
n8n Workflow Engine
    │
    ▼
[Clean Room Gate] — Local Model identifies PII → Redaction Map generated → stored to RAM-disk
                  — document anonymised (tokens replace PII) before any cloud contact  [→ log]
    │
    ├─ [Token ROI Gate] ──COMPLEX──► Frontier Model (planner/reasoner)  [→ log]
    │                                       │
    │                                       ├─ Handles directly
    │                                       │
    │                                       └─ [Delegate condition met]  [→ log]
    │                                           └─ tool call: run_local(prompt, context_pages[])
    │                                                   │
    │                                                   ▼
    │                                            Memory content injected via MCP
    │                                                   │
    │                                                   ▼
    │                                            Local Model
    │                                            responds in Caveman format
    │                                                   │
    │                                                   ▼
    │                                            Result returned to Frontier ◄─┐
    │                                                                           │
    └─ [Token ROI Gate] ──SIMPLE───► Local Model (Caveman format) ─────────────┘  [→ log]
    │
    ▼
[Re-constitution] — deterministic script swaps tokens → original values restored
                  — Redaction Map vaporised from RAM-disk
    │
    ▼
[Irreversible action?] ──YES──► PAUSE → Human confirmation (HITL) → proceed/abort  [→ log]
         │
        NO
         │
         ▼
    [Data Validation Gate] ── schema check on output ──FAIL──► flag / retry  [→ log]
         │
        PASS
         │
         ▼
    Continue / End turn
```

---

## 4. Connectivity Standard — MCP

- **Standard:** Model Context Protocol (MCP)
- **Role:** Standardised interface between Frontier Models and local data sources (file systems, databases, memory layer)
- **Benefit:** Secure, auditable data access without brittle custom integration code
- **Deployment:** MCP Servers expose local resources to the AI layer in a controlled, queryable way

---

## 5. State Management — OpenBrain

**Single Source of Truth principle:** all project state lives in one persistent memory layer — not in model context windows or volatile memory. The Frontier Model (or orchestrator) writes; Local Models read only via MCP.

OpenBrain is the memory layer: an open-source system built on SQLite or PostgreSQL + pgvector, with an MCP server built in.

**Core design principle — source/embedding separation:** raw source data and vector embeddings live in separate tables. When a better embedding model ships, the index rebuilds without touching source data. The memory layer is upgrade-safe as the model landscape evolves.

**Privacy:** embeddings are generated locally via any OpenAI-compatible local inference runtime (see Section 2) — documents never leave the machine during indexing. Consistent with the Clean Room Pipeline (Section 6A).

**Context injection:** n8n/MCP specifies which memory content is injected into each model call. OpenBrain's semantic search ensures only relevant records are sent per call, keeping prompts self-contained regardless of knowledge base size.

The backend scales in two phases, driven by operational need:

- **Phase 1 — SQLite (default start):** single local file, no infrastructure overhead, MCP-ready from day one. Semantic search via sqlite-vec is available immediately — no separate embedding layer to add later. Move on when concurrent write contention or dataset size becomes a real problem.
- **Phase 2 — PostgreSQL (scale):** same tool, upgraded backend. Migration is handled within OpenBrain — source data untouched, only the embedding index rebuilt. Handles concurrency and larger knowledge bases. Process log TTL enforcement becomes automatic via the `anonymization_log` Postgres table (see Section 6).

## 5a. Lightweight Variant — Markdown Wiki + Git

For simpler processes where state is limited, predictable, and managed by a single operator, a Markdown-based Wiki under local Git version control is sufficient — no embedding model, no database, no external service.

**When to use this instead of OpenBrain:**
- Single operator (no concurrent write contention)
- Context injection is predictable — n8n knows which pages to send per workflow step without needing semantic search
- Knowledge base is small enough that full or near-full injection per call is practical
- Minimising external dependencies is a priority

**Wiki contents:** same categories as OpenBrain — glossaries, domain-specific rulesets, workflow checkpoints, decisions and rationale log — but stored as plain Markdown files, human-readable without tooling.

**Context injection:** n8n/MCP specifies which Wiki pages are injected into each model call explicitly. No semantic retrieval layer — what gets sent is determined by workflow logic, not similarity search.

**Git discipline:**
- n8n commits automatically at defined workflow checkpoints — not model-initiated
- Each operator works on their own named branch; merge to `main` only after HITL sign-off
- If HITL rejects an output, n8n reverts the branch to pre-run state; the process log (an append-only file in the repo) is excluded from reversion — it must record that the run occurred and was rejected
- Git branching per run prevents concurrent overwrites — sufficient for single-operator use

**When to choose OpenBrain instead:** if the project scope already suggests a large or growing knowledge base, semantic retrieval, or multiple operators — start with OpenBrain from day one rather than migrating later. Migration from Wiki + Git to OpenBrain is possible if scope changes, but accurate scoping at the onset is the better investment.

---

## 6. Security & Privacy

### A. The Clean Room Pipeline

Designed for enterprise environments where PII must never reach cloud infrastructure.

**Pipeline steps:**

1. **Ingestion (local):** Raw document received and processed entirely on-premise
2. **Anonymisation (Local Model + n8n):**
   - **2a. Identification:** Local Model identifies all PII entities in the document
   - **2b. Map generation:** n8n generates a **Redaction Map** — a temporary, local-only JSON lookup table mapping unique placeholder tokens (e.g. `{{ENTITY_01}}`) to real values. Sensitive data replaced with tokens before leaving the local environment
   - **2c. Storage:** The Redaction Map is stored exclusively in a RAM-disk or encrypted temp folder — it never touches persistent storage and is vaporised automatically when the process ends or the machine loses power. No forensic recovery is possible. The map must remain live in RAM for the duration of the entire job — from Step 2 through Step 5 (re-constitution). If the process is interrupted before re-constitution completes, the job must be restarted from ingestion
   - **2d. Process log:** A Clean Room event is written to the system log for each job — job ID, document reference, timestamps of map creation and destruction, no PII values. See Section 7
3. **Cloud Processing (Frontier Model):** Receives anonymised text with tokens only. Treats `{{ENTITY_01}}` as constant variables in its reasoning. Has zero access to underlying identity data
4. **Synthesis returned (Frontier Model → local):** Output contains tokens, not real values
5. **Re-constitution (deterministic script):** A deterministic string replacement script uses the local Redaction Map to swap placeholders back to original values. **No model inference involved** — 100% fidelity guaranteed.
6. **HITL sign-off:** Human reviews the reconstituted document before release

> The Brain (Cloud) stays blind to PII. The Muscle (Local/Orchestrator) ensures the final document is accurate and complete.

### B. API Key Management

No hardcoded credentials anywhere in the system — not in n8n workflows, not in prompts, not in configuration files. API keys for Frontier Model providers (Azure OpenAI, Anthropic, Google Vertex AI, AWS Bedrock) are treated as secrets and managed accordingly.

- **Secret vault:** keys are stored in a dedicated secret management system (HashiCorp Vault, AWS Secrets Manager, or equivalent). n8n retrieves credentials at runtime — keys are never embedded in workflow definitions
- **Scoped keys:** each provider gets a dedicated key with minimum required permissions. A key scoped to one provider cannot access another
- **Key rotation:** keys are rotated on a defined schedule and immediately on any suspected exposure. Rotation is handled at the vault level — no workflow changes required
- **No logging of keys:** the rolling diagnostic layer (Section 7) must never capture authorisation headers or credential values

---

## 7. System Logging

The system maintains an immutable, append-only log of all decision-level events — separate from state management (Section 5/5a) and independent of n8n's native logging. State records where the project is; the log records what the system did to get there.

**Storage:** a flat append-only file (JSON or plain text). Human-readable without tooling, trivial to grep or export, and portable across all deployment variants. Neither the Frontier Model nor n8n may overwrite or delete entries — only append.

**Applies to:** both OpenBrain and Wiki+Git deployments. Logging is orthogonal to state management.

### Permanent layer — decision-level logging

What gets recorded:
- Token ROI gate decisions — which model was selected, which routing condition triggered it
- Gate outcomes — which gate fired, pass or fail
- HITL events — approved or rejected, timestamp, job reference
- Model delegation triggers — when and why the Frontier Model delegated to Local
- Clean Room events — job ID, document reference, timestamps of map creation and destruction (an instance of the same pattern, described in Section 6)

What is explicitly not recorded: prompt content, model responses, or document content. Full content logging would potentially store PII or sensitive business content outside Clean Room protection, creating a compliance liability where none existed. Decision-level logging answers the questions that matter in practice — what routing decision was made, which gate fired, did a human approve or reject — without retaining the underlying content.

### Rolling diagnostic layer — full logging

A permanent 72-hour rolling window running alongside the decision-level log in all environments including production. Captures full prompt and response content for incident investigation and routing calibration — available whenever something breaks, without needing to be activated in advance.

- Post-anonymisation content only — prompts and responses contain tokens (`{{ENTITY_01}}`), not real values. Pre-anonymisation content is never captured; the risk/reward ratio does not justify it. Implementation must enforce this at the pipeline level — the logger should be positioned after the Clean Room gate, not as a policy dependent on correct configuration
- 72-hour auto-purge — entries deleted automatically as the window rolls forward; no manual cleanup required

---

## 8. Optimisation Protocols

### Caveman Format
Local Model returns output in compressed English — no articles, no filler, no pleasantries. Core meaning only.
- **Saving 1:** Local Model generates fewer output tokens
- **Saving 2:** Frontier Model reads fewer input tokens (the expensive side)
- No reasoning penalty — Frontier Model reconstructs meaning natively
- Frontier system prompt must explicitly expect compressed input
- Real-world savings: 14–21% on Frontier Model input tokens (benchmarked on major models)

### Context Caching
Frequently used reference content (domain rulesets, glossaries, reference data) is cached via API features to avoid re-sending on every call.

### Token ROI Logic
n8n gates evaluate task complexity before routing. Delegation to Local Model only when the volume/repetition heuristic is met (see Section 2).

---

## 9. Safety Architecture

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

## 10. Key System Differentiator

The architecture is fully **model-agnostic** — no dependency on a specific provider or model version. Frontier and Local model slots can be swapped as better or cheaper models emerge without redesigning the pipeline.

Auditability is built in at every layer: n8n logs workflow execution, MCP provides auditable data access, the memory layer maintains decision history and knowledge state (with full provenance tracking in OpenBrain, or Git commit history in the Wiki+Git variant), and the Redaction Map process is traceable via a dedicated process log — recording job ID, document reference, and destruction timestamp without retaining any PII values. The map itself is vaporised; the proof that it was handled correctly is not.

---

## Open Questions (pending concrete use case)
- Token threshold for delegation — 300 tokens is a starting estimate, needs calibration
- OpenBrain knowledge structure for the specific domain — what content gets stored, what metadata is tagged; verify chunking behaviour against the actual content types used
- Does the target Local Model support tool use? (affects complexity of local execution)
- Remote kill switch — how to halt a runaway run from any device
- Local embedding model selection — which model and runtime, benchmarked against the specific domain content
