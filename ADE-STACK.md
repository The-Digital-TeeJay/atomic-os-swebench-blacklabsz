# A.D.E — Stack Técnico de Construção
> Stack opinativo por camada · modelo de dados · processos · lógica central · armadilhas de licença.
> Greenfield, multi-tenant, OSS-first, rápido de construir. `atomic-os` entra como **tool MCP built-in** (L0). Pesquisa verificada (jun/2026), licenças conferidas.

---

## 0. O stack default (uma escolha por camada — pra construir rápido)

| Camada | Escolha primária | Licença | Alternativa | Por quê |
|---|---|---|---|---|
| **Topologia** | **Monorepo (Turborepo/pnpm)**: `apps/web` (Next.js UI) + `apps/api` (**Hono** control plane) + `services/agent-worker` (Python FastAPI) | — | all-TS (Mastra) | massa madura de orquestração é Python; UI é TS; **Temporal** é a ponte durável TS↔Python |
| **L-API control plane** | **Hono** (não Next route handlers — estouram timeout em runs longos) | MIT | NestJS 11 / FastAPI | `streamSSE` ótimo, tenant-scoping por request |
| **L7 UI — shell** | **Next.js (App Router) + React 19 + shadcn/ui + Tailwind** | MIT | Remix | streaming nativo, AI SDK first-party |
| **L7 UI — canvas de agentes** | **React Flow (@xyflow)** | MIT | Rete.js v2 | nós/edges tipados p/ orquestração editável |
| **L7 UI — chat/stream** | **assistant-ui** | MIT | Vercel AI SDK `useChat` / CopilotKit | render de token + tool-call + HITL |
| **L7 UI — grafo de memória (escala)** | **Sigma.js** | MIT | Cytoscape.js | 50k+ nós WebGL (read-only) |
| **L7 UI — multiplayer** | **Yjs + Hocuspocus** (self-host) | MIT | Liveblocks (managed) | CRDT, sem taxa por-MAU |
| **L7 — contrato de eventos** | **AG-UI** sobre **SSE** (AI SDK data-stream); WS depois | MIT/Apache | — | padrão maduro agente→UI |
| **L6 Marketplace** | **proof-gated**: skills=`SKILL.md`, agentes=**A2A agent cards** | — | — | "só propaga o que nasce provado" |
| **L5 Observabilidade** | **Langfuse** (self-host, MIT) instrumentado via **OpenLLMetry→OTLP** | MIT/Apache | ⚠️ Phoenix=ELv2 / LangSmith=proprietário | trace de raciocínio/tool/handoff; OTel = hedge portável |
| **L4 Orquestração** | **LangGraph (peças MIT) + servidor próprio**, **Temporal** como backbone durável (SDK TS+Python) | MIT | OpenAI Agents SDK (licença 100% limpa) / Mastra (all-TS) | grafo durável + HITL + checkpoint Postgres |
| **L4 Durabilidade** | **Temporal** (MIT; comece no Cloud ~US$100/mo, self-host depois) | MIT | ⚠️ Inngest=SSPL / Restate=BSL / Trigger.dev=sem SDK Python | runs longos, retry/resume, cross-language |
| **L3 Definição de agente** | schema próprio ingerindo **AGENTS.md + SKILL.md**; skills de raciocínio (Caveman/RKT) plugáveis | — | — | padrão de fato (60k+ repos) |
| **L3 Conectores** | **Nango** (self-host) + **Docker MCP Catalog** | Elastic v2 / — | Composio (rápido, backend fechado) | você é dono dos tokens OAuth |
| **L2 Memória/RAG** | **Letta** (memória dual auto-editável) + **Graphiti** (grafo temporal) | Apache-2.0 | Mem0 / Cognee | substrato do "peso temporal/factício" |
| **L2 Vector store** | **pgvector** (→ **Qdrant** ao escalar) | PostgreSQL/Apache | Weaviate | RLS como isolamento multi-tenant |
| **L1 Gateway multi-LLM** | **LiteLLM** (self-host, pin de versão) | MIT | **Portkey** (Apache-2.0, licença mais limpa) | virtual keys + budget org→team→key + BYOK |
| **L-Exec Sandbox/terminal** | **E2B** ou **Daytona** (microVM/tenant) hosted; **Firecracker** self-host | Apache-2.0 / AGPL | Modal (gVisor) | isolamento hardware por tenant |
| **L0 Substrato provado** | **atomic-edit MCP** (built-in) + byte-floor transversal | — | — | **o moat** — ninguém mais tem |
| **DB** | **Supabase Postgres + pgvector** (RLS shared-schema, `tenant_id`) | Apache/PostgreSQL | Neon / Postgres puro | DB+vector+realtime+RLS num vendor só |
| **Auth/orgs/RBAC** | **Better Auth** (orgs+roles+invites+SAML/OIDC) | MIT | WorkOS (SSO enterprise) / Clerk (managed) | OSS, você é dono do DB (sem phone-home no kernel) |
| **Realtime** | **Supabase Realtime** (fan-out multi-cliente) + **Yjs** (CRDT do canvas) | Apache/MIT | Centrifugo | dois jobs distintos: broadcast vs colaboração |
| **Billing / Secrets** | **Stripe Billing Meters** · **envelope-encryption + KMS** (→ Infisical) | SaaS / MIT | OpenBao | meters agora obrigatórios; KMS = raiz de confiança BYOK |

---

## 1. Arquitetura em camadas (a visão, agora com tecnologia concreta)

```
┌──────────────────────────────────────────────────────────────────────────┐
│ L7 WORKSPACE   Next.js · React Flow (canvas) · assistant-ui (chat) ·        │
│                Sigma.js (grafo memória) · Yjs (multiplayer) · AG-UI/SSE     │
├──────────────────────────────────────────────────────────────────────────┤
│ L6 MARKETPLACE  proof-gated · SKILL.md + A2A agent cards · Docker MCP Cat.  │
├──────────────────────────────────────────────────────────────────────────┤
│ L5 OBSERVABILIDADE  Langfuse + OTel (trace raciocínio/tool/handoff, editável)│
├──────────────────────────────────────────────────────────────────────────┤
│ L4 ORQUESTRAÇÃO  LangGraph (grafo durável, MIT) + Temporal (execução durável)│
├──────────────────────────────────────────────────────────────────────────┤
│ L3 DEFINIÇÃO  AGENTS.md/SKILL.md · model·prompt·tools·skills(Caveman) │ CONN: Nango + MCP │
├──────────────────────────────────────────────────────────────────────────┤
│ L2 MEMÓRIA  Letta (dual) + Graphiti (temporal KG) · pgvector→Qdrant        │
├──────────────────────────────────────────────────────────────────────────┤
│ L1 GATEWAY  LiteLLM/Portkey · BYOK por agente/tenant · budgets · fallback  │
├──────────────────────────────────────────────────────────────────────────┤
│ L-EXEC  E2B/Daytona microVM por tenant · xterm.js↔node-pty · Playwright    │
├──────────────────────────────────────────────────────────────────────────┤
│ L0 SUBSTRATO (atomic-os, MCP)  proof-carrying — TRANSVERSAL a todas        │
└──────────────────────────────────────────────────────────────────────────┘
        Protocolos transversais: MCP · A2A · AGENTS.md · SKILL.md · AG-UI
```

---

## 2. Protocolos que a ADE fala nativamente (dia 1)

Convergência sob a **Agentic AI Foundation (Linux Foundation, dez/2025)** — adotar o "quad":
- **MCP** — barramento de tools/contexto (spec `2025-11-25`, transport *streamable HTTP* + stdio, OAuth 2.1). `atomic-edit` já é servidor MCP → **plugа de fábrica**.
- **A2A** — interop agente↔agente (agent cards assinados); 150+ orgs. (ACP morreu → fundido no A2A.)
- **AGENTS.md** — config de repo (60k+ repos). Ler/gerar.
- **SKILL.md** — skills portáveis com *progressive disclosure*. Unidade do marketplace.
- **AG-UI** — eventos agente→UI (17 tipos, SSE). Evita glue-code proprietário.
- **Skip:** ACP, `.soul` (sem footprint real). **Opcional:** `llms.txt` (só doc pública).

---

## 3. Modelo de dados (Postgres + pgvector, isolamento por RLS em `tenant_id`)

Núcleo do schema (toda tabela carrega `tenant_id`, política RLS força isolamento):

```
orgs(id, name, plan)                          -- tenant raiz
users(id, email, ...)
memberships(org_id, user_id, role)            -- RBAC (owner/admin/member)
agents(id, org_id, name, model_ref, system_prompt, config_jsonb)
agent_versions(id, agent_id, version, manifest_jsonb, proof_receipt)  -- proof-gated
skills(id, org_id, name, skill_md, kind)      -- Caveman/RKT/etc; SKILL.md
teams(id, org_id, name, graph_jsonb)          -- topologia LangGraph
team_members(team_id, agent_id, role)
workflows(id, org_id, definition_jsonb)       -- Temporal/LangGraph
runs(id, org_id, team_id, status, thread_id, started_at, ...)   -- thread_id = isolamento
run_events(id, run_id, seq, type, payload_jsonb)  -- stream (AG-UI), trace
messages(id, run_id, role, content, tool_calls_jsonb)
memories(id, org_id, agent_id, scope, content, embedding vector, created_at, valid_from, valid_to)  -- temporal
memory_edges(id, org_id, src, dst, relation, t_valid)   -- grafo temporal (Graphiti)
connections(id, org_id, provider, oauth_tokens_encrypted)  -- Nango
mcp_servers(id, org_id, url, transport, verified_bool)     -- Docker MCP Catalog
byok_keys(id, org_id, provider, key_encrypted)             -- envelope-encrypted (KMS)
marketplace_items(id, kind, ref, proof_status, downloads)  -- proof-gated
traces(id, run_id, langfuse_ref)
```

**Decisão multi-tenant:** Row-Level Security (RLS) em `tenant_id`/`org_id` como primitivo único de isolamento (não schema-per-tenant). Vetores ficam no mesmo Postgres (pgvector) → consistência transacional com metadados; migra hot/grande para Qdrant (filtro por payload de tenant) só quando escalar.

---

## 4. Processos & lógica central (os fluxos que fazem a ADE funcionar)

**P1 — Ciclo de vida de um agent-run (o coração):**
1. Request chega (UI/API) → resolve config do agente (model, prompt, tools, skills) de `agents`+`agent_versions`.
2. **L1 Gateway** escolhe o LLM (BYOK do tenant, budget, fallback) via LiteLLM.
3. **L4** executa o grafo LangGraph (estado *checkpointed* por `thread_id`); runs longos → **Temporal** garante durabilidade/retry/resume.
4. Tool-calls roteiam por **MCP**; edição de código vai pelo **L0 atomic** (prova pré-disco); execução/terminal vai pro **sandbox** (microVM do tenant).
5. **Eventos** emitidos em **AG-UI** → SSE → `run_events` + **assistant-ui** renderiza token/tool-call; canvas atualiza via Yjs.
6. **Memória** atualizada: Letta (working/archival) + Graphiti (aresta temporal com `valid_from/to`).
7. **Trace** → Langfuse. Tudo observável **e editável** na UI.

**P2 — Orquestração de time (multi-agente):** grafo supervisor no LangGraph; **handoffs** entre agentes via A2A; estado compartilhado no checkpointer/Store; **HITL** via `interrupt()` (pausa→aprovação humana→resume).

**P3 — Memória neurosimbólica (o diferencial):** admissão seletiva no contexto curto = tiers do Letta; **peso temporal/factício** = grafo bi-temporal do Graphiti (recência + validade de fato); decisão "o que entra/sai da atenção" é **logada com recibo** (estilo `atomic-memory` SHA) → memória *com prova*. (Camada VSA/atenção-neurosimbólica = aposta de pesquisa em cima disso, não bloqueia o produto.)

**P4 — Propagação proof-gated (marketplace):** publicar skill/agente → roda bateria de gates → **só o verificado propaga** (reusa a mecânica `unify-hosts`/proof-gated do atomic). Resolve o problema nº1 de marketplaces: confiança em componente da comunidade.

**P5 — Persona-packs:** um motor só; designer/dev/etc. = *bundles* diferentes de tools+skills+conectores sobre o mesmo engine. Composio/Nango = camada de conector pros não-devs.

---

## 5. ⚠️ Armadilhas de licença (mapeadas — críticas pra um SaaS multi-tenant)

| Componente | Armadilha | Mitigação |
|---|---|---|
| **tldraw** | SDK 4.0 exige licença comercial (~US$6k/ano) + marca d'água | usar **React Flow (MIT)**; tldraw só p/ whiteboard opcional |
| **LangGraph `langgraph-api`** | runtime de servidor = **Elastic v2** (proíbe SaaS hospedado a 3os) | usar peças **MIT** (grafo + checkpointer) + **servidor próprio** |
| **CrewAI** | multi-tenant/RBAC só no **AMP pago**; Python-only | se usar, fronteira de tenant é sua |
| **Composio** | SDK MIT mas **backend fechado guarda credenciais** | **Nango** (self-host, Elastic v2) — você é dono dos tokens |
| **Convex** | FSL (cláusula non-compete) | OK como app-backend, não redistribuir |
| **LiteLLM** | **incidente supply-chain (mar/2026, v1.82.7/8)** | pin de versão + verificar hash + Docker self-host pinado; ou **Portkey (Apache-2.0)** |
| **Daytona / Coder** | core **AGPL-3.0** (obrigação de disclosure se modificar+distribuir) | usar como serviço; ou **E2B (Apache-2.0)** |
| **Zep platform / Turbopuffer** | SaaS/proprietário | usar **Graphiti (Apache)** / **pgvector** |
| **Arize Phoenix** | **Elastic v2** (proíbe revender como serviço hospedado) | **Langfuse** (MIT, sem cláusula no-SaaS) |
| **Inngest / Restate / Trigger.dev** | SSPL / BSL 1.1 / sem SDK Python | **Temporal** (MIT, TS+Python) |
| **Helicone / Pipedream** | adquiridos (Mintlify / Workday) → manutenção/roadmap incerto | Langfuse (obs) / Nango (conn) |

**Regra:** toda escolha primária acima é permissiva (MIT/Apache/Postgres) e self-hostable — coerente com a tese "dono do substrato / proof-gated".

---

## 6. Ordem de construção (fast path)

- **Fase 0 — Fundação.** Monorepo Turborepo. Schema canônico de Agente/Time (estende AGENTS.md+SKILL.md). Postgres+pgvector+RLS. `atomic-edit` empacotado como MCP built-in (L0). LiteLLM proxy de pé (L1).
- **Fase 1 — Esqueleto vertical (1 agente E2E).** L1 gateway + L3 definição + L0 substrato + L5 trace + UI chat (assistant-ui). Entregável: 1 agente de code-review, BYOK, com Caveman plugado, editando com prova, trace visível.
- **Fase 2 — Times & workspace.** L4 (LangGraph + Temporal: handoffs/HITL) + L7 canvas (React Flow) + L2 memória (Letta+Graphiti) + multiplayer (Yjs). Entregável: montar/observar/**editar** um time ao vivo.
- **Fase 3 — Exec & remote-control.** Sandbox microVM (E2B/Daytona) + terminal (xterm.js↔node-pty) + Playwright. Entregável: agente-dev com terminal isolado por tenant.
- **Fase 4 — Comunidade.** L6 marketplace proof-gated (Docker MCP Catalog + SKILL.md/A2A). Entregável: publicar/instalar agente da comunidade com prova.
- **Fase 5 — Memória neurosimbólica.** VSA/atenção provada sobre Letta+Graphiti (a aposta de pesquisa).

---

## 7. Integração do atomic-os (L0)
Como `atomic-edit` **já é um servidor MCP**, ele entra como **mais uma tool MCP** que qualquer agente da ADE pode equipar — sem reescrita. Toda ação de código de qualquer agente pode ser roteada pelo envelope provado (byte-floor) → a ADE inteira ganha o piso de "não-quebra-provado" sem acoplar o produto ao substrato. É o moat conectado por um plugue padrão.

---

*Fontes consolidadas dos 6 relatórios de pesquisa (jun/2026): protocolos (AAIF/Linux Foundation), LangGraph/CrewAI/OpenAI Agents SDK (PyPI/npm/GitHub), LiteLLM/Portkey/OpenRouter, React Flow/assistant-ui/Yjs/tldraw, E2B/Daytona/Firecracker, Letta/Graphiti/pgvector/Qdrant/Nango. URLs nos relatórios individuais.*
