# Processo — A3: OpenSpec (real) + multi-agente

Braço **A3**: OpenSpec de verdade (CLI 1.4.1) + técnica **multi-agente** (cada papel
especialista rodou como **subagente com contexto isolado**, invocado via a ferramenta de
subagentes; o executor atuou como **Orchestrator**). O plano de testes mora no `design.md`
deste run (cópia do `design.md` do change `a3-veridia-test-plan`).

## 1. Entendimento do Problema

Veridia é um gateway de decisão de crédito com três superfícies sob teste: a **API síncrona**
`POST /v1/assessments` (3 motores HTTP, o `engine-bureau` lento ~1.2s e instável, SLA apertado
p95<1.5s/p99<2.5s), o **consumer Kafka** (pico noturno de ~2M msgs em ~40 min ≈ 833 msg/s vs
~50 msg/s normal) e o **Oracle** (tabela de 400M linhas, pool de 20 conexões/pod
compartilhado entre API e consumer). O objetivo do braço é produzir um plano K6 (E2E, load,
spike, stress, volume, soak) ancorado nos fluxos, SLAs e nas tensões T1–T5, com critérios de
aceite numéricos — não genérico.

## 2. Propostas por Papel

Cada papel foi um **subagente isolado**. Resumo conciso do que cada um propôs:

- **QA Planner:** priorizou os 5 fluxos E2E colocando **F1 (recusa por fraude)** e **F2
  (fallback do bureau) acima de F3 (aprovação direta)** por risco de *decisão incorreta*.
  Entregou matriz de cenários com bordas (bureau no timeout/5xx, bureau no limite do p95,
  múltiplos motores degradados, idempotência simples e concorrente, auditoria sob contenção).
  Para T1–T5, mapeou o **risco de negócio** de cada tensão (sem dizer "como"): T1 bureau quase
  estoura o SLA e o fallback é a última defesa; T2 dois regimes de volume; T3 pool Oracle pode
  derrubar a API por contenção interna; T4 meta de lag vaga não é auditável; T5 reuso de
  `customerId` mascara a escrita. Levantou 8 pendências para humano.

- **Backend/Integration Engineer:** rankeou os gargalos — **#1 `engine-bureau`** (p95 1.2s
  vs SLA 1.5s, sem cache/retry, timeout de 2s estoura o p99); **#2 Virtual Threads**
  amplificando in-flight (Little's Law) e expondo limites de conexão mais cedo; **#3 pool
  Oracle** 20/pod compartilhado no pico (o mais *mascarável*); #4 broker/12 partições como
  teto de paralelismo; #5 tabela de 400M encarecendo o SELECT de idempotência. Classificou o
  pico Kafka (T2) como **spike de arrival-rate com componente volume/soak** (conta: 2M/40min ≈
  833 msg/s, 16,7× o baseline, ~69 msg/s/partição). Exigiu mock do bureau com **latência/erro
  injetados**, seed de 400M e massa única. Antecipou divergência com o QA Perf sobre T3, T1,
  T4 e paridade QA→prod.

- **QA Performance Engineer** (recebeu as saídas do Planner e do Backend — dependência real):
  selecionou os **6 tipos de teste** com gatilho no alvo, despriorizando stress isolado da API
  (em favor do stress com sobreposição) e fundindo volume Kafka com soak. Propôs executores de
  **arrival-rate (modelo aberto)** para a API (VUs presos num closed-model mascarariam o SLA);
  dimensionou in-flight ~390 por Little's Law. Thresholds por teste ancorados no SLA. Para
  **T4**, decompôs "15 min" em **lag<180s, backlog<750k msgs, drenagem<900s** (hard cap),
  convergindo com os ~750k do Backend. Para **T3**, cenário de **sobreposição** API+Kafka com
  métrica **Δp95_api ≤ 150ms**. Para **T5**, `traceId`/`customerId` únicos (≥95%) + 5% de
  reuso proposital medido à parte. Marcou 5 divergências (D1–D5) para o Orchestrator.

## 3. Pontos de Discussão

Divergências que emergiram entre os subagentes (marcadas pelo QA Perf e/ou antecipadas pelo
Backend), levadas ao Orchestrator:

- **D-reuso de massa (T5):** QA Perf propôs 5% de reuso proposital de `traceId`; a leitura
  conservadora (massa só-única) evitaria qualquer contaminação da métrica de escrita.
- **D-modelo de carga:** arrival-rate (open-model) vs VUs/Little (closed-model) para a API.
- **D-meta de lag (T4):** lag operacional (180s) vs backlog em mensagens (~750k) como métrica
  primária.
- **D-Δp95 (T3):** o limite de ≤150ms para o impacto da sobreposição precisava ser acordado,
  não assumido.
- **D-SLA no pico (B/700 req/s):** o alvo só declara SLA para ~300 req/s; não há SLA formal
  para o pico de 700.
- **T3 isolado vs sobreposto:** Backend e QA Perf convergiram que teste isolado dá *falso
  verde*; a divergência era só sobre tornar a sobreposição **obrigatória** no aceite.

## 4. Resolução e Papel do Orquestrador

Como Orchestrator, consolidei cada divergência no `design.md`:

- **Modelo de carga →** adotado **arrival-rate (open-model)** (D2 do design): sob bureau
  lento, VUs presos derrubariam a taxa de chegada e esconderiam a violação de SLA. VUs de
  apoio dimensionados por Little's Law (~390 in-flight).
- **T4 →** mantidos os **três** thresholds (lag<180s **e** backlog<750k **e** drenagem<900s)
  em vez de escolher um só (D4); `C_max` (capacidade de consumo) marcado como **bloqueante** e
  pendência Q2.
- **T3 →** sobreposição **obrigatória** como pré-condição de aceite do pico; testes solo
  declarados insuficientes; Δp95_api ≤ 150ms adotado como meta, com alerta de não-linearidade
  QA(80)→prod(160) (D5, Q3).
- **Massa (T5) →** mantidos os **5% de reuso**, porém **segmentados por tag** e medidos à
  parte, para dar cobertura de idempotência sob carga sem contaminar a métrica de escrita
  (D8); fração exata vira pendência Q6.
- **Bureau (T1) →** mock com **latência/erro injetados** fixado como requisito (D6); score/
  fraud podem ser fixos.
- **SLA no pico →** não inventei número; registrado como decisão de negócio (Q1).

Cada tensão foi discutida por **pelo menos dois papéis**: T1 (Planner+Backend+Perf), T2
(Backend+Perf), T3 (os três), T4 (Backend+Perf, consolidado pelo Orchestrator), T5
(Planner+Backend+Perf).

## 5. Pendências

Transcritas para `design.md` (Open Questions) e `tasks.md` (6.2):

- **Q1:** SLA aceitável da API durante o spike de 700 req/s (não declarado). Decisão de negócio.
- **Q2:** `C_max` (capacidade máxima de consumo, msg/s) — bloqueia metas de backlog/drenagem.
- **Q3:** Extrapolação da contenção do pool de QA (4 pods/80 conexões) para prod (8 pods/160),
  comportamento não-linear.
- **Q4:** Regra de veredito quando múltiplos downstream degradam simultaneamente.
- **Q5:** Atomicidade da idempotência / constraint única em `traceId` (aceite de F4 concorrente).
- **Q6:** Fração de reuso de massa (5%) é o valor certo?

**Isolamento de contexto dos subagentes:** declarado, **não verificado**. A invocação via a
ferramenta de subagentes executou e retornou resultado coerente para os três papéis, mas não
há prova por inspeção de que cada subagente rodou com contexto genuinamente separado do
principal. Tratar como declarado (ver seção 6). Não houve degradação para multi-role: as três
invocações funcionaram.

## 6. Proxies de Esforço (factual)

- **Subagentes invocados:** **3** (QA Planner, Backend/Integration Engineer — em paralelo; QA
  Performance Engineer — depois, recebendo as saídas dos dois).
  - **Como foram invocados:** via a ferramenta de subagentes (`invoke_sub_agent`, preset
    `general-task-execution`), cada um com prompt contendo **apenas** seu papel
    (`agents/<papel>.md` resumido) + o alvo resumido; o Planner e o Backend sem o raciocínio um
    do outro; o Perf recebendo as saídas consolidadas do Planner e do Backend (dependência
    real). Isolamento **declarado, não verificado**.
- **Chamadas de ferramenta (aprox.):** ~18 — leituras de contexto (3), CLI OpenSpec
  (new/status/instructions/validate ≈ 4), 3 invocações de subagente, 9 escritas de artefato
  (proposal + 6 specs + design + tasks), cópia/contagem e escrita deste process.
- **Tamanho dos artefatos (linhas):**
  - `proposal.md`: 62
  - `specs/` (6 arquivos): 203 no total (e2e 57, sync-api 31, kafka 34, oracle 28, mass 29,
    bureau 24)
  - `design.md`: 209  *(o plano de testes)*
  - `tasks.md`: 37
  - `process.md`: 135
- **Nº de turnos deste braço:** 7  *(ver seção 7)*

## 7. Custo e Tempo Reais (preenchido pelo usuário, da interface)

- **Nº de turnos deste braço:** 7
- **Est. Credits Used (soma dos turnos):** 12.73
- **Elapsed time (soma dos turnos):** 12m 22s
