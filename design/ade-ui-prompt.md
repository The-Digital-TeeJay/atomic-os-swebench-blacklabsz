# Prompt de design — ADE (cole no Claude Design)

> Cole tudo abaixo. As seções marcadas com 🎛️ são "botões de ajuste" — troque antes de colar pra mudar a direção.

---

Você é um designer de produto sênior. Crie a **tela principal de um Agent Development Environment (ADE)** — uma plataforma SaaS onde qualquer pessoa constrói, orquestra e administra **times de agentes de IA**. Pense em "mission control para agentes de IA": **Linear** (precisão calma) × **Figma/FigJam** (canvas + multiplayer) × **editor de nós tipo n8n/React Flow** × **Raycast** (paleta de comandos). Alvo: desktop, alta fidelidade, pronto pra produção.

## Conceito central (a grande ideia)
É **uma UI só**. O fundo é um **canvas/grafo infinito** onde o usuário vê e administra cada time, agente e tarefa. Sobre o canvas flutua um **chat arrastável** que pode conversar com **três alvos**: o **Sistema** (orquestrador), o **líder de um time**, ou **um agente individual** de qualquer time. Clicar num agente no canvas abre um **inspector** lateral e re-mira o chat nele. Tudo é observável e editável ao vivo.

## Estrutura de layout (regiões)
1. **Top bar** (56px): logo "ADE" + seletor de workspace; alternador de visão (◇ Grafo · ▦ Canvas · ≣ Tabela); busca/⌘K; **avatares de presença** (multiplayer); botão **▶ Rodar fluxo**.
2. **Rail esquerda** (~230px): lista de **Times** (com bolinha de status e contagem), botão **＋ Novo time**, e seção **Biblioteca** (Skills, Tools/MCP). Rodapé: selo do substrato "⛓ atomic provado".
3. **Canvas central** (resto da tela): grafo com zonas de time, nós de agente, edges e tarefas. Dot-grid sutil. Minimapa + controles de zoom nos cantos.
4. **Inspector** (slide-over à direita, ~350px): abre ao selecionar um agente/time.
5. **Chat flutuante**: sobre o canvas, centralizado embaixo, arrastável e redimensionável.

## Componentes (detalhe)

**Zona de time** — container translúcido arredondado agrupando os agentes do time, com borda tracejada na cor do time. Header flutuante em pílula: nome do time + "líder X" + status (⚡ ativo / ▶ rodando / ○ ocioso). Colapsável.

**Card de agente (o nó)** — vidro (glassmorphism), ~160px largura. Contém:
- avatar com gradiente + iniciais;
- nome + papel (ex.: "Aria · Art Director");
- **badge do modelo** colorido por provedor (Claude / GPT-5.5 / Gemini / DeepSeek) — cada agente sua própria LLM (BYOK);
- **bolinha de status** pulsante: trabalhando (ciano) / ocioso (cinza) / erro (vermelho) / concluído (verde);
- **medidor de contexto em tempo real** (barra de orçamento de tokens, ex.: "61% · 8k"; fica âmbar/vermelha quando alta);
- **chips de skills/tools** (ex.: Caveman, RKT, Figma MCP) e um selo verde **"atomic ✓"** quando a edição é provada;
- **líder** marcado com coroa 👑 + anel dourado;
- **handles** de entrada/saída pra conectar nós;
- estados: hover (leve elevação), selecionado (glow no accent).

**Edges (conexões)** — curvas bezier com gradiente. Direcionais (seta). Tipos: handoff normal; **ativo** = fluxo animado + glow ciano (quando o agente está passando trabalho agora); **entre times** = tracejado dourado (handoff A2A entre times).

**Cards de tarefa** — cartões pequenos com título, dono, status e barra de progresso (ex.: "Landing redesign · Design→Dev · 62%"). Podem ficar numa "lane" ou ancorados a um agente.

**Chat flutuante (peça-chave)** — painel de vidro sobre o canvas, com:
- alça de arrastar no topo;
- **SELETOR DE ALVO**: pílula "Falando com: [alvo ▾]" → dropdown agrupado: **🜂 Sistema (ADE)** (orquestrador, vê todos os times); depois, por time, o **👑 líder** e cada **agente** individual;
- stream de mensagens com **tokens em streaming**, cartões de **tool-call**, e cada mensagem mostrando **quem respondeu** (avatar + badge do modelo);
- composer com anexo e botão enviar;
- pode minimizar pra uma bolha.

**Inspector** (slide-over direito) — abas **Config · Contexto · Trace**:
- **Config:** modelo (BYOK, via OpenRouter), system prompt (editável), skills & tools, temperatura.
- **Contexto** (o diferencial): medidor da janela em tempo real; lista de **blocos de memória em contexto** (pinados/editáveis, com a fonte: `core: persona`, `fato: … (Graphiti, válido agora)`, `histórico (compactado −63%)`); itens **despejados**; e um **slider de time-travel** pelos checkpoints da run.
- **Trace:** passos da execução (raciocínio, tool-calls, handoffs) com recibo "atomic ✓".

## 🎛️ Linguagem visual (ajuste aqui)
- **Tema:** dark "mission control" — fundo quase-preto (#0a0b12), painéis em **glassmorphism** (blur + translucidez), profundidade por camadas. *(Alternativa: variante clara "daylight".)*
- **Cores de accent:** **violeta elétrico #7c6cff** (primária) + **ciano #22d3ee** (secundária) + **dourado #f5c451** (líderes/prova). Badges de modelo tingidos pelo provedor. *(Troque o accent se quiser outra identidade.)*
- **Tipografia:** Inter (ou similar geométrica), pesos 600–800 em títulos.
- **Formas:** cantos 12–18px, sombras suaves, dot-grid no canvas.
- **Mood:** premium, calmo, espaçoso — **não** poluído. Densidade confortável. 🎛️ *(ou "compacta/densa" se preferir mais informação por tela.)*

## Estados e interações
- pan/zoom no canvas; arrastar nó; **arrastar de um handle** cria uma conexão (handoff);
- **clicar num agente** → abre inspector + re-mira o chat nele;
- duplo-clique no canvas → criar novo agente; botão → criar time;
- arrastar o chat; alternar o alvo do chat;
- **agente trabalhando** = pulso + edges ativos; **erro** = vermelho; **HITL** = agente pausado aguardando aprovação, com botões aprovar/rejeitar destacados;
- **multiplayer**: cursores/avatares de outros usuários;
- **⌘K** abre paleta de comandos.

## Entregável
Uma tela web de **alta fidelidade**, responsiva (desktop-first), num **estado já populado de exemplo**: **3 times** (Design, Dev, Research), ~**10 agentes** com modelos variados, 2–3 tarefas, **um handoff ativo animado**, o chat flutuante mirando o Sistema, e o inspector fechado (mostre também uma variação com ele aberto num agente). Mostre os estados de status (trabalhando/ocioso/erro/done) e o selo atomic ✓ em ao menos um agente.

## Referências de inspiração
Linear · Figma/FigJam · n8n / React Flow · Raycast · um painel de ops "mission control".
