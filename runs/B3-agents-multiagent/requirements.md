# Requisitos — Plano de testes K6 para Veridia (B3: spec do Kiro, multi-agente)

> Modo spec do Kiro. Este documento define **requisitos verificáveis** (estilo EARS) para o
> plano de testes de performance/E2E do gateway de decisão de crédito **Veridia**. O plano em
> si (modelagem de carga, thresholds, cenários) vive no `design.md` deste run. Tudo ancorado
> no alvo descrito em `scenario/target-app.md` (fluxos F1–F5, SLAs, tensões T1–T5).
>
> Produzido por Orchestrator consolidando três subagentes isolados (QA Planner,
> Backend/Integration Engineer, QA Performance Engineer). Ver `process.md`.

## Glossário / referências do alvo

- **F1** aprovação direta → `APPROVED`; **F2** fraude → `DECLINED`; **F3** timeout do bureau
  → `APPROVED_WITH_REVIEW` (fallback); **F4** caminho assíncrono Kafka (mesmo `traceId`);
  **F5** idempotência (sem auditoria duplicada).
- **SLA API:** p95 < 1.5s, p99 < 2.5s, 5xx < 0.5%, disponibilidade 99.9%.
- **Throughput:** ~300 req/s sustentado, pico ~700 req/s.
- **Kafka:** 12 partições; normal ~50 msg/s; pico ~2M msgs em ~40 min (~833 msg/s); lag
  normal < 10s; lag de pico "até 15 min" (vago — T4).
- **Oracle:** pool 20 conexões/pod **compartilhado** entre API síncrona e consumer; tabela
  de auditoria ~400M linhas sem expurgo.
- **Tensões:** T1 SLA apertado vs bureau lento/instável; T2 pico Kafka noturno; T3 pool Oracle
  compartilhado; T4 meta de lag vaga; T5 massa vs idempotência.

---

## 1. Requisitos de cobertura funcional (E2E)

**RQ-E2E-1 — Aprovação direta (F1).**
QUANDO os três motores (`engine-score`, `engine-fraud`, `engine-bureau`) retornam OK para um
pedido elegível, O plano DEVE verificar que `POST /v1/assessments` responde `APPROVED` na mesma
requisição E que um registro de auditoria é gravado com as latências por motor.
*Verificável:* asserção de corpo (`verdict == APPROVED`) + presença de 1 registro de auditoria.

**RQ-E2E-2 — Recusa por fraude (F2).**
QUANDO `engine-fraud` sinaliza risco com `engine-score` bom, O plano DEVE verificar que o
veredito é `DECLINED` (fraude domina o score) e que a auditoria registra os sinais.
*Verificável:* `verdict == DECLINED` com score positivo injetado no mock.

**RQ-E2E-3 — Fallback do bureau (F3).**
QUANDO `engine-bureau` dá timeout (>2s) OU retorna 5xx, O plano DEVE verificar que o veredito
é `APPROVED_WITH_REVIEW`, que **nenhum 5xx** vaza para o chamador por falha exclusiva do
bureau, e que a auditoria distingue o veredito de fallback de uma aprovação normal.
*Verificável:* `verdict == APPROVED_WITH_REVIEW` + `http_req_failed` do chamador não aumenta.

**RQ-E2E-4 — Caminho assíncrono Kafka (F4).**
QUANDO um pedido entra em `credit.assessment.requested`, O plano DEVE verificar que um
resultado correlato aparece em `credit.assessment.completed` com o **mesmo `traceId`**
(correlação 1:1, sem perda), via xk6-kafka.
*Verificável:* contagem de mensagens produzidas == consumidas; `traceId` de entrada == saída.

**RQ-E2E-5 — Paridade de veredito sync/async (F1–F4).**
O plano DEVE verificar que, para a mesma entrada, o veredito pelo caminho síncrono e pelo
assíncrono é consistente (sujeito a confirmação humana — ver RQ-OPEN-5).
*Verificável:* amostra de entradas idênticas pelos dois caminhos → vereditos iguais.

**RQ-E2E-6 — Idempotência (F5).**
QUANDO o mesmo `traceId` é reprocessado — sequencialmente E **concorrentemente** —, O plano
DEVE verificar que apenas **um** registro de auditoria existe e que nenhuma conclusão
conflitante é republicada.
*Verificável:* 1 linha de auditoria por `traceId` após N replays concorrentes.

---

## 2. Requisitos de performance (ancorados no SLA)

**RQ-PERF-1 — Latência sob carga sustentada.**
ENQUANTO a API recebe ~300 req/s sustentado (modelo aberto), O plano DEVE verificar
`http_req_duration p(95) < 1500ms` e `p(99) < 2500ms`.

**RQ-PERF-2 — Taxa de erro.**
ENQUANTO qualquer teste de carga/stress/spike roda, O plano DEVE verificar
`http_req_failed rate < 0.005` (5xx < 0.5%), medido no servidor e no cliente.

**RQ-PERF-3 — Pico de chegada (spike) síncrono.**
QUANDO a taxa de chegada salta em direção/acima de ~700 req/s, O plano DEVE medir a
degradação de p95/p99 e a recuperação, e DEVE verificar ausência de `dropped_iterations`
não explicados (guarda contra *coordinated omission*).

**RQ-PERF-4 — Ponto de ruptura (stress).**
O plano DEVE rampar a taxa de chegada **acima de 700 req/s** até identificar o *knee point*
(onde p95/p99 ou 5xx rompem o SLA) e DEVE nomear o recurso que satura primeiro (pool Oracle,
bureau, broker ou CPU).

**RQ-PERF-5 — Modelo de carga aberto obrigatório.**
Todos os testes de throughput/SLA DEVEM usar executores de **taxa de chegada**
(`constant-arrival-rate` / `ramping-arrival-rate`), não VUs fechados, para não mascarar os
gargalos (ver design §Modelo de carga). Exceção única: o teste de corrida de idempotência
(RQ-E2E-6), onde a simultaneidade é a grandeza sob teste.

**RQ-PERF-6 — Degradação do bureau dentro do SLA (T1).**
QUANDO o mock do bureau opera nos perfis `bureau-slow` (p95→1.8s) e `bureau-flaky` (10–30%
5xx), O plano DEVE verificar que o SLA do endpoint é tratado — seja via fallback dentro do
limite acordado (RQ-OPEN-2), seja nomeando a quebra como achado — e DEVE medir a **taxa de
fallback** (`APPROVED_WITH_REVIEW`).

---

## 3. Requisitos de carga assíncrona (Kafka) e lag

**RQ-KAFKA-1 — Baseline de consumo.**
ENQUANTO o consumer recebe ~50 msg/s, O plano DEVE verificar lag de consumo `< 10s`.

**RQ-KAFKA-2 — Pico noturno (T2).**
QUANDO a taxa de chegada rampa de 50 para ~833 msg/s e sustenta por ~40 min (modelo aberto,
12 partições), O plano DEVE medir o lag máximo durante o pico e o tempo de dreno após o pico.

**RQ-KAFKA-3 — Threshold de lag no pico (T4, numérico).**
O plano DEVE adotar um threshold **numérico** de lag no pico no lugar da meta vaga "até 15
min": linha vermelha `time-lag ≤ 900000ms` (15 min), dreno pós-pico `≤ 600000ms` (10 min),
perda zero + correlação 1:1 de `traceId`; alvo interno amber/green `≤ 300000ms` (5 min).
Sujeito a ratificação humana (RQ-OPEN-1).

---

## 4. Requisitos de acoplamento e contenção (T3)

**RQ-T3-1 — Cenário combinado obrigatório.**
O plano DEVE incluir um teste que roda a carga síncrona (~300 req/s) E o pico Kafka (~833
msg/s) **simultaneamente**, pois os dois caminhos disputam o mesmo pool Oracle (20/pod). Este
cenário é **bloqueante de release**.

**RQ-T3-2 — Medir o acoplamento, não só cada caminho.**
O plano DEVE comparar p95/p99 da API no cenário **isolado** vs. **combinado** (Δp95) e DEVE
observar, do lado do alvo, o tempo de espera por conexão e a exaustão do pool.

---

## 5. Requisitos de dados de massa (T5)

**RQ-DATA-1 — Unicidade por padrão.**
Para testes de carga/stress/spike/volume, O plano DEVE gerar `traceId` e `customerId`
**únicos** (ex.: UUID ou `${VU}-${__ITER}-${runSalt}`) para exercer o caminho real de escrita
e não o atalho de idempotência.

**RQ-DATA-2 — Fração de reuso controlada e isolada.**
O plano DEVE incluir uma fração pequena (~1–2%) de `traceId` **reutilizados e tagueados** para
exercer a idempotência (RQ-E2E-6), **excluída** dos thresholds de latência de SLA.

**RQ-DATA-3 — Seed representativo da auditoria.**
O plano DEVE exigir, como pré-condição de testes de volume/contenção, uma tabela de auditoria
semeada em escala representativa (~400M linhas) com profundidade de índice fiel; caso contrário,
o custo de escrita e a contenção do pool ficam subestimados (anotar como viés).

---

## 6. Requisitos de medição e fidelidade de ambiente

**RQ-MEAS-1 — Observabilidade no lado do alvo.**
O plano DEVE coletar, além das métricas de cliente do K6: uso/espera/exaustão do pool Oracle
(por pod); `records-lag-max` e time-lag por partição + taxa de dreno + rebalances; latência e
erros por downstream (score/fraude/bureau, p50/p95/p99 + timeouts + 5xx); taxa de fallback do
bureau; latência/5xx do lado servidor; latência de escrita de auditoria; heap/GC e eventos de
**pinning de Virtual Thread**; nº de pods/HPA.

**RQ-MEAS-2 — Mocks dos motores com injeção.**
O plano DEVE usar mocks de **distribuição de latência** para score/fraude e um mock
**programável** para o bureau (presets `bureau-healthy`, `bureau-slow`, `bureau-flaky`,
`bureau-brownout`, com capacidade de deslocar a distribuição no meio do teste). Mock de latência
fixa é aceitável apenas para E2E funcional puro.

**RQ-MEAS-3 — Não extrapolar QA→prod linearmente.**
O plano DEVE reportar métricas **por pod** e NÃO DEVE projetar capacidade de QA (4 pods) para
produção (8 pods) de forma linear, pois os limitadores (pool e downstream) não escalam com o
nº de pods.

**RQ-MEAS-4 — Paridade de partições Kafka.**
O plano DEVE exigir 12 partições no ambiente de teste; caso contrário o teste de pico não é
comparável a produção (anotar como pré-condição).

---

## 7. Pendências que exigem decisão humana (Open Questions)

- **RQ-OPEN-1 (T4):** ratificar o threshold numérico de lag no pico proposto em RQ-KAFKA-3
  (15 min linha vermelha / 5 min alvo) e definir a referência de medição (idade de mensagem vs
  fim da janela de batch).
- **RQ-OPEN-2 (T1/F3):** o veredito de fallback `APPROVED_WITH_REVIEW` deve cumprir o **mesmo**
  p95/p99 de um veredito normal, ou um limite diferente quando o bureau está fora?
- **RQ-OPEN-3:** teto aceitável de **taxa de fallback** (% de `APPROVED_WITH_REVIEW` por
  instabilidade do bureau) antes de considerar degradação de qualidade de decisão.
- **RQ-OPEN-4:** janela/escopo de medição de disponibilidade 99.9% e 5xx < 0.5% (por endpoint,
  por pod, janela móvel).
- **RQ-OPEN-5:** paridade sync/async (RQ-E2E-5) é obrigatória para a mesma entrada, ou
  divergências são aceitas?
- **RQ-OPEN-6:** escopo da idempotência — apenas `traceId`? Supressão cross-path (sync vs
  Kafka) é esperada ou só dentro do caminho Kafka?
- **RQ-OPEN-7:** capacidade-alvo máxima (C_max) para dimensionar o teste de stress e as metas
  de lag; é bloqueante para RQ-PERF-4 e RQ-KAFKA-3.
- **RQ-OPEN-8:** disponibilidade de seed de 400M e de stub injetável do bureau em QA
  (RQ-DATA-3, RQ-MEAS-2) — pré-condições bloqueantes.
