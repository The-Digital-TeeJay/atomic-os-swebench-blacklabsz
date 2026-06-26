# A.D.E — Agent Development Environment · Deep Research
> O que TEMOS · O que FALTA · Organização proposta · **+ o substrato `atomic` embutido (seção A)**
> Escopo: a ADE é o produto-mãe; `atomic-os` é **uma** ferramenta dentro dela (o substrato de garantia / kernel de confiança L0). **Documento auto-contido** — a seção A embute todo o conhecimento do atomic para o novo repo da ADE não depender do repo `atomic-os-swebench`. Documento vivo. Conferido em repo + rodado ao vivo + estado-da-arte (jun/2026).

---

## 0. A tese estratégica (leia isto primeiro)

O mercado de "ambiente para construir/orquestrar times de agentes" **já está lotado**: BridgeMind (BridgeSpace/BridgeSwarm), agentsmesh, Composio Agent Orchestrator, overstory, Conductor, Claude Squad, Vibe Kanban — e as big techs (Cursor, Claude Code, Codex) caminhando pra lá. Construir "mais um orquestrador de agentes" é entrar num oceano vermelho.

**O diferencial da nossa ADE não pode ser a orquestração — tem que ser o SUBSTRATO.** Todos os outros orquestram agentes que *torcem* para acertar (validam por teste e tentativa). Nós orquestramos agentes cujo trabalho **carrega prova** (verificação determinística pré-disco — o `atomic`). 

> **Posicionamento:** *"A ADE onde a saída de cada agente nasce provada."* O `atomic-os` deixa de ser "o produto" e vira o **kernel de confiança** da ADE — a única peça que ninguém no mercado tem. Tudo o mais (UI, gateway, memória, marketplace) nós **reusamos/adotamos** do estado-da-arte; só o substrato é nosso moat.

Esse mesmo princípio resolve o problema nº 1 de todo marketplace de agentes — **confiança em componente da comunidade**: nosso marketplace é **proof-gated** (*"só propaga o que nasce provado"*), mecânica que o `atomic` já implementa para hosts.

---

## A. O SUBSTRATO ATOMIC EMBUTIDO (L0) — conhecimento completo e auto-contido
> Esta seção embute **TODO** o conhecimento do `atomic-os` para que o repositório da ADE seja **auto-contido** (não dependa do repo `atomic-os-swebench`). Tudo abaixo foi conferido no código e **rodado ao vivo** (jun/2026). Onde é provado por número, está dito; onde é fronteira não-provada, também.

### A.1 O que é, em uma frase — e a lei
Um **substrato de edição de código byte-exato e proof-carrying**: um motor (as "mãos") dirigido por um modelo (o "cérebro"), governado por uma única lei aplicada recursivamente a 5 camadas — **"só existe o que está provado"** (*only the proven exists*):

| Camada | A mesma lei | Mecanismo |
|---|---|---|
| **Disco** | nenhum byte não-provado é escrito | floor de convergência pré-disco (P1) |
| **SO/Kernel** | o kernel bloqueia escritas sem envelope (inescapável) | `byte-guard-kernel` (eBPF LSM / Endpoint Security) + proof-token |
| **Auto-modificação** | o sistema só se altera de formas que prova não quebrar | `expand_self` + catraca monotônica |
| **Memória** | só soluções provadas viram aprendizado, em forma geral | admissão de operadores proof-carrying (3 leis) |
| **Afirmações** | nada é "pronto" sem número; derrota é registrada | disciplina anti-fachada + medição A/B |

**A inversão central:** o espaço de ação certo não é o "patch de texto", é a **transação de intenção provada** — o agente declara o resultado pretendido, a camada calcula a menor mutação-byte fiel, **prova antes de tocar o disco**, e só então materializa. AST/LSP/diff são órgãos internos, nunca a interface. *"Diff humano é saída, não entrada."*

### A.2 Arquitetura (de baixo pra cima)
1. **Kernel de inescapabilidade** — o SO recusa escritas não-envelopadas (proof-token: `.atomic/write-tokens/<pid>-<sha>.json`; sem token → escrita bloqueada). Em tráfego real bloqueou **1.088 mutações nativas, 0 liberadas silenciosamente**.
2. **Engine `atomic-edit-mcp` v4.0.0** — servidor MCP, **123 tools**, **266 gates**, **29 linguagens** (tree-sitter).
3. **Kernel `expand_self`** — única via legal de mudar o atomic, sob catraca monotônica (~583–595 promoções).
4. **6 MCPs irmãos** — memory, swarm, sentinel, dashboard, edit-bench, edit-evolution.
5. **Camada de agente** + loop de governança de 7 fases (PLAN→INVESTIGATE→PROPOSE→VALIDATE→COMMIT→VERIFY).
6. **Loop A/B + motor de pesos** (mede, aprende, generaliza).

### A.3 O engine MCP (as "mãos") — como foi construído
- **Stack:** TypeScript sobre `@modelcontextprotocol/sdk` (SDK oficial), transporte **stdio**, schemas **Zod**, AST via **tree-sitter** (15 gramáticas nativas + web-tree-sitter wasm) + **ts-morph**. Entry: `server.ts` (orquestrador) → 33 módulos `server-tools-*.ts`, sob um *gate de tamanho de arquivo*.
- **O pipeline de toda tool de edição (o código real, ex. `atomic_read_and_edit`):**
  1. `resolveSafeTarget` — guarda de contenção (não escapa do repo / não toca protegido);
  2. `guardSha` — concorrência otimista (o arquivo não mudou debaixo de você);
  3. `replaceText` — calcula a **menor mutação-byte fiel**, preserva o resto;
  4. `requireNegativeProofForRemovedBytes` — **o byte-default invertido**: se a edição **remove bytes**, exige o parâmetro `proofOfIncorrectness`, senão **recusa**;
  5. `commit` — valida a árvore **antes** de tocar o disco, escreve atômico (sem arquivo rasgado), grava trace/recibo.
- Parâmetros universais: `preview` (dry-run), `verify: typecheck|lint`, `expectedSha256`, `lock`.
- Multi-host: shim que reescreve schemas pro Codex; launcher único supervisionado (`atomic-edit-mcp-launcher.sh`) pra onde todos os hosts apontam.

### A.4 A contribuição formal — a álgebra **(a)+(e)** (o que é genuinamente inédito)
Dois mecanismos fundidos numa propriedade só:
- **(a) byte-default invertido** — bytes corretos-por-construção são imutáveis a ações negativas; apagar exige *disproof recomputado* sobre os bytes removidos (o gate **recomputa**, não confia em digest).
- **(e) álgebra de comutação módulo-invariante** — duas edições verificadas comutam sse `mod/read` disjuntos.
- **A unificação:** o read-closure dos gates e o `readLoci` do disproof são o **mesmo objeto** → um merge que comuta preserva *tanto o veredito positivo quanto a obrigação negativa* (não dito por OT/CRDT/Darcs/Pijul/Unison/PCC).
- **Provas (machine-checked):** Z3 (todas as configs) + indução N-ária em **Lean 4**; refinamento runtime (73.728×2 configs); solidez externa sobre **169.171** pares reais (zod/type-fest/zustand) com **0 vereditos não-sólidos**.
- **Propriedades P1–P10:** Floor, Soundness, Completeness, Closure, Monotonic admission, Ratchet, **Confluência preservadora de obrigação (P7)**, Disproof recomputável (P8), Truth-funnel (P9), Convergência byte-positiva (P10).

### A.5 A camada de aprendizado — "pesos" proof-carrying (NÃO é rede neural)
`weights_admit.py` (determinístico, CPU-only, sem LLM). Um "peso" = operador de resolução generalizado `{class, trigger, strategy, instances[], proof_n}`. 3 leis: **(1)** capturar N não um (absorve por compressão MDL); **(2)** nascer sob necessidade; **(3)** fidelidade monotônica. Estado: **8 operadores** + 45–47 demolições `CLASS-*`. Tem **VSA** (Vector Symbolic Architecture) real com reforço por uso — a semente neurosimbólica que a ADE reaproveita na camada de memória/atenção (L2).

### A.6 Os 6 MCPs irmãos
`atomic-memory` (ledger de intenção semântica, recibo SHA-256 — base da "memória com prova"), `atomic-sentinel` (daemon de tarefas falhas/locks), `atomic-swarm` (orquestração multi-agente), `atomic-dashboard` (observabilidade), `atomic-edit-bench` (medição), `atomic-edit-evolution` (loop A/B).

### A.7 Estado verificado ao vivo (o que rodei) + o teto honesto
**Verificado ao vivo (jun/2026):**
- `weights_admit.py --selftest` → **`ALL LAWS HOLD: True`** (Python puro, sem deps).
- `npm install && node paradigm-verify.mjs` → **15/16 verde** (1 RED = P3, gates de recurso do SO que exigem privilégio de kernel; 1 SKIP = Lean ausente, honestamente não-contado).
- `python3 formal/atomic-algebra/confluence_z3.py` → **ALL GREEN** (teorema da álgebra machine-checked).

**O teto honesto (do próprio paper — anti-fachada):**
- Placar amplo: **1 vitória / 9 empates / 2 derrotas** vs worker nativo (paridade-a-ligeiramente-mais-fraco). **Dominância geral é falsificada por número.**
- **Transferência de operadores aprendidos = NULL** (em isolamento, o peso adicionou zero valor marginal sobre o gate). O "elevador" comprovado hoje é o **gate** (verificação + iteração forçada), não os pesos.
- Rice é **contornado, não derrotado**: a álgebra decide interferência sobre read-closure decidível; correção de **intenção** ("faz o que o humano quis") está **fora de escopo por construção**.
- Faltam: prova externa em escala (SWE-bench on/off, modelo fixo — `EXTERNAL_BLOCKED`) e a casca de produto.

### A.8 Como o atomic integra na ADE como L0 (o ponto crucial)
- **Plugа de fábrica:** `atomic-edit` **já é um servidor MCP** → na ADE entra como **mais uma tool MCP** que qualquer agente equipa, **sem reescrita**. Agentes de código (time Dev) equipam; agentes de design podem não.
- **Envelope provado transversal:** toda ação de código de qualquer agente (editar/criar/executar) pode ser **roteada pelo byte-floor** → a ADE inteira ganha o piso "não-quebra-provado" sem acoplar o produto ao substrato.
- **Marketplace proof-gated (L6):** reusa a mecânica `unify-hosts` / propagação proof-gated → *"só propaga o que nasce provado"* resolve a confiança em componente da comunidade.
- **Memória com prova (L2/L5):** o recibo SHA-256 do `atomic-memory` vira o mecanismo de *"contexto com prova"* — cada decisão de admissão/eviction de memória logada e verificável.
- **Divisão honesta de papéis:** o atomic garante **integridade** (não quebrar); a correção de **intenção** fica com a camada de QA-contract/spec (AGENTS.md/SKILL.md + verificadores executáveis). Os dois juntos = "fez a coisa certa **e** não quebrou nada".

### A.9 Como portar para o novo repo da ADE (concreto)
- **Fonte canônica:** `core/atomic-edit/` (TypeScript, MCP, stdio). Requer **Node 22**.
- **Opções de embute (do mais simples ao mais limpo):**
  1. **MCP externo apontado:** o agent-runtime da ADE aponta seu cliente MCP para o launcher `atomic-edit-mcp-launcher.sh` (como `.mcp.json` faz hoje). Zero código novo.
  2. **Vendored / git submodule:** copiar `core/atomic-edit` para `services/atomic-edit` (ou submodule) no monorepo da ADE; buildar no CI.
  3. **Pacote instalável:** empacotar como npm/MCP publicável (`npx atomic-edit`) — resolve embute + a casca de produto de uma vez (recomendado a médio prazo).
- **Build:** `npm install && npm run build` → `dist/`. Launcher resolve Node + supervisiona (crash-recovery, dist-lkg fallback).
- **Como o agente usa:** equipar o servidor MCP no config de tools do agente → aparecem `atomic_edit`, `atomic_read_and_edit`, `atomic_exec`, etc. Edição de código passa a nascer provada.

### A.10 Comandos de verificação reproduzíveis (carregar pro novo repo)
```bash
# 1) o aprendizado obedece suas leis (Python puro, sem deps)
python3 core/agent/atomic-full-ab/local-loop/weights_admit.py --selftest   # → ALL LAWS HOLD: True
# 2) a bateria de provas P1–P10 (precisa build)
cd core/atomic-edit && npm install && node paradigm-verify.mjs             # → 15/16 verde
# 3) a álgebra formal (precisa z3-solver)
pip install z3-solver && python3 formal/atomic-algebra/confluence_z3.py    # → ALL GREEN
```

---

## 1. O cenário (estado-da-arte) — com quem competimos e de quem copiamos

### 1.1 ADEs / orquestradores diretos (a referência de produto)
- **BridgeMind** — o mais próximo da nossa visão: **BridgeSpace** (ADE desktop, até 16 agentes em paralelo, kanban, editores, multi-pane terminal), **BridgeMCP** (camada de contexto compartilhado entre Cursor/Claude Code/Windsurf/Codex), **BridgeSwarm** (agentes como time de engenharia), **BridgeVoice**.
- **agentsmesh** — "AI Agent Workforce": AgentPods (workstations remotas, PTY sandbox, git worktree isolation), kanban, **self-hostable BYOK** (Claude Code, Codex, Gemini, Aider, OpenCode).
- **Composio Agent Orchestrator / overstory / Conductor / Claude Squad / Vibe Kanban** — orquestração de agentes paralelos de código, adapters plugáveis por runtime.
- **Big techs** — Cursor (background agents, multi-modelo), Claude Code (skills + subagents + hooks), Codex (AGENTS.md nativo). Caminham para a visão de "time de agentes", mas presos a um host.

### 1.2 Padrões (a fundação de config que vamos adotar)
- **AGENTS.md** — formato aberto, **stewarded pela Agentic AI Foundation (Linux Foundation)**, lido nativamente por 20+ ferramentas, **60.000+ repos**. É o "README para agentes".
- **SKILL.md** — spec de skills (Claude/Codex/Copilot), **progressive disclosure** (carrega nome+descrição; corpo só quando casa a task; assets só quando precisa). Frontmatter: `name, description, allowed-tools, model, context: fork, arguments`.
- **MCP** — protocolo de tools/contexto (já dominamos: `atomic-edit` = servidor MCP de 123 tools).
- **design.md / spec-driven** — spec como fonte da verdade (Spec Kit, OpenSpec, BMAD).
- ⚠️ **Nuance de pesquisa:** AGENTS.md **gerado por LLM** PIOROU resultado (−2% sucesso, +23% custo) por duplicar o óbvio. **Config tem que conter o não-óbvio**, escrito/curado — não boilerplate.

### 1.3 Frameworks de orquestração (o motor)
- **LangGraph** (seu escolhido) — grafos de estado, handoffs, checkpoints. **CrewAI** (times por papel), **AutoGen**, **OpenAI Agents SDK** (handoffs/guardrails/tracing), **Google ADK**.

### 1.4 Memória, contexto e observabilidade (a infra invisível que decide tudo)
- **Memória dual** (consenso 2026): *working memory* (sessão/intermediários) + *persistent memory* (conhecimento, decisões, preferências do time). **RAG** sobre ambos.
- **Observabilidade**: LangSmith, Braintrust (IDE-native via MCP), Arize Phoenix, Helicone, Galileo, Datadog LLM. Capturam raciocínio ponta-a-ponta, tool-calls, **handoffs agente-a-agente**, ops de memória.

### 1.5 Técnicas plugáveis de raciocínio/economia (os "equipamentos" de agente)
- **Caveman** — compressão de prompt/saída: tira filler, usa setas pra causalidade → **−40-58% tokens**, fato preservado. Bônus: brevidade força modelos grandes a errar menos (*"verbosity reverses performance"*).
- **RKT / rtk, LLMLingua** — compressão algorítmica de contexto. **InftyThink** (raciocínio longo em janelas). 
- → Na ADE, isto vira **módulo plugável por agente** ("equipe seu agente de review com Caveman p/ economia").

---

## 2. O que TEMOS (inventário honesto do repo)

| Ativo (no repo) | É primitivo de ADE para… | Maturidade | Observação honesta |
|---|---|---|---|
| **`core/atomic-edit`** (MCP, 123 tools, 266 gates) | **Substrato/garantia** (o moat) | 🟢 Alta (15/16 verde, Z3 OK) | Único diferencial real. Reposicionar como "trust kernel". |
| **`atomic-memory`** (sibling MCP) | Camada de **memória/RAG** | 🟡 Média | Hoje é "intent ledger" focado em edição; generalizar p/ memória dual de qualquer agente. |
| **`atomic-swarm`** (sibling MCP) | **Orquestração multi-agente** | 🟡 Média | Acoplado a edição; transformar em orquestrador genérico (ou trocar por LangGraph). |
| **`atomic-dashboard`** (sibling MCP) | **Observabilidade** | 🟡 Média | Cobre edição/traces; falta raciocínio/handoffs genéricos. |
| **`atomic-sentinel`** | Daemon de tarefas falhas/locks | 🟡 Média | Base p/ resiliência/auto-recovery. |
| **`atomic-edit-bench` / `-evolution`** | **Medição + loop A/B** | 🟡 Média | Disciplina de número; reusável p/ avaliar agentes. |
| **`vendor/atomic-workflows`** (engine + dezenas de `.js`) | **Workflows** | 🟡 Média | Já existe motor de workflow versionado — reaproveitar. |
| **`weights_admit.py`** (motor de pesos) | **Skill-library proof-carrying** | 🟢 Alta (selftest passa) | Admissão de operadores por prova/compressão — base do "auto-evolutivo". |
| **Agent driver** (`local_atomic_agent.py`, `swe_modal_agent.py`) | **Integração de modelo** | 🟡 Média | Dirige DeepSeek; precisa abstração multi-provider (OpenRouter/BYOK). |
| **Launcher único + `unify-hosts.sh`** | **Propagação proof-gated** | 🟢 Alta | Mecânica perfeita p/ **marketplace confiável**. |
| **`vendor/aider-atomic`** | Edição polyglot/benchmark | 🟡 | Integração com Aider (host). |
| **`vendor/coglang`** | DSL "cognitiva" / notação | 🔴 Baixa (cosmético) | Paper admite: notação não gera cognição. Não priorizar. |
| **`vendor/selfloop`** | Fitness/grounding/learning-curve | 🟡 | Maquinaria de auto-melhoria; experimental. |

**Resumo do que temos:** já possuímos **primitivos** para substrato, memória, swarm, observabilidade, workflows, aprendizado e propagação confiável — mas todos **acoplados a um único caso de uso (edição de código verificada)** e **sem casca de produto**. Temos as peças de um motor; não temos o carro.

---

## 3. O que FALTA (gap analysis vs ADE estado-da-arte)

| Capacidade ADE | Estado-da-arte tem | Nós temos | Gap | Prioridade |
|---|---|---|---|---|
| **UI / workspace omni-channel** (Chat · Canvas · Graph) | BridgeSpace, Cursor | ❌ nada | **TOTAL** — maior buraco | 🔴 P0 |
| **Config modular de agente/time** (AGENTS.md/SKILL.md ingest + editor visual) | AGENTS.md (60k repos), SKILL.md | ⚠️ parcial (lê MCP) | Falta schema 1ª classe + UI de edição | 🔴 P0 |
| **Multi-provider gateway** (LLM/key por agente: OpenRouter/Anthropic/BYOK) | agentsmesh, OpenRouter | ⚠️ driver acoplado | Abstração de provider plugável | 🔴 P0 |
| **Orquestração genérica** (handoffs, estado compartilhado, workflows) | LangGraph, CrewAI, OpenAI SDK | ⚠️ swarm+workflows acoplados | Adotar **LangGraph** como motor; ligar nossos nós | 🟠 P1 |
| **Memória dual + RAG genérica** (working+persistent por agente/time) | consenso 2026 | ⚠️ intent-ledger | Generalizar `atomic-memory` p/ qualquer agente | 🟠 P1 |
| **Observabilidade total + EDITÁVEL** (raciocínio, IO, state, handoffs) | LangSmith/Phoenix/Braintrust | ⚠️ traces de edição | A mecânica "tudo observável **e editável**" é nosso diferencial de UX — construir | 🟠 P1 |
| **Marketplace / comunidade** (compartilhar agentes/skills/tools/workflows) | (emergente) | ❌ + temos propagação proof-gated | Infra de marketplace **proof-gated** (moat) | 🟠 P1 |
| **Skills de raciocínio plugáveis** (Caveman/RKT/LLMLingua por agente) | técnicas existem soltas | ❌ | Empacotar como módulos plugáveis | 🟢 P2 |
| **Substrato de garantia** (proof-carrying) | ❌ NINGUÉM tem | ✅ **atomic** | É só **integrar** como kernel transversal | ✅ (temos) |
| **Casca de produto** (install, onboarding, docs) | todos | ❌ | Empacotamento | 🟠 P1 |
| **Suporte a "todo tipo de código"** | tree-sitter 29 langs (percepção) | ⚠️ gates profundos só TS/Py/Go | Aprofundar 2-3 langs; ser honesto no resto | 🟢 P2 |

---

## 4. A organização proposta (arquitetura em camadas)

Princípio: **adotar o máximo do estado-da-arte; só construir o que é nosso moat.** De baixo pra cima:

```
┌─────────────────────────────────────────────────────────────┐
│  L7 · WORKSPACE OMNI-CHANNEL    Chat · Canvas · Graph         │  ← CONSTRUIR (P0)
│      views observáveis E editáveis de agentes/times/IO/state  │
├─────────────────────────────────────────────────────────────┤
│  L6 · MARKETPLACE / COMUNIDADE   agentes·skills·tools·flows   │  ← CONSTRUIR (P1)
│      *proof-gated* (só propaga o que nasce provado) ← MOAT     │
├─────────────────────────────────────────────────────────────┤
│  L5 · OBSERVABILIDADE + EDIÇÃO   trace de raciocínio/handoff   │  ← atomic-dashboard + adotar (Phoenix/LangSmith)
│      tudo inspecionável e editável em tempo real ← diferencial │
├─────────────────────────────────────────────────────────────┤
│  L4 · ORQUESTRAÇÃO   times, handoffs, estado, workflows        │  ← ADOTAR LangGraph + atomic-workflows/swarm
├─────────────────────────────────────────────────────────────┤
│  L3 · DEFINIÇÃO DE AGENTE   AGENTS.md·SKILL.md·design.md       │  ← ADOTAR padrões + schema/UI (P0)
│      modelo·system prompt·tools·engine·skills(Caveman/RKT)     │
├─────────────────────────────────────────────────────────────┤
│  L2 · MEMÓRIA / CONTEXTO   working + persistent + RAG          │  ← generalizar atomic-memory (P1)
├─────────────────────────────────────────────────────────────┤
│  L1 · PROVIDER GATEWAY   OpenRouter · Anthropic · BYOK por ag. │  ← CONSTRUIR abstração (P0)
├─────────────────────────────────────────────────────────────┤
│  L0 · SUBSTRATO DE GARANTIA (atomic-os)   proof-carrying byte  │  ← TEMOS ✅ — o kernel de confiança transversal
│      toda ação de toda camada pode passar pelo envelope provado│
└─────────────────────────────────────────────────────────────┘
```

**A sacada arquitetural:** L0 (atomic) é **transversal**, não "mais uma camada empilhada". Qualquer ação de qualquer agente em qualquer camada — editar arquivo, rodar comando, mudar state — pode ser **envelopada e provada** pelo substrato. É o que faz a ADE inteira ser *"determinística o quanto possível"* (seu requisito): o não-determinismo do LLM fica contido por um piso provado.

**Mapa "agente avançado monta seu time" → camadas:**
- escolhe LLM por agente (OpenRouter/BYOK) → **L1**
- escreve system prompt + equipa tools/skills (Caveman p/ economia, RKT p/ raciocínio) → **L3**
- liga agentes em time com handoffs/estado → **L4**
- contexto individual e de grupo, memórias diferentes → **L2**
- observa e **edita** relações/IO/state ao vivo → **L5**
- publica/baixa agentes da comunidade (proof-gated) → **L6**
- tudo num workspace Chat/Canvas/Graph → **L7**
- e cada ação de código nasce **provada** → **L0** (o que ninguém mais tem)

---

## 5. Roadmap faseado (sugestão)

- **Fase 0 — Fundação & decisão de arquitetura.** Definir o schema canônico de Agente/Time (estendendo AGENTS.md + SKILL.md). Decidir: LangGraph como motor de L4. Empacotar `atomic-edit` como MCP instalável (resolve gap de casca + vira o kernel L0). Escrever `make verify` de um comando.
- **Fase 1 — O esqueleto vertical (1 agente ponta-a-ponta).** L1 gateway (multi-provider) + L3 definição + L0 substrato + L5 trace mínimo. Entregável: criar 1 agente de code-review, com Caveman plugado, rodando com edições provadas e trace observável.
- **Fase 2 — Times & workspace.** L4 (LangGraph: handoffs/estado compartilhado) + L7 (UI Chat→Canvas→Graph) + L2 (memória dual). Entregável: montar/observar/editar um **time** ao vivo.
- **Fase 3 — Comunidade.** L6 marketplace **proof-gated** reusando `unify-hosts`/propagação. Entregável: publicar e instalar um agente da comunidade com prova.
- **Fase 4 — Auto-evolução.** Ligar `weights_admit` + selfloop: agentes/skills que melhoram por prova (o loop gerar→testar→selecionar→promover, hoje só parcial).

---

## 6. Riscos & verdades honestas (anti-fachada)
- **Não virar "mais um orquestrador".** Se a UI/orquestração for o pitch, perdemos pro BridgeMind. O pitch é o **substrato provado** — não diluir.
- **Acoplamento atual.** Memory/swarm/dashboard estão amarrados a edição. Generalizá-los custa; avaliar adotar peças prontas (LangGraph, Phoenix) e manter só o atomic como nosso.
- **Config gerada por LLM piora** (dado de pesquisa). O editor de agentes deve forçar conteúdo **não-óbvio curado**, não boilerplate.
- **"Determinístico" tem teto.** O substrato garante *integridade* (não quebrar), não *intenção* (Rice). Vender o que é verdade: piso de segurança provado + orquestração observável — não "agentes infalíveis".
- **A prova externa do atomic ainda falta** (ablation SWE-bench em escala) — herdamos esse débito; vale fechar para sustentar o pitch do moat.

---

## Fontes
- BridgeMind / BridgeSpace / BridgeSwarm — https://www.bridgemind.ai/ · https://www.bridgemind.ai/products/bridgespace
- AGENTS.md (Agentic AI Foundation / Linux Foundation) — https://github.com/agentsmd/agents.md · https://www.morphllm.com/agents-md-guide
- Agent Skills / SKILL.md spec — https://agentskills.io/specification · https://www.firecrawl.dev/blog/agent-skills
- Multi-agent orchestration landscape 2026 — https://github.com/andyrewlee/awesome-agent-orchestrators · https://github.com/ComposioHQ/agent-orchestrator · https://github.com/jayminwest/overstory
- Agent observability 2026 — https://mlflow.org/articles/what-is-agent-observability-a-2026-developer-guide/ · https://www.augmentcode.com/tools/best-ai-agent-observability-tools
- Caveman / token compression — https://betterstack.com/community/guides/ai/caveman-llm/ · https://tomfranks.dev/blog/2026-04-15-caveman-compression/
- Best multi-agent frameworks 2026 (LangGraph/CrewAI) — https://gurusup.com/blog/best-multi-agent-frameworks-2026
