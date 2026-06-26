# Memória do projeto

## Contexto macro: este repo é UMA peça de uma A.D.E
Estamos construindo uma **A.D.E — Agent Development Environment**: um ambiente omni-channel, modular e community-driven para criar/gerenciar múltiplos agentes e times de agentes (inspirado em BridgeMind, Cursor, Claude Code, Codex). `atomic-os-swebench` (este repo) é **uma ferramenta dentro da ADE**: o **substrato de garantia / kernel de confiança** (proof-carrying), não o produto inteiro.

> **Tese estratégica da ADE:** o mercado de orquestração de agentes está lotado. Nosso diferencial NÃO é orquestrar — é o **substrato provado** (o `atomic`). Posicionamento: *"a ADE onde a saída de cada agente nasce provada"*. Reusar estado-da-arte (UI, gateway, LangGraph, memória, observabilidade); só o substrato é moat. Marketplace deve ser **proof-gated** ("só propaga o que nasce provado").

**Deep research completo (temos/falta/organização em 8 camadas + roadmap):** ver [`ADE-DEEP-RESEARCH.md`](./ADE-DEEP-RESEARCH.md).
**Stack técnico de construção (6 frentes pesquisadas: stack por camada + modelo de dados + processos + armadilhas de licença + ordem de build):** ver [`ADE-STACK.md`](./ADE-STACK.md).

### Stack default decidido (jun/2026, OSS-first, multi-tenant)
- **Topologia:** monorepo Turborepo — `apps/web` (Next.js) + `apps/api` (**Hono**) + `services/agent-worker` (Python FastAPI); ponte durável = **Temporal**.
- **L7 UI:** React Flow (canvas) · assistant-ui (chat) · Sigma.js (grafo memória) · Yjs (multiplayer) · AG-UI/SSE. ⚠️ tldraw = armadilha de licença.
- **L4 Orquestração:** LangGraph (peças MIT + servidor próprio — `langgraph-api` é ELv2) + Temporal (durável).
- **L2 Memória:** Letta + Graphiti (grafo temporal) sobre pgvector→Qdrant.
- **L1 Gateway:** LiteLLM (pin de versão) ou Portkey (Apache-2.0).
- **Exec:** E2B/Daytona (microVM/tenant) · xterm.js↔node-pty · Playwright.
- **Backend:** Supabase Postgres+pgvector (RLS) · Better Auth · Stripe Meters · Langfuse (⚠️ Phoenix=ELv2) · secrets KMS envelope.
- **Protocolos dia 1:** MCP + A2A + AGENTS.md + SKILL.md + AG-UI. `atomic-edit` (MCP) = tool built-in L0.

## O que é o `atomic-os` (resumo verificado no código)
Substrato de edição de código byte-exato **proof-carrying** sob a lei *"only the proven exists"*: nada toca o disco sem prova; auto-modifica só via portão de prova (`expand_self`); aprende acumulando operadores verificados (`weights_admit.py`).
- **Engine:** `core/atomic-edit/` — servidor **MCP** (`@modelcontextprotocol/sdk`, stdio, Zod), **123 tools**, **266 gates**, 29 langs (tree-sitter). Entry: `server.ts`. Padrão de toda tool: guarda de contenção → guarda de sha → menor mutação → **prova de remoção** (`proofOfIncorrectness`) → `commit` valida-antes-de-escrever.
- **6 MCPs irmãos** (`vendor/mcp-siblings/`): memory, swarm, sentinel, dashboard, edit-bench, edit-evolution.
- **Contribuição formal real:** álgebra **(a)+(e)** (byte-default invertido + confluência preservadora de obrigação), machine-checked em Z3 + Lean (`formal/`).
- **Estado honesto (do próprio paper):** placar amplo = 1 vitória/9 empates/2 derrotas vs nativo (paridade-a-mais-fraco); transferência de operadores = NULL; "lift" vem do GATE, não dos pesos. Falta: prova externa em escala (SWE-bench on/off — `EXTERNAL_BLOCKED`), casca de produto.

## Como testar (verificado ao vivo, jun/2026)
- `python3 core/agent/atomic-full-ab/local-loop/weights_admit.py --selftest` → `ALL LAWS HOLD: True`.
- `cd core/atomic-edit && npm install && node paradigm-verify.mjs` → **15/16 green** (P3 precisa de privilégio de kernel; Lean = SKIP honesto).
- `pip install z3-solver && python3 formal/atomic-algebra/confluence_z3.py` → **ALL GREEN**.

## Branch de trabalho
Desenvolver em `claude/confident-knuth-9g7iz3`. Commit/push só quando o usuário pedir explicitamente.

## Recomendações em aberto (minha análise)
1. Comprar o número externo (ablation SWE-bench on/off, modelo fixo).
2. Medir no eixo certo: regressão / estado-quebrado / patch-aplica-de-primeira (onde o substrato vence), não Pass@1.
3. Empacotar `atomic-edit` como MCP instalável (vira o kernel L0 da ADE + resolve a casca de produto).
4. Cortar o excesso de manifestos (.md grandiosos) — mostrar honestidade > escrever sobre ela.
