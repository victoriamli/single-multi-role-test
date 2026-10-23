## Context

**Veridia** é um gateway de decisão de crédito (Kotlin + Spring Boot, Virtual Threads,
Kubernetes: 4 pods em QA / 8 em prod) com três superfícies sob teste:

1. **API síncrona `POST /v1/assessments`** — chama 3 motores HTTP em paralelo
   (`engine-score` p95 ~80ms, `engine-fraud` p95 ~250ms, `engine-bureau` p95 ~1.2s e
   **instável**) e só responde quando os 3 voltam ou estoura o timeout de 2s por downstream
   (sem retry). SLA: **p95 < 1.5s, p99 < 2.5s**, 5xx < 0.5%, throughput sustentado ~300 req/s
   (pico ~700 req/s).
2. **Consumer Kafka** `credit.assessment.requested` → `credit.assessment.completed`, 12
   partições, idempotência por `traceId` (SELECT Oracle). **Pico noturno** de ~2M msgs em
   ~40 min (~833 msg/s) vs ~50 msg/s normal. Lag fora do pico < 10s; no pico "aceitável até
   15 min" (meta vaga).
3. **Oracle** — auditoria por avaliação; tabela ~400M linhas sem expurgo; **pool de 20
   conexões/pod compartilhado** entre API e consumer.

Este design é o **plano de testes K6** do alvo. Ele consolida a análise de três especialistas
executados como **subagentes isolados** (QA Planner, Backend/Integration Engineer, QA
Performance Engineer) e as decisões do Orchestrator sobre as divergências. O processo está em
`runs/A3-openspec-multiagent/process.md`.

Pontos de tensão a endereçar: **T1** SLA apertado vs bureau; **T2** pico Kafka; **T3** pool
Oracle compartilhado; **T4** meta de lag vaga; **T5** massa mal gerada mascara escrita.

## Goals / Non-Goals

**Goals:**
- Cobrir os **5 fluxos críticos** E2E com asserção de **veredito**, não só de latência.
- Escolher e justificar os **tipos de teste** (E2E/load/spike/stress/volume/soak) para o alvo.
- Definir **modelagem de carga** (executor K6, taxa/VUs, estágios) alinhada a ~300/~700 req/s
  e ~833 msg/s, com **thresholds numéricos** ancorados nos SLAs.
- Transformar a meta vaga de lag (T4) em **thresholds testáveis**.
- Expor a **contenção do pool Oracle** (T3) com um cenário de sobreposição e métrica
  diferencial.
- Definir **estratégia de massa** que não mascare a carga de escrita (T5).
- Nomear **riscos de medição** e **pendências** de decisão humana.

**Non-Goals:**
- Implementar scripts K6 completos (apenas trechos ilustrativos de `options`/`thresholds`).
- Redesenhar o alvo (ex.: adicionar cache/retry ao bureau, expurgo da tabela, novo pool).
- Afinar tuning fino de infraestrutura (sizing de pods, partições) além do necessário para a
  validade do teste.

## Decisions

### D1 — Tipos de teste: incluir os 6, com foco e exclusões justificadas

| Tipo | Aplica? | Justificativa ancorada no alvo |
|------|---------|--------------------------------|
| **E2E** | Sim | Os 5 fluxos definem corretude (veredito), não só performance — F1 fraude e F2 fallback têm risco de **decisão errada**. |
| **Load** | Sim | Validar o SLA no throughput sustentado de ~300 req/s (T1). |
| **Spike** | Sim | Pico declarado de ~700 req/s; subida abrupta expõe o custo do bureau no p99. |
| **Stress** | Sim | Achar o **knee point** acima de 700 req/s e qual recurso satura primeiro (pool/VT/downstream). |
| **Volume** | Sim | Pico Kafka de ~2M mensagens e tabela de 400M linhas encarecendo o SELECT de idempotência. |
| **Soak** | Sim (fundido) | Os 40 min sustentados do pico Kafka **são** o soak relevante; não há soak isolado de dias no escopo. |

**Exclusões/fusões:** stress isolado **só da API** foi despriorizado em favor do stress com
**sobreposição Kafka** (D5), porque o isolado dá falso verde sobre o pool (T3). Volume Kafka
"puro" foi **fundido** com soak — 2M msgs + 40 min são o mesmo evento.

### D2 — Modelo aberto (arrival-rate) para a API, não closed-model por VUs

O Orchestrator adotou a posição do QA Performance Engineer (divergência D2 do processo):
usar **executores de arrival-rate** na API. Num closed-model por VUs, se o bureau fica lento,
os VUs ficam presos esperando a resposta e a **taxa de chegada cai sozinha**, mascarando a
violação de SLA sob a carga alvo. O modelo aberto mantém a pressão de ~300/~700 req/s
independentemente da latência de resposta — que é exatamente o que o SLA promete ao cliente.

Dimensionamento de VUs de apoio por **Little's Law**: com ~300 req/s e tempo de resposta
dominado pelo bureau (~1.2s), in-flight ≈ 300 × 1.3s ≈ **~390 requisições simultâneas** →
`preAllocatedVUs`/`maxVUs` devem cobrir isso com folga para absorver picos de latência.

```js
// Ilustrativo — Load API
scenarios: {
  load_api: {
    executor: 'ramping-arrival-rate',
    startRate: 50, timeUnit: '1s',
    preAllocatedVUs: 500, maxVUs: 800,
    stages: [
      { target: 300, duration: '2m' },   // ramp/warm-up
      { target: 300, duration: '15m' },  // sustentado
    ],
  },
},
thresholds: {
  http_req_duration: ['p(95)<1500', 'p(99)<2500'],
  http_req_failed: ['rate<0.005'],
}
```

### D3 — Thresholds por teste (ancorados no SLA)

| Teste | Executor | Perfil | Thresholds-chave |
|-------|----------|--------|------------------|
| **A. Load API** | `ramping-arrival-rate` | 50→300 req/s (2m), 300 sustentado (15m) | `p(95)<1500`, `p(99)<2500`, `rate<0.005` |
| **B. Spike API** | `ramping-arrival-rate` | 300→700 em 30s, 700 por 5m | SLA medido no pico; degradação reportada como decisão (D5-pendência) |
| **C. Stress API+Kafka** | `ramping-arrival-rate` + `xk6-kafka` | API além de 700 req/s + Kafka concorrente | threshold **afrouxado** para achar knee; registrar recurso que satura |
| **D. Pico Kafka** | `constant-arrival-rate` (xk6-kafka) | ~833 msg/s por 40m (warm-up incluso) | `lag<180s`, `backlog<750k`, `drain<900s` |
| **E. E2E F1–F5** | `per-vu-iterations` | 1 iteração por cenário, baixa concorrência | checks de veredito (ver D7); `completed/requested rate==1` (F5) |
| **F. Sobreposição (T3)** | A + D simultâneos | 300 req/s API + 833 msg/s Kafka | `Δp95_api ≤ 150ms` vs solo; `oracle_pool_wait_ms` observado |

### D4 — T4: "15 min" vira três thresholds numéricos

A meta vaga foi decomposta (convergência entre Backend ~750k e QA Perf):

- **Lag operacional** durante o pico: **< 180s**.
- **Backlog máximo**: **< 750.000 mensagens**.
- **Tempo de drenagem** após o pico: **< 900s** (os "15 min", como *hard cap*, não meta
  normal).

Conta: para o backlog não crescer indefinidamente, a capacidade de consumo `C_max` precisa
ser > 833 msg/s. O backlog acumulado em regime de déficit é
`B ≈ drain_time × (taxa_entrada − C_max)`. Com `drain_time ≤ 900s` e um `C_max` alvo de ~900
msg/s sobre baseline de 50, `B ≈ 900 × (900 − 50) / ...` fica na ordem de **~765k** mensagens
— convergente com a estimativa independente de ~750k do Backend. **`C_max` é bloqueante**:
sem medi-lo, as metas de backlog/drenagem são provisórias (pendência Q2).

### D5 — T3: sobreposição obrigatória + medição diferencial

Os três papéis convergiram que **testar caminhos isolados dá falso verde**. Decisão: o
critério de aceite do pico depende do teste **F (sobreposição)**, não dos testes solo. Métrica
central: **`Δp95_api` = p95(API em sobreposição) − p95(API solo) ≤ 150ms** (≤ ~10% do
orçamento de SLA). Instrumentar no lado do alvo: `oracle_pool_wait_ms`, `active_connections`,
lag por partição. Alerta de extrapolação: QA tem 4 pods (80 conexões) vs prod 8 pods (160); a
contenção é **não-linear**, então resultados de QA não escalam diretamente (pendência Q3).

### D6 — T1: bureau com instabilidade **injetada**, não mock fixo

O mock do `engine-bureau` SHALL injetar latência (p95 ~1.2s) e erros (timeout/5xx em fração
configurável). Mock fixo esconderia justamente o risco que T1 descreve. `engine-score` e
`engine-fraud` podem ser mocks fixos (rápidos e estáveis). Medir **taxa de fallback**
(`APPROVED_WITH_REVIEW`) como métrica de primeira classe e garantir que timeout de downstream
**não** vira 5xx da API (`http_req_failed < 0.005`).

### D7 — E2E com asserção de veredito, priorizando decisão sobre latência

Seguindo o QA Planner, F1 (fraude → `DECLINED`) e F2 (fallback → `APPROVED_WITH_REVIEW`) têm
prioridade sobre F3 (aprovação direta) por risco de **decisão incorreta**. Checks K6 por
fluxo:

| Fluxo | Entrada | Veredito esperado | Check |
|-------|---------|-------------------|-------|
| F3 aprovação | 3 motores OK | `APPROVED` | status 200 + `verdict==APPROVED` + p95<1.5s |
| F1 fraude | fraud=risco, score=bom | `DECLINED` | `verdict==DECLINED` |
| F2 fallback | bureau timeout/5xx | `APPROVED_WITH_REVIEW` | `verdict==APPROVED_WITH_REVIEW` + não-5xx |
| F5 Kafka | produz em `requested` | msg em `completed` mesmo `traceId` | `rate(completed/requested)==1` |
| F4 idempotência | reenvia `traceId` | sem auditoria duplicada | contagem auditoria == 1 (inclui caso concorrente) |

### D8 — T5: unicidade por padrão + 5% de reuso proposital

`traceId` via `uuidv4`; `customerId` via `cust-<VU>-<ITER>-<timestamp>`. **≥ 95% únicos** para
não ativar idempotência e medir a carga real de escrita; **~5% de reuso deliberado** para
exercitar o caminho de idempotência, medido **à parte**. O seed de 400M linhas deve **excluir**
os `traceId` de teste, para o SELECT ser representativo sem gerar falso positivo de dedupe.
O Orchestrator manteve os 5% (divergência D1 do processo) por dar cobertura de idempotência
sob carga sem contaminar a métrica de escrita, desde que segmentada por tag.

### D9 — Matriz de rastreabilidade (teste → fluxo/tensão)

| Teste | Cobre | Capability (spec) |
|-------|-------|-------------------|
| A Load API | T1, SLA sustentado | `sync-api-load-stress` |
| B Spike API | pico ~700, T1 | `sync-api-load-stress` |
| C Stress | knee point, T3 | `sync-api-load-stress`, `oracle-pool-contention` |
| D Pico Kafka | T2, T4 | `kafka-peak-throughput` |
| E E2E | F1–F5 | `e2e-critical-flows` |
| F Sobreposição | T3 | `oracle-pool-contention` |
| (transversal) Bureau | T1 | `bureau-degradation-resilience` |
| (transversal) Massa | T5 | `mass-data-strategy` |

## Risks / Trade-offs

- **[Falso verde por medição isolada]** → Teste F (sobreposição) é pré-condição de aceite do
  pico; testes solo não bastam (D5).
- **[Extrapolação QA→prod não-linear]** (80 vs 160 conexões) → Reportar resultados de QA como
  piso, não como predição de prod; marcar como pendência (Q3).
- **[`C_max` desconhecido]** → Metas de backlog/drenagem (D4) são provisórias até medir a
  capacidade real do consumer; bloqueante (Q2).
- **[SLA no pico de 700 req/s não declarado]** → O alvo só declara SLA para ~300 req/s; se o
  p95 degradar no spike, é decisão de negócio aceitar ou não (Q1).
- **[Dependência de mock injetável do bureau]** → Sem injeção de latência/erro, T1 não é
  exercitado; mock fixo invalida o teste (D6).
- **[Seed de 400M em QA]** → Custo/tempo de preparar a tabela; sem ele, o SELECT de
  idempotência fica irrealisticamente barato (D8).
- **[Virtual Threads mascarando saturação]** → VT permitem muitos in-flight; o limite real
  pode ser o pool Oracle ou o bureau, não CPU — instrumentar in-flight/VT ativas para não
  concluir "tem folga" cedo demais.

## Migration Plan

Não se aplica — artefato de **planejamento de testes**, sem deploy de produção. A "execução"
é a futura implementação dos scripts K6 (fora do escopo deste change), na ordem das tasks.

## Open Questions

- **Q1 (T4/SLA pico):** Qual o SLA aceitável da API durante o spike de 700 req/s? O alvo só
  declara SLA para ~300 req/s. (Decisão de negócio.)
- **Q2 (C_max):** Qual a capacidade máxima de consumo do consumer (msg/s)? Bloqueia as metas
  de backlog/drenagem de D4.
- **Q3 (QA→prod):** Como extrapolar a contenção do pool de 4 pods (QA) para 8 pods (prod) dado
  o comportamento não-linear?
- **Q4 (veredito multi-downstream):** Qual o veredito quando mais de um motor degrada
  simultaneamente (ex.: fraude indisponível + bureau timeout)? Regra de negócio não descrita.
- **Q5 (idempotência):** Há constraint única em `traceId` / a dedupe é atômica sob
  concorrência? Afeta o aceite de F4 concorrente.
- **Q6 (fração de reuso):** Os 5% de reuso de massa (D8) são o valor certo, ou deve ser
  ajustado conforme o perfil real de reenvio em produção?
