# Requirements — Plano de testes K6 para Veridia (B2: Spec do Kiro + multi-role)

> Modo spec do Kiro. Requisitos de teste **verificáveis** (estilo EARS) para o gateway de
> decisão de crédito **Veridia**, descrito em `scenario/target-app.md`. Cada requisito é
> ancorado nos fluxos, endpoints, SLAs e pontos de tensão (T1–T5) do alvo. O **como** (plano
> concreto) vive em `design.md`.

## Introdução

O objetivo é garantir que Veridia atenda seus SLAs sob carga esperada, de pico e extrema, e
que os fluxos de negócio críticos (aprovação, recusa por fraude, fallback do bureau, caminho
Kafka assíncrono e idempotência) funcionem de ponta a ponta. Os requisitos cobrem E2E, carga,
volume, stress e spike, com thresholds mensuráveis e estratégia de massa que não mascare a
carga real.

### Glossário de referências
- **SLA API:** `POST /v1/assessments` p95 < 1.5s, p99 < 2.5s, 5xx < 0.5%, disponibilidade 99.9%.
- **Throughput:** ~300 req/s sustentado, pico ~700 req/s (produção, 8 pods; QA 4 pods).
- **Kafka:** tópico `credit.assessment.requested` → `credit.assessment.completed`, 12
  partições; baseline ~50 msg/s; pico ~2M msgs em ~40 min (~833 msg/s).
- **Tensões:** T1 SLA apertado vs bureau lento/instável; T2 pico Kafka noturno; T3 pool Oracle
  (20/pod) compartilhado API↔consumer; T4 meta vaga de lag no pico; T5 massa vs idempotência.

---

## Requisitos

### RQ-E2E — Fluxos de negócio críticos

**RQ-E2E-1 — Aprovação direta (Fluxo 1)**
- **ENQUANTO** os 3 motores (`engine-score`, `engine-fraud`, `engine-bureau`) retornam OK,
  **QUANDO** um `POST /v1/assessments` de cliente elegível é enviado, **O sistema DEVE**
  responder veredito `APPROVED` dentro do SLA (p95 < 1.5s).
- **Verificável por:** asserção de corpo (`verdict == APPROVED`), não só HTTP 200.

**RQ-E2E-2 — Recusa por fraude (Fluxo 2)**
- **QUANDO** `engine-fraud` sinaliza risco, **O sistema DEVE** responder `DECLINED` mesmo com
  score bom.
- **Verificável por:** asserção `verdict == DECLINED` com `engine-score` OK no stub.

**RQ-E2E-3 — Degradação do bureau / fallback (Fluxo 3, T1)**
- **QUANDO** `engine-bureau` dá timeout (2s, sem retry), **O sistema DEVE** responder
  `APPROVED_WITH_REVIEW` e **DEVE** permanecer dentro do SLA p95 < 1.5s / p99 < 2.5s.
- **Verificável por:** stub do bureau com timeout injetado + asserção de veredito e de latência.

**RQ-E2E-4 — Caminho assíncrono Kafka (Fluxo 4)**
- **QUANDO** um pedido entra em `credit.assessment.requested`, **O sistema DEVE** publicar o
  resultado em `credit.assessment.completed` com o **mesmo `traceId`**.
- **Verificável por:** produção/consumo via xk6-kafka correlacionando `traceId`.

**RQ-E2E-5 — Idempotência (Fluxo 5, T5)**
- **QUANDO** o mesmo `traceId` é reprocessado, **O sistema DEVE NÃO** gerar registro de
  auditoria duplicado.
- **Verificável por:** contagem de linhas de auditoria por `traceId` == 1 após reenvio.

### RQ-LOAD — Carga esperada

**RQ-LOAD-1 — API síncrona em carga de produção**
- **ENQUANTO** sob ~300 req/s sustentado, **O sistema DEVE** manter p95 < 1.5s, p99 < 2.5s e
  5xx < 0.5%.
- **Verificável por:** cenário `constant-arrival-rate` a 300 req/s com os thresholds acima.

**RQ-LOAD-2 — Consumer Kafka em volume de baseline**
- **ENQUANTO** sob ~50 msg/s, **O sistema DEVE** manter lag de consumo < 10s.
- **Verificável por:** produção a 50 msg/s + medição de lag (métrica do lado do alvo).

### RQ-SPIKE / RQ-VOLUME — Pico Kafka noturno (T2, T4)

**RQ-PEAK-1 — Absorção do pico noturno**
- **QUANDO** ~2M mensagens chegam em ~40 min (~833 msg/s), **O sistema DEVE** drenar o backlog
  completo em ≤ 15 min após o fim do despejo (teto contratual) e manter lag de regime < 180s.
- **Verificável por:** `ramping-arrival-rate` / produção acelerada + métricas de lag e backlog.

**RQ-PEAK-2 — Threshold numérico derivado do T4**
- **O plano DEVE** substituir a meta vaga "aceitável até 15 min" por thresholds testáveis:
  lag de regime < 180s, backlog de pico < 750k msgs, tempo total de dreno < 900s.
- **Verificável por:** thresholds explícitos no cenário de pico (ver `design.md` D4).

### RQ-STRESS — Além da capacidade (T1)

**RQ-STRESS-1 — Ponto de ruptura da API**
- **QUANDO** a taxa de chegada sobe além de ~700 req/s, **O sistema DEVE** ter seu *knee point*
  identificado (onde p95 estoura o SLA ou 5xx > 0.5%).
- **Verificável por:** `ramping-arrival-rate` crescente até degradação, registrando a taxa de
  ruptura.

**RQ-STRESS-2 — Stress com bureau instável (T1)**
- **ENQUANTO** o stub do bureau injeta picos de timeout/5xx, **O plano DEVE** medir o efeito no
  SLA e na taxa de fallback.
- **Verificável por:** cenário de stress combinado com perfil de erro do bureau.

### RQ-CONTENTION — Pool Oracle compartilhado (T3)

**RQ-CONT-1 — API + consumer concorrentes**
- **QUANDO** a API síncrona e o consumer Kafka de pico rodam **simultaneamente**, **O sistema
  DEVE** ter o impacto no pool (tamanho 20/pod) medido como Δp95 vs. baseline isolado.
- **Verificável por:** execução concorrente comparada a execução isolada (Δp95, erros, lag).

### RQ-DATA — Estratégia de massa (T5)

**RQ-DATA-1 — Unicidade por padrão**
- **O plano DEVE** gerar `traceId` e `customerId` únicos por iteração (≥ 95%) para não ativar
  o caminho de idempotência e não mascarar a carga de escrita.

**RQ-DATA-2 — Fatia de reuso controlada**
- **O plano DEVE** reservar uma fração marcada (~5%) de `traceId` reutilizados para exercitar
  explicitamente a idempotência (RQ-E2E-5), sem contaminar as métricas de carga.

**RQ-DATA-3 — Seed de volume para Oracle**
- **O plano DEVE** assumir a tabela de auditoria pré-carregada (~400M linhas) para que as
  consultas de idempotência por índice reflitam a realidade, excluindo os `traceId` de teste.

### RQ-OBS — Observabilidade (não só K6)

**RQ-OBS-1 — Métricas do lado do alvo**
- **O plano DEVE** exigir coleta, além do K6: lag de consumer, uso do pool Oracle, latência por
  downstream e taxa de fallback do bureau.
- **Verificável por:** lista de métricas-alvo documentada como pré-condição de execução.

### RQ-ENV — Ambiente e fidelidade

**RQ-ENV-1 — Extrapolação QA→produção**
- **O plano DEVE** registrar que QA tem 4 pods e produção 8, e que a extrapolação de capacidade
  **não é linear** (pool, bureau e broker escalam de forma diferente).

**RQ-ENV-2 — Fidelidade dos stubs**
- **O plano DEVE** exigir stubs dos motores com **latência e erro injetáveis** (não fixos) para
  reproduzir T1; stubs fixos invalidam os testes de bureau.

---

## Critérios de aceite globais (resumo verificável)

| Requisito | Métrica-chave | Threshold |
|-----------|---------------|-----------|
| RQ-LOAD-1 | `http_req_duration` p95 / p99 | p95 < 1500ms, p99 < 2500ms |
| RQ-LOAD-1 | `http_req_failed` | rate < 0.005 |
| RQ-LOAD-2 | lag consumer (baseline) | < 10s |
| RQ-PEAK-1/2 | lag de regime / backlog / dreno | < 180s / < 750k / < 900s |
| RQ-E2E-3 | fallback sob SLA | `verdict==APPROVED_WITH_REVIEW` e p95 < 1500ms |
| RQ-CONT-1 | Δp95 concorrente vs isolado | reportado + sem violar SLA |
| RQ-E2E-5 | auditoria por `traceId` | == 1 |

## Pendências (decisão humana)
- Confirmar o threshold de T4 proposto (180s/900s) com o negócio.
- Confirmar capacidade-alvo máxima para calibrar o stress (knee point esperado).
- Confirmar disponibilidade em QA de: stub injetável do bureau e seed de 400M.
