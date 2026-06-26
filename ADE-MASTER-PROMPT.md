# ADE — MASTER BUILD PROMPT (self-contained)
> Paste this entire document into a capable coding agent (Claude Code / Codex / Cursor) to bootstrap and build the **ADE**. It embeds the full strategy, the decided tech stack, the data model, the core processes, the UI vision, and the **complete `atomic-os` substrate knowledge** — so the build needs no other source. Verified June 2026.

---

## 0. YOUR MISSION (role)

You are a **principal engineer + product architect**. Build the **ADE — Agent Development Environment**: a multi-tenant, OSS-first SaaS where *any* person (designer, developer, analyst, ops) visually builds, orchestrates, observes, and edits **teams of AI agents** for any kind of work. Reuse the state of the art for everything that is commodity (UI, orchestration, gateway, memory, observability); build only the one thing nobody else has — the **proof-carrying substrate** (`atomic`, see §3). Ship fast, stay multi-tenant-safe, and never fake done.

Work in phases (§11). At each phase produce runnable code, tests, and a short honest status (what works, what's stubbed, what's pending). Prefer permissive-licensed, self-hostable components (see the license-trap map in §5). Do not invent capabilities you cannot back.

---

## 1. WHAT THE ADE IS (the product)

A unified workspace — **one UI** — whose surface is an **infinite graph/canvas** where the user sees and administers every team, agent, and task, with a **floating, draggable chat** layered over it. In the chat the user can talk to the **System** (orchestrator), to a **team leader**, or to an **individual agent** of any team. Clicking an agent opens an **inspector** and re-targets the chat to it. Everything is observable and editable live.

A power user assembles a team of agents — each with its own LLM (via gateway or BYOK key), its own system prompt, its own tools/skills/MCP servers, and reasoning add-ons (e.g. token-saving skills). Agents, teams, the relations between them, shared I/O and state, and the evolution of each agent's memory over time are **all observable and fully editable**. Personas are served by one engine + persona-specific bundles ("packs") of tools/skills/connectors.

---

## 2. THE STRATEGIC THESIS (why we win)

The market for "build/orchestrate agent teams" is **crowded** (BridgeMind/BridgeSpace, agentsmesh, Composio Orchestrator, overstory, Conductor, Claude Squad, Vibe Kanban; Cursor/Claude Code/Codex drifting toward it). Orchestration is **not** our moat. **Our moat is the SUBSTRATE:** everyone else orchestrates agents that *hope* to be right (validate by test-and-retry); we orchestrate agents whose code work **carries proof** (deterministic, pre-disk verification — `atomic`).

> **Positioning:** *"The ADE where every agent's output is born proven."* `atomic` is the **L0 trust kernel**, transversal to all layers. Everything else is reused. The community **marketplace is proof-gated** — *"only what is born proven propagates"* — which solves the #1 problem of agent marketplaces (trusting community components).

Honest scope: `atomic` guarantees **integrity** (the tree never breaks), *not* **intent** ("did it do what the human meant" — undecidable, Rice). Intent is covered by the spec/QA-contract layer (AGENTS.md/SKILL.md + executable verifiers). Sell both together: proof-carrying integrity + observable orchestration. Never claim "infallible agents."

---

## 3. THE EMBEDDED SUBSTRATE: `atomic-os` (L0) — complete knowledge

> This section embeds everything the build needs about `atomic`. It is a real system (verified live, June 2026), with a real honest ceiling. Treat it as the **L0 trust kernel** of the ADE.

### 3.1 What it is, in one sentence — and the law
A **byte-exact, proof-carrying code-editing substrate**: a deterministic engine (the "hands") driven by an LLM (the "brain"), under one law applied recursively to 5 layers — **"only the proven exists."**

| Layer | Same law | Mechanism |
|---|---|---|
| **Disk** | no unproven byte is written | pre-disk convergence floor (P1) |
| **OS/Kernel** | kernel blocks un-enveloped writes (inescapable) | `byte-guard-kernel` (eBPF LSM / Endpoint Security) + proof-token |
| **Self-change** | system mutates itself only in proven-safe ways | `expand_self` + monotonic ratchet |
| **Memory** | only proven solutions become learning, in general form | proof-carrying operator admission (3 laws) |
| **Claims** | nothing is "done" without a number; a loss is logged as a loss | anti-facade + A/B measurement |

**Central inversion:** the right action space is not the "text patch" but the **proven intent transaction** — the agent declares the intended result, the layer computes the smallest faithful byte-mutation, **proves it before touching disk**, then materializes. AST/LSP/diff are internal organs, never the interface. *"Human diff is output, not input."*

### 3.2 Architecture (bottom-up)
1. **Inescapability kernel** — the OS refuses un-enveloped writes (proof-token `.atomic/write-tokens/<pid>-<sha>.json`; no token → write blocked). In live traffic it blocked **1,088 native mutations, 0 silently allowed**.
2. **Engine `atomic-edit-mcp` v4.0.0** — MCP server, **123 tools**, **266 gates**, **29 languages** (tree-sitter).
3. **`expand_self` kernel** — the only legal path to change atomic, under a monotonic ratchet (~583–595 promotions).
4. **6 sibling MCPs** — memory, swarm, sentinel, dashboard, edit-bench, edit-evolution.
5. **Agent layer** + 7-phase governance loop (PLAN→INVESTIGATE→PROPOSE→VALIDATE→COMMIT→VERIFY).
6. **A/B loop + weights/compression engine** (measure, learn, generalize).

### 3.3 The MCP engine (the "hands") — how it is built
- **Stack:** TypeScript on `@modelcontextprotocol/sdk` (official), **stdio** transport, **Zod** schemas, AST via **tree-sitter** (15 native grammars + web-tree-sitter wasm) + **ts-morph**. Entry: `server.ts` (orchestrator) → 33 `server-tools-*.ts` modules, under a file-size gate.
- **Per-tool pipeline (real code, e.g. `atomic_read_and_edit`):**
  1. `resolveSafeTarget` — containment guard (cannot escape repo / touch protected files);
  2. `guardSha` — optimistic concurrency (file unchanged under you);
  3. `replaceText` — compute the **smallest faithful byte-mutation**, preserve the rest;
  4. `requireNegativeProofForRemovedBytes` — **the inverted byte-default**: if the edit removes bytes it **requires** a `proofOfIncorrectness`, else **refuses**;
  5. `commit` — validate the tree **before** disk, write atomically (no torn files), record trace/receipt.
- Universal params: `preview` (dry-run), `verify: typecheck|lint`, `expectedSha256`, `lock`. Multi-host: Codex schema shim + one supervised launcher `atomic-edit-mcp-launcher.sh` all hosts point to.

### 3.4 The formal contribution — the **(a)+(e)** algebra (the genuinely un-cited cell)
Two mechanisms fused into one property:
- **(a) inverted byte-default** — correct-by-construction bytes are immutable to negative actions; deletion requires a **recomputed** disproof over the actually-removed bytes (the gate recomputes, never trusts a digest).
- **(e) commute-modulo-invariant algebra** — two verified edits commute iff their `mod/read` sets are disjoint.
- **The unification:** the gates' read-closure and the disproof's `readLoci` are the **same object** → a commuting merge provably preserves **both** the positive verdict **and** the negative obligation (unstated in OT/CRDT/Darcs/Pijul/Unison/PCC).
- **Machine-checked:** Z3 (all configs) + N-way induction in **Lean 4**; runtime refinement (73,728×2 configs); external soundness over **169,171** real edit-pairs (zod/type-fest/zustand) with **0 unsound verdicts**.
- **Properties P1–P10:** Floor, Soundness, Completeness, Closure, Monotonic admission, Ratchet, **Obligation-preserving confluence (P7)**, Recomputable disproof (P8), Truth-funnel (P9), Byte-positive convergence (P10).

### 3.5 The learning layer — proof-carrying "weights" (NOT a neural net)
`weights_admit.py` (deterministic, CPU-only, no LLM). A "weight" = a generalized resolution operator `{class, trigger, strategy, instances[], proof_n}`. 3 laws: **(1)** capture N not one (MDL/compression absorption); **(2)** born under necessity; **(3)** monotonic fidelity. State: **8 operators** + ~45 `CLASS-*` demolitions. Contains a real **VSA** (Vector Symbolic Architecture) with usage-reinforced weights — the neurosymbolic seed the ADE reuses in its memory/attention layer (L2).

### 3.6 The 6 sibling MCPs
`atomic-memory` (semantic intent ledger, SHA-256 receipts — basis of "memory with proof"), `atomic-sentinel` (failed-task/lock daemon), `atomic-swarm` (multi-agent), `atomic-dashboard` (observability), `atomic-edit-bench` (measurement), `atomic-edit-evolution` (A/B loop).

### 3.7 Live-verified state + the honest ceiling
**Verified live (June 2026):** `weights_admit.py --selftest` → `ALL LAWS HOLD: True`; `npm install && node paradigm-verify.mjs` → **15/16 green** (1 RED = P3 OS-resource gates needing kernel privilege; 1 SKIP = Lean absent, honestly not counted); `confluence_z3.py` → **ALL GREEN**.
**Honest ceiling (from its own anti-facade paper):** broad scoreboard **1 win / 9 ties / 2 losses** vs a native worker (parity-to-slightly-weaker — *general dominance falsified*). **Learned-operator transfer = NULL** (in isolation the weight added zero marginal value over the gate; the proven "lifter" is the **gate**, not the weights). Rice is **side-stepped, not defeated** (intent-correctness out of scope). Pending: the external SWE-bench on/off ablation at scale, and the product shell.

### 3.8 How `atomic` plugs into the ADE as L0
- **Native plug:** `atomic-edit` **is already an MCP server** → in the ADE it is **just another MCP tool** any agent equips (no rewrite). Code agents (the Dev team) equip it; design agents may not.
- **Transversal proof envelope:** any agent's code action (edit/create/exec) can be routed through the byte-floor → the whole ADE gains a "proven-no-break" floor without coupling the product to the substrate.
- **Proof-gated marketplace (L6):** reuse its `unify-hosts` / proof-gated propagation → *"only proven propagates"* solves community-component trust.
- **Memory with proof (L2/L5):** `atomic-memory`'s SHA-256 receipt becomes the "context with proof" mechanism — each memory admission/eviction decision logged and verifiable.

### 3.9 Porting it into the ADE repo
Canonical source: `core/atomic-edit/` (TypeScript, MCP, stdio; **Node 22**). Embed options: (1) point the agent-runtime's MCP client at the launcher (`.mcp.json` style, zero new code); (2) vendor/submodule it into `services/atomic-edit`; (3) package as an installable `npx atomic-edit` MCP (recommended mid-term — fixes embedding + the product-shell gap at once). Build: `npm install && npm run build` → `dist/`. Verification commands: `weights_admit.py --selftest`; `node paradigm-verify.mjs`; `confluence_z3.py`.

---

## 4. ARCHITECTURE — 8 layers (atomic is L0, transversal)

```
L7 WORKSPACE   Next.js · React Flow (canvas) · assistant-ui (chat) · Sigma.js (memory graph) · Yjs (multiplayer) · AG-UI/SSE
L6 MARKETPLACE proof-gated · skills=SKILL.md · agents=A2A agent cards · Docker MCP Catalog
L5 OBSERVABILITY Langfuse + OpenLLMetry/OTel (reasoning/tool/handoff traces, editable)
L4 ORCHESTRATION LangGraph (durable graph, MIT) + Temporal (durable execution)
L3 DEFINITION  AGENTS.md/SKILL.md · model·prompt·tools·skills(Caveman/RKT)   | CONN: Nango + MCP
L2 MEMORY      Letta (dual/working) + Graphiti (temporal KG) · pgvector→Qdrant
L1 GATEWAY     LiteLLM / Portkey · BYOK per agent/tenant · budgets · fallbacks
L-EXEC         E2B/Daytona microVM per tenant · xterm.js↔node-pty · Playwright
L0 SUBSTRATE   atomic-os (MCP) — proof-carrying — TRANSVERSAL to all layers
   Cross-cutting protocols: MCP · A2A · AGENTS.md · SKILL.md · AG-UI
```

---

## 5. THE DECIDED TECH STACK (per layer + license traps)

**Topology:** monorepo (Turborepo/pnpm) — `apps/web` (Next.js + React 19 + shadcn/ui + Tailwind), `apps/api` (**Hono** control plane — NOT Next route handlers, which time out on long runs), `services/agent-worker` (Python FastAPI). **Temporal** is the durable TS↔Python bridge.

| Concern | Primary pick | License | Alt |
|---|---|---|---|
| Agent canvas | **React Flow (@xyflow)** | MIT | Rete.js v2 |
| Chat/stream | **assistant-ui** | MIT | Vercel AI SDK `useChat` / CopilotKit |
| Memory graph view | **Sigma.js** | MIT | Cytoscape.js |
| Multiplayer | **Yjs + Hocuspocus** | MIT | Liveblocks (managed) |
| Event contract | **AG-UI over SSE** (WS later) | MIT/Apache | — |
| Orchestration | **LangGraph (MIT pieces + own server)** | MIT | OpenAI Agents SDK / Mastra |
| Durable exec | **Temporal** (start on Cloud) | MIT | Inngest=SSPL / Restate=BSL / Trigger.dev=no-Python |
| Memory runtime | **Letta** (dual/self-editing) + **Graphiti** (temporal KG) | Apache-2.0 | Mem0 / Cognee |
| Vector store | **pgvector** → Qdrant at scale | PostgreSQL/Apache | Weaviate |
| LLM gateway | **LiteLLM** (pin versions) | MIT | **Portkey** (Apache-2.0) |
| Connectors | **Nango** (self-host) + **Docker MCP Catalog** | Elastic v2 / — | Composio (fast, closed backend) |
| Sandbox/terminal | **E2B / Daytona** (microVM/tenant) | Apache / AGPL | Modal (gVisor) / Firecracker self-host |
| DB | **Supabase Postgres + pgvector**, RLS shared-schema | Apache/PostgreSQL | Neon |
| Auth/orgs/RBAC | **Better Auth** (orgs+roles+SSO) | MIT | WorkOS / Clerk |
| Realtime | **Supabase Realtime** (fan-out) + **Yjs** (canvas) | Apache/MIT | Centrifugo |
| Observability | **Langfuse** (self-host) + OpenLLMetry | MIT/Apache | — |
| Billing/Secrets | **Stripe Billing Meters** · envelope-encryption + KMS (→ Infisical) | SaaS / MIT | OpenBao |

**⚠️ License traps (avoid for multi-tenant SaaS):** tldraw (commercial license + watermark → use React Flow); **LangGraph `langgraph-api` server = Elastic v2** (use MIT pieces + own server); CrewAI/Composio (multi-tenant/creds gated to paid/closed); **Arize Phoenix = ELv2** (use Langfuse); Inngest=SSPL / Restate=BSL / Trigger.dev=no-Python (use Temporal); Convex=FSL; LiteLLM had a Mar-2026 supply-chain incident (pin + self-host, or use Portkey); Daytona/Coder core=AGPL (use as service, or E2B Apache). Every primary pick above is permissive (MIT/Apache/Postgres) and self-hostable.

---

## 6. STANDARDS / PROTOCOLS (speak natively, day 1)
Converged under the **Agentic AI Foundation (Linux Foundation, Dec 2025)**. Adopt the quad + AG-UI:
- **MCP** — tool/context bus (spec `2025-11-25`, streamable-HTTP + stdio, OAuth 2.1). `atomic-edit` is already an MCP server → plugs in natively.
- **A2A** — agent↔agent interop (signed agent cards); 150+ orgs. (ACP is dead → merged into A2A.)
- **AGENTS.md** — repo config (60k+ repos). Read/generate. *Note: LLM-generated AGENTS.md hurt results (−2% success, +23% cost) — config must carry curated non-obvious info, not boilerplate.*
- **SKILL.md** — portable skills w/ progressive disclosure; the marketplace unit.
- **AG-UI** — agent→UI events (17 types, SSE). Avoids proprietary glue.
- Skip: ACP, `.soul`. Optional: `llms.txt` (public docs only).

---

## 7. DATA MODEL (Postgres + pgvector; RLS on `tenant_id` as the isolation primitive)
Every table carries `org_id`/`tenant_id`; RLS policies enforce isolation. Core tables:
```
orgs(id, name, plan)                                   -- tenant root
users(id, email)  ·  memberships(org_id, user_id, role)   -- RBAC
agents(id, org_id, name, model_ref, system_prompt, config_jsonb)
agent_versions(id, agent_id, version, manifest_jsonb, proof_receipt)   -- proof-gated
skills(id, org_id, name, skill_md, kind)               -- Caveman/RKT/...; SKILL.md
teams(id, org_id, name, graph_jsonb)                   -- LangGraph topology
team_members(team_id, agent_id, role)
workflows(id, org_id, definition_jsonb)                -- Temporal/LangGraph
runs(id, org_id, team_id, status, thread_id, started_at)   -- thread_id = isolation
run_events(id, run_id, seq, type, payload_jsonb)       -- AG-UI stream + trace
messages(id, run_id, role, content, tool_calls_jsonb)
memories(id, org_id, agent_id, scope, content, embedding vector, valid_from, valid_to)  -- temporal
memory_edges(id, org_id, src, dst, relation, t_valid)  -- Graphiti temporal graph
connections(id, org_id, provider, oauth_tokens_encrypted)   -- Nango
mcp_servers(id, org_id, url, transport, verified_bool)      -- Docker MCP Catalog
byok_keys(id, org_id, provider, key_encrypted)              -- envelope (KMS)
marketplace_items(id, kind, ref, proof_status, downloads)   -- proof-gated
traces(id, run_id, langfuse_ref)
```

---

## 8. CORE PROCESSES & LOGIC

**P1 — Agent-run lifecycle:** request → resolve agent config (model/prompt/tools/skills) → **L1 gateway** picks LLM (tenant BYOK, budget, fallback via LiteLLM) → **L4** runs the LangGraph graph (checkpointed by `thread_id`; long runs → **Temporal** for durability/retry/resume) → tool calls route via **MCP**; code edits go through **L0 atomic** (pre-disk proof); exec/terminal goes to the per-tenant **sandbox** → events emitted as **AG-UI** over SSE → `run_events` + assistant-ui renders tokens/tool-calls → **memory** updated (Letta tiers + Graphiti temporal edges) → **trace** to Langfuse. All observable AND editable.

**P2 — Team orchestration:** LangGraph supervisor graph; **handoffs** via A2A; shared state in checkpointer/Store; **HITL** via `interrupt()` (pause → human approve → resume).

**P3 — Real-time context administration (the differentiator):** context is ASSEMBLED per LLM call from persistent sources. Administer 3 surfaces — **sources** (Letta editable in-context memory blocks via REST API + LangGraph state/checkpoints + Graphiti temporal facts), the **assembler** (token-budgeted: summarization/compaction, priority-based eviction, staged loading — where the "attention" logic lives), and **observation** (live via AG-UI/SSE stream; full replay via Langfuse). Two loops: intra-run (stream + `interrupt()` to intervene) and inter-run (edit persistent memory blocks → shape the next assembly). Each admission/eviction decision logged with an atomic-style receipt → *context with proof* (the neurosymbolic-attention seed, built on Letta+Graphiti+VSA; treat deep VSA-attention as a research bet, not a blocker).

**P4 — Proof-gated marketplace:** publish skill/agent → run the gate battery → only verified propagates (reuse atomic's `unify-hosts` mechanic).

**P5 — Persona packs:** one engine; designer/dev/etc = different bundles of tools+skills+connectors over the same engine. Composio/Nango = the connector layer for non-devs.

---

## 9. THE UI (one UI: graph/canvas + floating chat + inspector)
"Mission control for AI agents": Linear (calm precision) × Figma/FigJam (canvas + multiplayer) × n8n/React Flow (node editor) × Raycast (command palette). Dark, glassmorphism, one violet accent (#7c6cff) + cyan (#22d3ee) + gold (#f5c451 for leaders/proof).
- **Canvas:** teams as translucent grouped zones; agents as glass node-cards showing avatar, role, **per-provider model badge**, pulsing live status (working/idle/error/done), **real-time context meter** (token budget bar), skill chips, **atomic ✓** proof badge, **leader crown**; bezier edges (active=animated glow, cross-team=dashed gold A2A); task cards with progress.
- **Floating chat** (draggable, glass over canvas): a **target selector** pill — System (orchestrator) / a team **leader** / an **individual agent**; streaming tokens, tool-call cards, per-message author+model badge. Clicking a node re-targets the chat.
- **Inspector** (right slide-over, tabs Config/Context/Trace): model (BYOK), system prompt, skills/tools, and the **real-time context** panel (live token budget, pinned/editable memory blocks with source = core/Graphiti-temporal/compacted-history, evicted items, time-travel over checkpoints), plus the step trace with atomic receipts.
- Top bar (workspace switcher, view toggle Graph/Canvas/Table, ⌘K, presence avatars, Run); left rail (teams + library: skills, tools/MCP); minimap + zoom; multiplayer cursors; HITL approve/reject affordance.

---

## 10. BUILD ORDER (phased — produce runnable code + honest status each phase)
- **Phase 0 — Foundation:** Turborepo monorepo; canonical Agent/Team schema (extends AGENTS.md+SKILL.md); Postgres+pgvector+RLS; `atomic-edit` wired as a built-in MCP (L0); LiteLLM proxy up (L1); one-command `make verify`.
- **Phase 1 — Vertical slice (1 agent E2E):** L1 gateway + L3 definition + L0 substrate + L5 trace + chat UI (assistant-ui). Deliverable: a code-review agent, BYOK, with a token-saving skill (Caveman), editing with proof via atomic, trace visible in Langfuse.
- **Phase 2 — Teams & workspace:** L4 (LangGraph + Temporal: handoffs/HITL) + L7 canvas (React Flow) + L2 memory (Letta+Graphiti) + multiplayer (Yjs). Deliverable: build/observe/**edit** a team live.
- **Phase 3 — Exec & remote-control:** per-tenant microVM sandbox (E2B/Daytona) + terminal (xterm.js↔node-pty) + Playwright. Deliverable: a dev agent with isolated terminal + remote-control.
- **Phase 4 — Community:** proof-gated marketplace (Docker MCP Catalog + SKILL.md/A2A). Deliverable: publish/install a community agent with proof.
- **Phase 5 — Neurosymbolic memory:** VSA/proven attention over Letta+Graphiti (the research bet).

---

## 11. HONEST CONSTRAINTS (anti-facade — do not violate)
- Don't become "another orchestrator" — the pitch is the **proven substrate**; don't dilute it.
- `atomic` guarantees **integrity, not intent** (Rice). Pair it with the spec/QA-contract layer for "did it do the right thing."
- "Deterministic" has a ceiling: the substrate forbids broken states; it does not create intelligence.
- LLM-generated config hurts — force curated non-obvious content in agent/team definitions.
- Never fake green: a failing test is reported as failing; a stubbed feature is labeled stubbed; a number you don't have is "pending," not invented.
- Honor every license trap in §5. Prefer permissive + self-hostable. Secrets only via env/KMS, never hardcoded.

---

## 12. WHAT TO PRODUCE (now)
Start at **Phase 0**: scaffold the Turborepo monorepo (`apps/web`, `apps/api`, `services/agent-worker`, shared `packages/*`), the Postgres schema (§7) with RLS, the `atomic-edit` MCP wired in as L0, the LiteLLM gateway, and `make verify`. Output the tree, the key files, setup instructions, and an honest status of what runs vs what is stubbed. Then proceed phase by phase, pausing for review at each phase boundary. Keep every primary pick from §5; flag any deviation and why.
