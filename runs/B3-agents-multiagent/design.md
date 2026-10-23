# Design — Plano de testes K6 para Veridia (B3: spec do Kiro, multi-agente)

> **Este é o artefato de planejamento de testes avaliado.** Consolida as contribuições de três
> subagentes isolados (QA Planner, Backend/Integration Engineer, QA Performance Engineer),
> arbitradas pelo Orchestrator. Ancorado em `scenario/target-app.md` (F1–F5, SLAs, T1–T5).
> Foco é **planejamento**, não implementação — trechos de `options`/`thresholds` são
> ilustrativos.

## D0. Modelo mental do alvo (gargalos dominantes)

Consolidado da análise do Backend/Integration Engineer, confirmado contra o alvo:

1. **`engine-bureau` (downstream lento/instável) — T1.** O veredito síncrono só sai quando os
   3 motores retornam; o relógio da requisição é limitado pelo mais lento até o timeout de 2s.
   O p95 do bureau (~1.2s) sozinho consome ~80% do orçamento de 1.5s. Sem retry, sem cache.
   É o **primeiro** a quebrar e está fora do nosso controle.
2. **Pool Oracle (20/pod) compartilhado — T3.** Com **Virtual Threads**, threads são baratas;
   o limitador real sob bloqueio passa a ser o **pool de conexões**. A escrita de auditoria da
   API e o `SELECT` de idempotência + escrita do consumer competem pelas mesmas 20 conexões/pod.
   Exaustão do pool vira espera de aquisição que **empilha** sobre a latência de downstream →
   p95/p99 estouram mesmo com o bureau saudável. É o acoplamento que torna "sync" e "async"
   **não** independentes.
3. **Folga estrutural do SLA — T1.** p95<1.5s abaixo do timeout de 2s: não há folga
   arquitetural.
4. **Teto do consumer vs pico Kafka — T2.** 2M/40min ≈ **833 msg/s** (~16× o normal de 50);
   12 partições ⇒ ~70 msg/s por partição; cada mensagem é limitada pelo pool.
5. **Tabela de auditoria ~400M sem expurgo** — custo de escrita cresce, segura conexão por mais
   tempo, realimenta (2).
6. **Taxa de fallback silenciosa** — bureau 5xx/timeout desloca o mix de vereditos para
   `APPROVED_WITH_REVIEW`; sem medir, passa-se no SLA de latência degradando a qualidade de
   decisão.

## D1. Seleção de tipos de teste (inclusão/exclusão justificada)

| Tipo | Decisão | Justificativa ancorada no alvo |
|------|---------|--------------------------------|
| **E2E** | **Incluir** | Validar F1–F5 (veredito correto, não só HTTP 200) incl. fallback e idempotência; âncora de correção para os testes de carga. |
| **Load** | **Incluir** | Provar p95<1.5s / p99<2.5s / 5xx<0.5% a ~300 req/s sustentado — a condição de produção declarada. |
| **Stress** | **Incluir** | SLA tem folga mínima (T1); precisamos do *knee point* acima de 700 req/s e de qual recurso satura primeiro (pool vs bureau). |
| **Spike** | **Incluir** | Dois picos reais: salto síncrono rumo a ~700 req/s e, sobretudo, o salto de chegada Kafka 50→833 msg/s (T2). |
| **Soak** | **Incluir (acoplado ao pico Kafka)** | O pico de ~40 min **é** um soak: contenção de pool, custo de escrita na tabela de 400M e GC/pinning acumulam; lag bom no minuto 5 pode divergir no minuto 35. Não há SLA de duração "pura", então soak isolado não se justifica — o soak vive dentro do Teste E. |
| **Volume** | **Incluir como pré-condição, não como forma de tráfego** | O "volume" relevante é o **estado de dados**: 400M linhas de auditoria + massa de `traceId` únicos. Volume mal modelado (reuso de chave) mascara a escrita (T5). Então volume é condição de dados dos testes E/F, não um teste de tráfego separado. |

Nada é descartado integralmente — o alvo exercita todos os tipos; a decisão foi **fundir**
soak e volume nos cenários que os tornam significativos, não inventá-los como corridas soltas.

## D2. Modelo de carga: aberto (arrival-rate) vs fechado (VU)

**Decisão: modelo ABERTO (taxa de chegada) para todos os testes de throughput/SLA.** Exceção
única: o teste de corrida de idempotência (Teste G), onde a **simultaneidade** é a grandeza
sob teste.

Justificativa (arbitragem do Orchestrator sobre o item de debate levantado pelo Backend):
- **Coordinated omission.** Um modelo fechado (VUs) recua exatamente quando o sistema sofre: se
  o bureau fica lento, cada VU espera e **envia menos**, baixando a carga real e **escondendo**
  os gargalos (1) e (2). Isso produz *falso verde*. O modelo aberto mantém a taxa de chegada
  independentemente da latência, revelando a quebra.
- **Little's Law (L = λ × W).** A 300 req/s com latência média ~1.3s (dominada pelo bureau),
  as requisições em voo ≈ `300 × 1.3 ≈ 390`. Dimensionamos o pool de VUs do K6 com folga para
  não auto-limitar: `preAllocatedVUs: 500`, `maxVUs: 800`. Se o K6 ficar sem VUs, aparece como
  `dropped_iterations` — que tratamos como **falha de teste** (guarda anti-omission), não como
  resultado.
- O caminho Kafka também é aberto: dirigir **taxa de chegada**, não despejar um batch fixo o
  mais rápido possível (isso mediria o produtor/broker, não o dreno do consumer).

## D3. Cenários E2E (F1–F5) — correção primeiro

| ID | Fluxo | Cenário | Asserção de aceite |
|----|-------|---------|--------------------|
| E2E-1 | F1 | 3 motores OK | `verdict==APPROVED`; 1 auditoria com latências por motor; dentro do SLA |
| E2E-2 | F2 | fraude sinaliza, score bom | `verdict==DECLINED` (fraude domina) |
| E2E-3 | F3 | bureau timeout (>2s) | `verdict==APPROVED_WITH_REVIEW`; **sem 5xx** ao chamador; auditoria marca fallback |
| E2E-4 | F3 | bureau 5xx | fallback dispara (não só por timeout) |
| E2E-5 | F4 | pedido via `credit.assessment.requested` | resultado em `credit.assessment.completed`, **mesmo `traceId`**, 1:1 (xk6-kafka) |
| E2E-6 | F1↔F4 | mesma entrada sync e async | vereditos consistentes (sujeito a RQ-OPEN-5) |
| E2E-7 | F5 | `traceId` duplicado sequencial | 1 única auditoria |

Os E2E rodam isolados (baixa carga, correção) **e** como `checks` embutidos sob carga nos
testes B/E/F, para pegar regressão de veredito sob stress (ex.: fallback que vira erro).

## D4. Cenários de carga/performance (Testes A–G) — modelagem e thresholds

Alvos de carga: **~300 req/s sustentado**, **pico ~700 req/s**; Kafka **50→833 msg/s**.

### Teste A — E2E funcional (correção, baixa carga)
- **Executor:** `per-vu-iterations` ou `shared-iterations`, poucos VUs. Objetivo é correção, não vazão.
- **Mocks:** latência fixa aceitável (E2E puro). Bureau em presets p/ E2E-3/E2E-4.
- **Thresholds:** `checks: ['rate==1.0']`; `http_req_failed: ['rate<0.005']`.

### Teste B — Load sustentado (SLA em produção)
- **Executor:** `constant-arrival-rate`, `rate: 300`, `timeUnit: '1s'`, `duration: '30m'`,
  `preAllocatedVUs: 500`, `maxVUs: 800`. Warm-up 2–3 min descartado da agregação.
- **Mock bureau:** `bureau-healthy` (distribuição realista p95~1.2s).
- **Thresholds:**
  ```js
  thresholds: {
    http_req_duration: ['p(95)<1500', 'p(99)<2500'],
    http_req_failed:   ['rate<0.005'],
    dropped_iterations:['count<1'],   // guarda anti coordinated-omission
  }
  ```

### Teste C — Stress / breakpoint (knee point)
- **Executor:** `ramping-arrival-rate`, estágios subindo **além de 700 req/s** (ex.: 300→500→
  700→900→1100), cada degrau 5–8 min para o pool/downstream reagir.
- **Objetivo:** achar onde p95/p99 ou 5xx rompem o SLA e **nomear o recurso que satura**
  (pool Oracle, bureau, CPU, broker). Sem threshold de *pass/fail* fixo — o resultado é o knee.
- **Métricas-chave:** p95/p99 por degrau, 5xx, espera de aquisição do pool (lado alvo).

### Teste D — Spike síncrono
- **Executor:** `ramping-arrival-rate`, salto rápido baseline→~700+ req/s, hold curto, retorno.
- **Thresholds:** medir degradação e **recuperação**; `dropped_iterations: ['count<1']`;
  observar p99 durante o salto (risco de HPA lag no início).

### Teste E — Pico Kafka (spike de chegada + soak) — T2
- **Executor (xk6-kafka, modelo aberto):** `ramping-arrival-rate` sobre o produtor —
  50 → ~833 msg/s, **hold ~40 min**, drop para 50, janela de dreno.
- **Ambiente:** **12 partições** (pré-condição RQ-MEAS-4).
- **Thresholds (lag, T4 — ver D4.1):**
  ```js
  thresholds: {
    kafka_consumer_lag_ms:        ['max<900000'],   // linha vermelha 15 min
    kafka_drain_after_peak_ms:    ['max<600000'],   // dreno ≤10 min pós-pico
    kafka_msgs_lost:              ['count==0'],      // perda zero
    // alvo interno (amber/green): lag máx ≤ 300000 (5 min)
  }
  ```
- **Nota:** lag é lido do **lado do alvo** (`records-lag-max`/time-lag por partição), não só do K6.

### Teste F — Contenção combinada (T3) — **bloqueante de release**
- **Executor:** rodar **Teste B (300 req/s sync)** E **Teste E (pico Kafka 833 msg/s)**
  **simultaneamente** no mesmo ambiente, contra o mesmo pool.
- **Objetivo:** expor a contenção do pool Oracle que os testes isolados escondem.
- **Thresholds + comparação:**
  ```js
  thresholds: {
    http_req_duration: ['p(95)<1500', 'p(99)<2500'], // SLA sync DEVE sobreviver ao pico
    http_req_failed:   ['rate<0.005'],
    kafka_consumer_lag_ms: ['max<900000'],
  }
  ```
  Comparar **Δp95 = p95(F) − p95(B isolado)**; critério de alerta: Δp95 significativo indica
  acoplamento perigoso (recomendação: Δp95 ≤ 150ms como meta interna, a confirmar com C_max).
- **Observabilidade obrigatória (lado alvo):** espera/exaustão de pool por pod, lag por
  partição, taxa de fallback do bureau, pinning de VT.

### Teste G — Corrida de idempotência concorrente (F5) — exceção de modelo fechado
- **Executor:** `shared-iterations`/`per-vu-iterations` com **N VUs disparando o mesmo
  `traceId` ao mesmo tempo** (a simultaneidade é o objeto, não a vazão).
- **Thresholds:** após a corrida, **exatamente 1** registro de auditoria por `traceId`;
  nenhuma conclusão conflitante republicada.

### D4.1 — Threshold numérico para T4 (substitui a meta vaga)

Proposta consolidada do QA Performance Engineer, a ratificar por humano (RQ-OPEN-1):
- **Linha vermelha:** `time-lag` máximo durante o pico `≤ 15 min (900000 ms)` — honra o "até
  15 min" declarado, agora mensurável.
- **Companheiro (recuperação):** tempo para o lag voltar a `<10s` após o fim do pico
  `≤ 10 min (600000 ms)` — garante que o lag **limpa antes da meia-noite** (janela 23h–00h).
- **Integridade:** perda zero + correlação 1:1 de `traceId`.
- **Alvo interno (amber/green):** `≤ 5 min (300000 ms)` — folga operacional antes da linha
  vermelha contratual.

Racional: "até 15 min" só é defensável se o lag **drena** dentro da janela; por isso o par
linha-vermelha + tempo-de-dreno. O alvo de 5 min dá margem para variação de produção (8 pods,
bureau pior) sem estourar o contrato.

## D5. Riscos de medição (onde o teste pode medir a coisa errada)

1. **Falso verde por modelo fechado.** VUs recuam sob latência do bureau e escondem (1)/(2).
   → Mitigação: modelo aberto + `dropped_iterations<1` (D2).
2. **Acoplamento escondido por teste isolado (T3).** Testar sync e Kafka separados passa, mas
   produção acopla no pool. → Mitigação: Teste F obrigatório + Δp95 isolado vs combinado.
3. **Massa que mascara a escrita (T5).** Reuso de `traceId` dispara idempotência e mede o
   atalho, não a escrita real. → Mitigação: unicidade por padrão (D6).
4. **Mock de latência fixa.** Esconde a variância/erros do bureau que **causam** a quebra do
   SLA. → Mitigação: mock programável com distribuição + 5xx + deslocamento (RQ-MEAS-2).
5. **Tabela de auditoria vazia em QA.** Escrita parece mais rápida que prod, segura conexão
   menos tempo, **subestima** a contenção. → Mitigação: seed de 400M (RQ-DATA-3).
6. **Extrapolação QA(4 pods)→prod(8 pods) não-linear.** Pool e downstream não escalam com pods.
   → Mitigação: reportar **por pod**, nunca projetar ×2 (RQ-MEAS-3).
7. **Degradação silenciosa por fallback (#6).** Passar no SLA de latência enquanto a taxa de
   `APPROVED_WITH_REVIEW` sobe. → Mitigação: medir taxa de fallback como métrica de primeira
   classe; teto a definir (RQ-OPEN-3).
8. **Pinning de Virtual Thread / GC sob soak.** Pinning em seções bloqueantes/`synchronized`
   reintroduz um limitador de thread e apaga o benefício de VT. → Mitigação: observar eventos
   de pinning e GC durante o Teste E/F.
9. **HPA lag no início do spike.** Autoscaling atrasa no salto do Teste D/E → p99 transitório.
   → Mitigação: anotar nº de pods/HPA na linha do tempo.

## D6. Estratégia de dados de massa (T5)

- **Unicidade por padrão (RQ-DATA-1):** `traceId` e `customerId` únicos por iteração — UUID ou
  `${__VU}-${__ITER}-${runSalt}` — para testes B/C/D/E/F. Exerce o caminho real de escrita.
- **Fração de reuso tagueada (RQ-DATA-2):** ~1–2% de `traceId` deliberadamente repetidos e
  **marcados** (tag K6) para exercer a idempotência (E2E-7/Teste G), **excluídos** dos
  thresholds de latência de SLA para não contaminar o p95.
- **Seed de auditoria (RQ-DATA-3):** pré-popular a tabela em ~400M linhas com profundidade de
  índice fiel; os `traceId` de teste **não** devem colidir com o seed (senão ativam idempotência
  falsa). Marcar como pré-condição bloqueante dos testes de volume/contenção.

## D7. Mocks e ambiente (fidelidade)

- **score/fraude:** mocks de **distribuição** (log-normal em torno do p95 declarado, cauda
  realista). Fixo só para E2E funcional.
- **bureau:** mock **programável** com presets: `bureau-healthy` (p95~1.2s), `bureau-slow`
  (p95→1.8s), `bureau-flaky` (10–30% 5xx), `bureau-brownout` (latência+erros), com capacidade
  de **deslocar a distribuição no meio do teste** (simular degradação de horário de pico sob
  carga). Sem isso, os testes de bureau não validam T1.
- **Kafka:** 12 partições em QA (paridade com prod). 
- **Pods:** QA 4 vs prod 8 — reportar por pod; não projetar linear (RQ-MEAS-3).

## D8. Matriz consolidada (teste × carga × threshold × fluxo/tensão × requisito)

| Teste | Tipo | Executor / carga | Threshold-chave | Fluxo/Tensão | Requisito |
|-------|------|------------------|-----------------|--------------|-----------|
| A | E2E | per-vu-iterations, baixo | `checks==1.0`, 5xx<0.005 | F1–F5 | RQ-E2E-1..6 |
| B | Load | const-arrival 300/s, 30m | p95<1500, p99<2500, 5xx<0.005, dropped<1 | F1/F3, T1 | RQ-PERF-1/2/5 |
| C | Stress | ramping-arrival >700/s | knee point + recurso saturado | T1/T3 | RQ-PERF-4 |
| D | Spike | ramping-arrival →700+/s | degradação+recuperação, dropped<1 | pico sync | RQ-PERF-3 |
| E | Spike+Soak | ramping-arrival Kafka 50→833/s, 40m+dreno | lag<900000, dreno<600000, perda==0 | F4, T2/T4 | RQ-KAFKA-1/2/3 |
| F | Combinado | B + E simultâneos | SLA sync sobrevive; Δp95; pool | T3 | RQ-T3-1/2 |
| G | Idempotência | VUs concorrentes mesmo traceId (fechado) | 1 auditoria/traceId | F5 | RQ-E2E-6 |

## D9. Ordem de execução sugerida

1. **A** (correção) — porta de entrada; nada adianta medir performance se o veredito está errado.
2. **B** (baseline de SLA, bureau saudável) — âncora.
3. **G** (idempotência) — barato, independente.
4. **E** (pico Kafka isolado) — caracteriza o dreno e o lag.
5. **C/D** (stress/spike sync) — knee point e recuperação.
6. **F** (combinado, bloqueante de release) — por último, com todas as observabilidades ligadas.
Repetir B/E/F com presets `bureau-slow`/`bureau-flaky` para exercer T1.

## D10. Pendências (decisão humana) — ver `requirements.md` §7

T4 (ratificar 15 min/5 min e a referência de medição), SLA do fallback (mesmo p95/p99?), teto
de taxa de fallback, janela de medição de disponibilidade/5xx, paridade sync/async, escopo da
idempotência, **C_max** (bloqueante para stress e metas de lag), e disponibilidade de seed de
400M + stub injetável do bureau em QA (pré-condições bloqueantes).
