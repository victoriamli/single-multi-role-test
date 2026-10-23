# Process — B2 (Spec do Kiro + multi-role)

> Registro de **como** o plano foi produzido (processo e raciocínio), não o plano. O plano vive
> em `design.md`. Técnica: multi-role de **contexto único** — os quatro papéis de `agents/*.md`
> encenados num só contexto, com o Orchestrator conduzindo o debate antes de consolidar.

## 1. Entendimento do Problema

Veridia é um gateway de decisão de crédito (Kotlin/Spring + Virtual Threads, K8s) com uma API
síncrona (`POST /v1/assessments`) que orquestra 3 motores externos e um caminho assíncrono via
Kafka, persistindo auditoria em Oracle. O objetivo do braço é um **plano de testes K6** (E2E,
carga, volume, stress, + spike/soak quando couber) ancorado nos fluxos, SLAs e nos cinco pontos
de tensão (T1–T5). O desafio central: o `engine-bureau` (p95 ~1.2s, instável) quase estoura
sozinho o SLA (p95<1.5s), e o pico Kafka noturno (2M/40min) disputa o pool Oracle (20/pod) com a
API. Interpretação: priorizar os testes que expõem esses acoplamentos e converter metas vagas
(T4) em thresholds numéricos.

## 2. Propostas por Papel

- **QA Planner:** mapeou os 5 fluxos críticos (aprovação, fraude, fallback do bureau,
  Kafka async, idempotência) como E2E com asserção de **veredito** (não só HTTP). Priorizou por
  risco: fallback do bureau (F3) e idempotência (F5) são regras de negócio sensíveis. Apontou T1
  e T3 como os riscos de negócio dominantes e levantou T4 como requisito não testável que
  precisa de número. Não propôs carga (fora do seu papel).
- **QA Performance Engineer:** selecionou E2E/load/spike/volume/stress; descartou **soak puro**
  (sem SLA de duração; o pico de 40 min age como soak parcial). Defendeu **modelos abertos**
  (`constant/ramping-arrival-rate`) em vez de `ramping-vus`, dimensionados por Little's Law, com
  `dropped_iterations` monitorado. Propôs thresholds ancorados no SLA (p95<1500, p99<2500,
  5xx<0.005) e a estratégia de massa única vs. fatia de reuso para T5.
- **Backend/Integration Engineer:** confirmou os gargalos: bureau (T1), pool Oracle (T3),
  Virtual Threads que empurram o limite para o pool/downstream sob bloqueio. Classificou o pico
  Kafka (T2) como **volume + spike de arrival-rate** (~833 msg/s vs 50), não soak. Exigiu stubs
  com latência/erro **injetáveis** e seed de 400M no Oracle, senão os testes de bureau/volume
  medem um mundo irreal. Para T4, propôs pensar em **tempo de dreno** e backlog, não só "lag".

## 3. Pontos de Discussão

- **Modelo de carga (aberto vs fechado):** Performance queria arrival-rate; levantou-se que VUs
  fechados são mais simples. Backend reforçou que, com bureau lento, VUs presos reduziriam a
  chegada efetiva e dariam **falso verde** no SLA.
- **Classificação do pico Kafka (T2):** soak? volume? spike? Planner via "volume"; Performance
  via "spike de chegada"; Backend via "volume + saturação de pool na janela".
- **T3 isolado vs concorrente:** Performance inicialmente modelaria API e Kafka em testes
  separados; Backend alertou que isso **esconde** a disputa do pool — o risco mais caro.
- **T4 (meta vaga):** quanto é "aceitável"? Um único número de lag vs. um conjunto (lag + backlog
  + dreno).
- **T5 (massa):** unicidade total vs. permitir reuso para testar idempotência sem poluir o p95.

## 4. Resolução e Papel do Orquestrador

O Orchestrator forçou que cada tensão fosse discutida por ≥2 papéis e consolidou:
- **Carga → modelo aberto** (D2): a objeção do Backend sobre falso verde venceu; arrival-rate com
  `preAllocatedVUs/maxVUs` por Little's Law.
- **T2 → híbrido volume+spike** (D1/SP-1): acomoda as três visões; 40 min curtos p/ soak clássico,
  mas o volume acumulado entrega o essencial.
- **T3 → CT-1 concorrente obrigatório** (D8), comparando Δp95 contra baselines isolados — rodado
  por último porque depende deles. Resolveu a divergência isolado-vs-concorrente a favor do Backend.
- **T4 → três thresholds** (D5): lag de regime <180s, backlog <750k, dreno <900s (teto de 15 min).
  Combinou a visão de lag (Planner/Performance) com a de dreno (Backend). Número marcado como
  pendência a confirmar com o negócio.
- **T5 → unicidade ≥95% + 5% reuso tagueado + seed 400M** (D6), medindo os dois caminhos sem
  contaminação.
- **Pré-condições** elevadas a requisito (RQ-ENV-2, RQ-OBS-1): stubs injetáveis e observabilidade
  do lado do alvo, senão os testes de T1/T3 perdem validade.

## 5. Pendências

- Confirmar os números de T4 (lag<180s / dreno<900s) com o negócio (P1/D9).
- Capacidade-alvo máxima para calibrar o knee point do stress (P2).
- Disponibilidade em QA de stub injetável do bureau e seed de 400M — bloqueante (P3).
- Fator de ajuste para extrapolar QA (4 pods) → prod (8 pods), não-linear (P4).
- Retenção/expurgo da auditoria, fora de escopo mas afeta fidelidade do volume (P5).

## 6. Proxies de Esforço (factual)

- **Subagentes invocados:** 0 (multi-role de contexto único — os papéis foram encenados, não
  delegados a subagentes isolados).
- **Chamadas de ferramenta (aprox.):** ~14 (9 leituras de contexto: 5 agents + 3 scenario +
  output-format; 2 de rubric; 1 listagem; 3 escritas de artefato; edições de rubric/checklist).
- **Tamanho dos artefatos:** requirements ~150 linhas, design ~150 linhas, process ~95 linhas.
- **Nº de turnos deste braço:** 4 (turno inicial de leitura+plano; 3 turnos de escrita de
  artefatos guiados pela task list).

## 7. Custo e Tempo Reais (preenchido pelo usuário, da interface)

- **Nº de turnos deste braço:** 4
- **Est. Credits Used (soma dos turnos):** 4.63
- **Elapsed time (soma dos turnos):** 4m 37s
