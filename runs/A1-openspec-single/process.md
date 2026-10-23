# Registro de processo — Braço A1 (OpenSpec real + single-role)

> Técnica: resolução direta, um único fluxo de raciocínio, sem papéis distintos.
> Processo: OpenSpec de verdade via CLI (`openspec` 1.4.1), não simulado.
> O plano de testes em si vive em `design.md`; aqui registra-se **como** ele foi produzido.

## 1. Entendimento do Problema

O alvo é **Veridia**, um gateway síncrono de decisão de crédito (Kotlin/Spring Boot, Virtual
Threads, K8s 4 pods QA / 8 prod) com três superfícies de carga acopladas: a API
`POST /v1/assessments` (orquestra `engine-score`/`engine-fraud`/`engine-bureau`, SLA p95<1.5s,
p99<2.5s), o consumer Kafka (pico noturno de ~2M msgs em ~40 min vs. ~50 msg/s normal) e o
Oracle (auditoria de ~400M linhas, pool de 20 conexões/pod compartilhado entre API e consumer).
O objetivo é um **plano de testes K6** (E2E, carga, volume, stress) ancorado nos 5 fluxos
críticos e nas 5 tensões (T1 SLA vs. bureau; T2 pico Kafka; T3 pool compartilhado; T4 meta de
lag vaga; T5 massa vs. idempotência). O foco é planejamento, não implementação de scripts.
O desafio central percebido: o SLA de 1.5s é quase todo consumido pelo bureau (p95 ~1.2s),
então os testes precisam exercer o fallback sob instabilidade e medir o acoplamento do pool
sem testar os caminhos isoladamente (o que esconderia o risco real).

## 2. Propostas por Papel

Single-role: sem papéis distintos. Consolidado num único raciocínio. As três perspectivas que
a rubrica associa a papéis foram cobertas por um só executor:

- **Planejamento (fluxos/riscos/prioridade):** priorizou os 5 fluxos E2E como gate funcional e
  mapeou cada tensão a uma capability (`e2e-decision-flows`, `sync-api-load-stress`,
  `kafka-peak-load`, `oracle-contention-volume`).
- **Performance (tipos de teste/modelagem/thresholds):** escolheu E2E+load+stress+spike+volume
  (soak parcial via pico de 40 min), modelo aberto (`constant-arrival-rate`/`ramping-arrival-rate`)
  para não mascarar saturação, e thresholds numéricos derivados direto dos SLAs.
- **Backend/integração (gargalos/viabilidade):** identificou bureau instável e pool Oracle
  compartilhado como gargalos dominantes; propôs stub parametrizável para o bureau e teste
  concorrente (API+Kafka) para expor a contenção do pool, com seed de 400M linhas como
  pré-requisito.

## 3. Pontos de Discussão

Sem debate entre papéis (single-role). Os trade-offs ponderados num só fluxo:

- **Pico Kafka é spike, volume ou soak?** Decidido: carga sustentada de alta taxa
  (`constant-arrival-rate` por 40 min) + um spike separado para o *início* do pico (50→830).
- **Bureau real vs. stub em QA:** stub parametrizável, para reprodutibilidade e para medir
  Veridia (não o bureau).
- **Testar caminhos isolados vs. concorrentes (T3):** o teste decisivo é concorrente; o isolado
  só serve de baseline, pois isolar esconde o acoplamento do pool.
- **Meta de lag "15 min" (T4):** adotado threshold mais estrito (≤5 min) com justificativa de
  partições/taxa, marcado como pendência humana por ser autoral.

## 4. Resolução e Papel do Orquestrador

Sem orquestrador/debate. As decisões foram tomadas diretamente e registradas como D1–D8 no
`design.md`, cada uma com alternativa considerada e justificativa ancorada no alvo. A ordem de
produção seguiu a dependência real imposta pelo OpenSpec (`openspec status`): proposal →
(specs, design) → tasks. Os templates reais de cada artefato foram obtidos via
`openspec instructions <artifact> --json` e preenchidos conforme o formato exigido
(`### Requirement` SHALL + `#### Scenario` WHEN/THEN nas specs).

## 5. Pendências

- **T4:** threshold de lag no pico (≤5 min) e drenagem pós-pico (≤15 min) são proposta do
  plano; precisam de confirmação dos donos do SLA.
- **Retention/offset do Kafka:** define se um backlog de 5 min é seguro quanto a expiração.
- **Escala QA→produção (4→8 pods):** premissa de escala ~linear não medida; QA é o ambiente de
  execução.
- **Observabilidade do pool Oracle:** se o tempo de aquisição de conexão não for exposto, C-1
  depende de inferência por diferença de p95.
- **Seed de ~400M linhas:** sem ele, V-1/C-1 ficam inconclusivos.
- **Soak longo (horas):** fora do escopo atual; confirmar se o negócio quer.

## 6. Proxies de Esforço (factual)

- **Subagentes invocados:** 0
- **Chamadas de ferramenta (aprox.):** ~17 — leituras de contexto (1 batch de 5 arquivos),
  CLI OpenSpec (new, status, instructions ×3 em 3 comandos), 7 escritas de artefato
  (proposal, 4 specs, design, tasks), validação, cópia+contagem, leitura das rubricas,
  e as escritas da rubrica/checklist.
- **Tamanho dos artefatos (linhas):**
  - proposal.md: 64
  - specs (4 arquivos): 223 no total (e2e 69, sync-api 52, kafka-peak 55, oracle-contention 47)
  - design.md: 231
  - tasks.md: 42
  - process.md: ~90

> Proxies de esforço relativo, não medidas exatas.

## 7. Custo e Tempo Reais (preenchido pelo usuário, da interface)

- **Nº de turnos deste braço:** 20
- **Est. Credits Used (soma dos turnos):** 6.74
- **Elapsed time (soma dos turnos):** 7m 4s
