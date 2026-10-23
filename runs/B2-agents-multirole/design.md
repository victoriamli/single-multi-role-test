# Design — Plano de testes K6 para Veridia (B2: Spec do Kiro + multi-role)

> Este é **o plano de testes** (artefato lido pela avaliação). Produzido no modo spec do Kiro
> a partir de `requirements.md`, consolidando o debate dos quatro papéis conduzido pelo
> Orchestrator. Ancorado em `scenario/target-app.md`. Foco em planejamento, não implementação:
> trechos curtos de `options`/`thresholds` são ilustrativos.

## D0. Contexto e premissas de medição

- **Alvo:** Veridia, Kotlin/Spring Boot + Virtual Threads, K8s. QA = 4 pods, Prod = 8 pods.
- **Capacidade de referência:** ~300 req/s sustentado, ~700 req/s pico (prod). Em QA (metade
  dos pods) as metas são verificadas em escala proporcional e **não extrapoladas linearmente**
  (pool Oracle, bureau e broker escalam diferente — RQ-ENV-1).
- **Stubs dos 3 motores** com latência/erro **injetáveis** (RQ-ENV-2): `engine-score` ~80ms,
  `engine-fraud` ~250ms, `engine-bureau` ~1.2s com perfil de timeout/5xx ligável. Stub fixo
  invalida os testes de T1.
- **Oracle** pré-carregado ~400M linhas de auditoria, excluindo os `traceId` de teste
  (RQ-DATA-3), para a idempotência consultar índice sob tamanho realista.
- **Observabilidade do lado do alvo** (RQ-OBS-1): lag de consumer, uso do pool (20/pod),
  latência por downstream, taxa de fallback do bureau — coletadas em paralelo ao K6.

## D1. Tipos de teste — escolhidos e descartados

| Tipo | Decisão | Justificativa (ancorada no alvo) |
|------|---------|----------------------------------|
| **E2E** | **Incluir** | Validar os 5 fluxos com asserção de **veredito** (não só HTTP). Fallback do bureau (F3) e idempotência (F5) são regras de negócio, não só latência. |
| **Load** | **Incluir** | Verificar SLA a ~300 req/s (API) e lag < 10s a 50 msg/s (Kafka) — a carga declarada de produção. |
| **Spike** | **Incluir** | O pico Kafka (833 msg/s vs 50 msg/s) é, no arrival-rate, um spike de chegada. Também há spike de erro do bureau (T1). |
| **Volume** | **Incluir** | 2M msgs acumuladas + 400M linhas no Oracle: o custo de escrita/índice sob backlog só aparece em volume. |
| **Stress** | **Incluir** | Achar o *knee point* da API acima de 700 req/s e o efeito do bureau instável (T1). |
| **Soak (puro)** | **Descartar como teste dedicado** | Não há SLA de duração/vazamento declarado. O comportamento prolongado relevante (pico de 40 min + pool saturado) já é coberto pelo cenário de pico Kafka, que atua como soak parcial. Reavaliar se surgir requisito de estabilidade longa. |

> **Nota do debate:** o pico Kafka foi classificado como **híbrido volume + spike-arrival**, não
> soak. Os 40 min são curtos demais para soak clássico, mas o volume acumulado e a saturação do
> pool durante a janela entregam o essencial que um soak buscaria aqui.

## D2. Modelagem de carga — por teste

Decisão transversal (QA Performance + Backend, consolidada pelo Orchestrator): usar **modelos
abertos** (`constant-arrival-rate` / `ramping-arrival-rate`) para a API, **não** `ramping-vus`.
Com VUs fechados, um `engine-bureau` lento faz cada VU esperar e **reduz a taxa de chegada
efetiva**, mascarando a violação do SLA. O modelo aberto mantém a chegada fixa e expõe a fila
real. Dimensionar `preAllocatedVUs`/`maxVUs` por Little's Law e monitorar `dropped_iterations`.

### Carga (LT)
- **LT-1 — API @ produção:** `constant-arrival-rate`, 300 req/s, 10 min + 2 min warm-up.
  - `preAllocatedVUs` ≈ 300 × 1.5s ≈ 450; `maxVUs` ≈ 700 (folga p/ cauda do bureau).
  - Massa: `traceId`/`customerId` únicos (RQ-DATA-1).
- **LT-2 — Consumer @ baseline:** produção xk6-kafka a 50 msg/s por 10 min; medir lag.

```js
// LT-1 (ilustrativo)
scenarios: {
  api_load: { executor: 'constant-arrival-rate', rate: 300, timeUnit: '1s',
              duration: '10m', preAllocatedVUs: 450, maxVUs: 700 }
}
thresholds: { http_req_duration: ['p(95)<1500','p(99)<2500'], http_req_failed: ['rate<0.005'] }
```

### Spike (SP)
- **SP-1 — Pico Kafka noturno:** `ramping-arrival-rate` de 50 → 833 msg/s em alguns minutos,
  sustentar o pico, voltar a 50. Representa 2M/40 min de forma acelerada/escalada. Medir lag,
  backlog e tempo de dreno.
- **SP-2 — Spike de erro do bureau (T1):** sob LT-1 estável, injetar janela de timeout/5xx no
  `engine-bureau`; medir SLA e taxa de fallback (F3).

### Volume (VOL)
- **VOL-1 — Oracle sob massa:** com 400M linhas pré-carregadas, rodar LT-1 + consumo de pico e
  medir latência das consultas de idempotência e da escrita de auditoria sob índice grande.

### Stress (ST)
- **ST-1 — Knee point da API:** `ramping-arrival-rate` crescente (ex.: 300 → 1000+ req/s em
  estágios) até p95 > 1.5s **ou** 5xx > 0.5%; registrar a taxa de ruptura.
- **ST-2 — Stress + bureau instável:** ST-1 com perfil de erro do bureau ligado, medindo quanto
  o fallback segura o SLA.

### Contenção (CT) — T3
- **CT-1 — API + consumer concorrentes:** rodar LT-1 (API) **simultaneamente** com SP-1 (pico
  Kafka) no mesmo ambiente; comparar p95/erros/lag contra as execuções isoladas (Δp95). É o
  único teste que expõe a disputa do pool Oracle (20/pod).

## D3. Cenários E2E

| ID | Fluxo | Setup de stub | Asserção |
|----|-------|---------------|----------|
| E2E-1 | Aprovação direta | 3 motores OK | `verdict==APPROVED`, p95<1.5s |
| E2E-2 | Recusa por fraude | `engine-fraud`=risco, score OK | `verdict==DECLINED` |
| E2E-3 | Fallback bureau (T1) | `engine-bureau`=timeout 2s | `verdict==APPROVED_WITH_REVIEW`, p95<1.5s/p99<2.5s |
| E2E-4 | Kafka async | produz em `...requested` | mesmo `traceId` em `...completed` |
| E2E-5 | Idempotência (T5) | reenvia `traceId` | 1 linha de auditoria por `traceId` |

E2E rodam via o motor Gherkin declarativo (HTTP + Kafka) e também como checks dentro dos
cenários de carga, para validar que o veredito permanece correto **sob** carga, não só a frio.

## D4. Thresholds numéricos (ancorados no SLA)

| Teste | Métrica | Threshold | Origem |
|-------|---------|-----------|--------|
| LT-1 / ST-1 | `http_req_duration` | `p(95)<1500`, `p(99)<2500` | SLA API |
| LT-1 / ST-1 | `http_req_failed` | `rate<0.005` | Taxa de erro 5xx < 0.5% |
| LT-1 (longo) | disponibilidade | > 99.9% | SLO |
| LT-2 | lag consumer | `< 10s` | SLO baseline |
| SP-1 | lag de regime (pico) | `< 180s` | **T4 numérico** (ver D5) |
| SP-1 | backlog máximo | `< 750k msgs` | **T4 numérico** |
| SP-1 | tempo total de dreno | `< 900s` (15 min) | Teto contratual |
| E2E-3 | veredito + latência | `APPROVED_WITH_REVIEW` e p95<1500ms | Fluxo 3 / T1 |
| CT-1 | Δp95 concorrente vs solo | reportado; sem violar SLA | T3 |
| E2E-5 | auditoria por `traceId` | `== 1` | Fluxo 5 / T5 |

## D5. Resolução do T4 (meta vaga → testável)

"Aceitável até 15 min" não é threshold. Decomposto em três metas verificáveis, justificadas:
- **Lag de regime < 180s** durante o pico: o sistema deve processar quase na taxa de chegada,
  não só "não cair"; 3 min é folga operacional sem deixar a fila crescer sem limite.
- **Backlog máximo < 750k msgs**: ~833 msg/s por ~15 min de teto = ~750k — o backlog não pode
  exceder o que é drenável dentro do teto contratual.
- **Tempo total de dreno < 900s** após o fim do despejo: materializa os "15 min" como limite
  duro. Número a confirmar com o negócio (pendência P1).

## D6. Estratégia de massa (T5)

- **Unicidade por padrão (RQ-DATA-1):** gerar `traceId` e `customerId` únicos por iteração
  (≥ 95%) — ex.: `traceId = runId + VU + iter`. Reusar `customerId` dispararia idempotência e
  **mascararia a carga de escrita** no Oracle.
- **Fatia de reuso marcada (RQ-DATA-2):** ~5% de `traceId` deliberadamente repetidos, com tag
  própria, para medir o caminho de idempotência **em separado** (métricas segregadas), sem
  poluir o p95 da carga principal.
- **Seed de volume (RQ-DATA-3):** 400M linhas pré-carregadas no Oracle, excluindo os `traceId`
  de teste, para a consulta por índice refletir o tamanho real.
- **Geração:** dados pré-gerados em arquivo (SharedArray) ou derivados determinísticos por VU;
  evitar colisão entre VUs e entre execuções (isolamento — context-base).

## D7. Riscos de medição

1. **Falso verde por modelo fechado (T1):** VUs presos ao bureau reduzem a chegada efetiva e
   escondem a violação do SLA → por isso o modelo aberto em D2.
2. **Acoplamento escondido (T3):** testar API e consumer isolados nunca revela a disputa do pool
   → CT-1 concorrente é obrigatório; reportar Δp95.
3. **Massa que mascara escrita (T5):** reuso de `customerId`/`traceId` ativa idempotência e
   subestima a carga de escrita → unicidade por padrão (D6).
4. **Stub de bureau fixo:** sem injeção de latência/erro, os testes de T1 (E2E-3, SP-2, ST-2)
   medem um mundo irreal → RQ-ENV-2.
5. **Extrapolação QA→prod não-linear:** 4 pods vs 8; pool, broker e bureau não dobram juntos →
   não prometer números de prod a partir de QA sem fator de ajuste explícito.
6. **Lag medido só pelo K6:** o lag real é do consumer group Kafka, não do produtor → coletar do
   lado do alvo (RQ-OBS-1).
7. **Volume ausente no índice:** medir idempotência sem as 400M linhas dá latência otimista.

## D8. Matriz de execução (acionável)

| ID | Tipo | Executor/carga | Threshold-chave | Fluxo/Tensão | Requisito |
|----|------|----------------|-----------------|--------------|-----------|
| E2E-1..5 | E2E | Gherkin + checks | veredito correto | F1–F5 | RQ-E2E-1..5 |
| LT-1 | Load | arrival 300 req/s, 10m | p95<1.5s, 5xx<0.5% | API / T1 | RQ-LOAD-1 |
| LT-2 | Load | Kafka 50 msg/s, 10m | lag<10s | async | RQ-LOAD-2 |
| SP-1 | Spike/Vol | arrival 50→833 msg/s | lag<180s, dreno<900s | T2/T4 | RQ-PEAK-1/2 |
| SP-2 | Spike | erro bureau sob LT-1 | SLA + fallback | T1/F3 | RQ-STRESS-2 |
| VOL-1 | Volume | LT-1 + pico sob 400M | latência idempot./escrita | T3/T5 | RQ-DATA-3 |
| ST-1 | Stress | arrival 300→1000+ | knee point | T1 | RQ-STRESS-1 |
| ST-2 | Stress | ST-1 + bureau instável | SLA sob erro | T1 | RQ-STRESS-2 |
| CT-1 | Contenção | LT-1 + SP-1 juntos | Δp95, sem violar SLA | **T3** | RQ-CONT-1 |

**Ordem sugerida:** E2E (a frio) → LT-1/LT-2 (baseline) → SP-1 → VOL-1 → ST-1/ST-2 → CT-1
(concorrente, por último, pois depende dos baselines isolados para o Δp95).

## D9. Pendências (decisão humana)
- **P1:** confirmar T4 (lag<180s / dreno<900s) com o negócio.
- **P2:** capacidade-alvo máxima para calibrar o knee point do stress.
- **P3:** disponibilidade em QA de stub injetável do bureau e seed de 400M (bloqueante para
  E2E-3, SP-2, ST-2, VOL-1).
- **P4:** fator de ajuste aceito para extrapolar QA (4 pods) → prod (8 pods).
- **P5:** retenção/expurgo da auditoria — fora de escopo do plano, mas afeta a fidelidade do
  volume ao longo do tempo.
