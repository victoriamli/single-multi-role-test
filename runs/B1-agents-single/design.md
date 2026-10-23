# Plano de testes — Veridia (gateway de decisão de crédito)

> Braço **B1 — Spec do Kiro + single-role**. Este `design.md` é o artefato de planejamento
> (onde o plano de testes mora). Executor: K6 (binário/Docker), scripts versionados em pasta
> interna do projeto de QA. Motor E2E declarativo (Gherkin, HTTP + Kafka via xk6-kafka).
> Ambiente base: QA com **4 pods**; produção com **8 pods**. Foco: planejamento, não scripts.
>
> Rastreabilidade: cada decisão referencia fluxos F1–F5 e tensões T1–T5 de `target-app.md` e
> os requisitos `RQ-*` de `requirements.md`.

---

## D1. Tipos de teste: escolhidos e descartados

| Tipo | Decisão | Justificativa ancorada no alvo |
|------|---------|--------------------------------|
| **E2E** | **Usar** | Validar os 5 fluxos (F1–F5) incl. fallback do bureau (F3) e idempotência (F5), com asserção de **veredito**, não só status HTTP. Motor Gherkin cobre HTTP e Kafka. |
| **Load** | **Usar** | SLA só faz sentido sob a carga sustentada de produção (~300 req/s) e lag Kafka fora do pico (<10s). É o teste de conformidade do SLA (RQ-LOAD-1/2). |
| **Stress** | **Usar** | T1 (SLA apertado vs. bureau lento) e T3 (pool Oracle) só revelam o ponto de ruptura acima do pico (~700 req/s). Buscar o *knee* (RQ-STRESS-1..3). |
| **Spike** | **Usar** | T2: o batch noturno (2M msgs/40min) e o salto 300→700 req/s na API são picos abruptos reais, não rampa suave (RQ-SPIKE-1/2). |
| **Soak** | **Usar (focado)** | A **drenagem** do pico Kafka e a estabilidade de recursos (pool/memória) sob carga prolongada são o risco de T3/T4. Soak aqui é dirigido à drenagem, não "rodar por horas sem objetivo" (RQ-SOAK-1/2). |
| **Volume** | **Usar** | T3+T5: auditoria de ~400M linhas muda o custo da escrita e da consulta de idempotência; precisa ser testado com a tabela realmente grande (RQ-VOL-1/2). |

**Nada foi integralmente descartado**, porque o alvo exercita todos os seis tipos. O que foi
**despriorizado**: soak "puro" de longa duração sem foco em drenagem (baixo retorno frente ao
custo); e teste de stress do bureau isolado do resto (a instabilidade do bureau só importa
dentro do fluxo síncrono sob carga — por isso vira RQ-STRESS-3 acoplado, não um teste à parte).

---

## D2. Modelo de carga: por que **open model** (arrival-rate)

**Decisão:** usar executores de **taxa de chegada** (`constant-arrival-rate` /
`ramping-arrival-rate`) para todos os testes de carga/stress/spike da API síncrona, **não**
`ramping-vus`.

**Justificativa (T1, RQ-MEAS-2):** o `engine-bureau` tem p95 ~1.2s e é instável. Num modelo
fechado (VUs), quando o bureau fica lento os VUs ficam **presos** aguardando resposta, a taxa
oferecida **cai sozinha** e os percentis parecem bons — exatamente o cenário que mascara a
violação do SLA. O open model mantém a taxa oferecida fixa independentemente da latência
downstream, expondo o enfileiramento real e os `dropped_iterations` quando o sistema não
acompanha. É o único modelo que mede honestamente o SLA sob um downstream lento.

**Dimensionamento de VUs (Little's Law):** VUs em voo ≈ taxa × latência média. Para 300 req/s
com latência média conservadora de ~1.0s (puxada pelo bureau), ~300 iterações concorrentes;
com folga para caudas, `preAllocatedVUs ≈ 400` e `maxVUs ≈ 800` no teste de load; no stress a
~700 req/s, `preAllocatedVUs ≈ 900`, `maxVUs ≈ 1500`. A JVM usa **Virtual Threads**, então o
lado servidor suporta muitas chamadas bloqueantes concorrentes — o limite esperado migra para
o **pool Oracle (20/pod)** e para o bureau, não para threads de plataforma.

**Kafka:** produção de mensagens via **xk6-kafka** com perfil de arrival-rate próprio
(`ramping-arrival-rate` de mensagens/s), medindo lag como métrica customizada, não VUs HTTP.

---

## D3. Cenários K6 e perfis de carga (estágios)

### LT-1 — Load API síncrona (RQ-LOAD-1, RQ-LOAD-3)
- Executor: `ramping-arrival-rate`. Estágios: 0→300 req/s em 3 min (ramp), **300 req/s por
  30 min** (steady), 300→0 em 2 min.
- Mix de veredito: ~80% F1 (aprovação), ~12% F2 (fraude), ~8% F3 (degradação bureau induzida),
  para que os percentis reflitam o caminho crítico real e não só o caso rápido.
- Massa: `traceId`/`customerId` únicos (RQ-DATA-1).

### LT-2 — Load consumer Kafka fora do pico (RQ-LOAD-2)
- Produção constante de ~50 msg/s por 30 min via xk6-kafka em `credit.assessment.requested`.
- Métrica: lag do consumer group (12 partições) < 10s sustentado.

### ST-1 — Stress API: busca do knee (RQ-STRESS-1, RQ-STRESS-3)
- Executor: `ramping-arrival-rate`. Estágios escalonados: 300 → 500 → 700 → 900 → 1100 req/s,
  patamares de 5 min cada. Injeção de instabilidade crescente no bureau em patamares altos.
- Objetivo: localizar a taxa em que p95 cruza 1.5s **ou** 5xx cruza 0.5%; reportar taxa e
  requisições em voo no knee.

### CT-1 — Contenção concorrente API + Kafka (T3, RQ-STRESS-2, RQ-MEAS-1)
- Rodar LT-1 (API a 300 req/s) **simultaneamente** com um fluxo Kafka a 50 msg/s e, numa
  segunda fase, com o pico Kafka ativo. Comparar p95 da API com LT-1 isolado → **Δp95**.
- Objetivo: quantificar o acoplamento via pool Oracle (20/pod) que o teste isolado esconde.

### SP-1 — Spike Kafka noturno (T2, T4, RQ-SPIKE-1, RQ-SOAK-1)
- Injetar ~2M mensagens em ~40 min (≈ 833 msg/s de pico) via xk6-kafka, com a API sob carga
  nominal concorrente. Medir backlog máximo, tempo de drenagem até lag < 10s, perda de msgs.

### SP-2 — Spike API (RQ-SPIKE-2)
- `ramping-arrival-rate` com salto abrupto 300→700 req/s mantido 5 min, depois volta a 300.
  Medir janela de recuperação do p95 ao SLA e 5xx durante/após o pico.

### VT-1 — Volume + idempotência (T3, T5, RQ-VOL-1/2, RQ-E2E-5)
- **Pré-condição:** tabela de auditoria semeada com ~400M linhas (seed com `traceId`
  distintos — RQ-DATA-3). Rodar escrita concorrente de pico + fração marcada de `traceId`
  repetidos (RQ-DATA-2) para medir custo real da consulta de idempotência e da escrita.

### SK-1 — Soak de estabilidade (RQ-SOAK-2)
- Carga nominal (300 req/s) por ≥ 2–4 h, observando deriva de latência/erro/conexões sem
  mudança de carga. Secundário; roda quando houver janela.

---

## D4. Thresholds numéricos (ancorados nos SLAs)

Todos como `thresholds` K6 — critério de parada objetivo (RQ-MEAS-4).

| Teste | Métrica | Threshold | Origem |
|-------|---------|-----------|--------|
| LT-1 / SP-2 | `http_req_duration` | `p(95)<1500` e `p(99)<2500` (ms) | SLA API |
| LT-1 / ST-1 | taxa 5xx (`http_req_failed` filtrado) | `rate<0.005` | SLA erro < 0.5% |
| LT-1 | disponibilidade (checks OK) | `rate>0.999` | SLA 99.9% |
| E2E F3 | veredito = `APPROVED_WITH_REVIEW` em timeout do bureau | `rate>0.99` dos casos injetados | F3 |
| LT-2 | lag Kafka fora do pico (métrica custom) | `max<10000` (ms) | SLO lag normal |
| SP-1/SK Kafka | **lag no pico** | proposto **< 5 min** (300.000 ms); backlog máx **< 750k msgs**; drain até <10s **< 10 min** | ver D4.1 (T4) |
| CT-1 | Δp95 (concorrente − isolado) | reportar; alerta se `>150 ms` | T3 |
| VT-1 | latência consulta idempotência | estável vs. baseline (desvio `<20%`) | T3/T5 |

### D4.1 — Transformando T4 ("aceitável até 15 min") em threshold testável
A meta oficial é vaga. **Proposta:** adotar **lag de pico < 5 min** como alvo de teste, com
15 min como **teto absoluto de falha**. Racional: a meta informal de 15 min é o limite de
tolerância do negócio; testar contra o limite não dá margem de segurança nem detecta
degradação precoce. Fixar o alvo em 1/3 do teto (5 min) cria sinal de alarme antes do estouro
e é defensável pela matemática do pico: 2M msgs / 40 min ≈ 833 msg/s de entrada vs. capacidade
de consumo a validar; se a vazão sustentada do consumer (a medir em ST/SP) for ≥ ~1.100 msg/s,
o backlog acumulado drena em poucos minutos após o fim da injeção. **Pendência para humano**
(ver D7): confirmar 5 min como alvo contratual.

---

## D5. Riscos de medição (onde o teste pode medir a coisa errada)

1. **Falso verde por modelo fechado (T1).** VUs presos no bureau lento reduzem a taxa
   oferecida e "limpam" os percentis. Mitigação: open model + monitorar `dropped_iterations`
   e a taxa efetivamente entregue vs. alvo (RQ-MEAS-2).
2. **Acoplamento escondido pelo teste isolado (T3).** Medir API e Kafka separados dá p95 bom
   e some com a disputa pelo pool Oracle. Mitigação: CT-1 concorrente + Δp95 obrigatório
   (RQ-MEAS-1).
3. **Massa que mascara a carga de escrita (T5).** Reusar `customerId`/`traceId` dispara
   idempotência (caminho de leitura barato) em vez de escrita. Mitigação: unicidade por
   padrão; repetição só na fração marcada (RQ-DATA-1/2).
4. **Volume ausente no ambiente de teste.** Rodar idempotência contra tabela pequena não
   reproduz o custo de 400M linhas. Mitigação: seed de volume como pré-condição dura de VT-1
   (RQ-DATA-3).
5. **Extrapolar QA (4 pods) para produção (8 pods) linearmente (RQ-MEAS-3).** Pool Oracle,
   bureau e Kafka não escalam proporcionalmente ao nº de pods. Mitigação: reportar resultados
   como "QA, 4 pods" e marcar a projeção de produção como não-linear/aberta.
6. **Estabilidade do estímulo do bureau.** Se o stub do bureau não reproduzir fielmente os
   picos de timeout/5xx, F3 e ST-1/RQ-STRESS-3 medem um comportamento irreal. Mitigação:
   stub com injeção de latência/erro configurável e documentada.

---

## D6. Estratégia de dados de massa (T5)

- **Geração runtime:** `traceId` único por iteração (UUID) e `customerId` sorteado de um
  espaço grande (milhões), por padrão → mede a carga real de escrita (RQ-DATA-1).
- **Fração de idempotência:** ~2–5% das iterações reinjetam um `traceId` já usado, **tagueado**
  (`tag: idempotent=true`), para medir o caminho de idempotência isoladamente sem poluir os
  percentis de escrita (RQ-DATA-2).
- **Seed de volume:** ~400M linhas pré-carregadas com `traceId` de namespace separado
  (ex.: prefixo `seed-`) para não colidir com a massa runtime (RQ-DATA-3).
- **Isolamento entre execuções:** cada run usa um namespace/prefixo de `traceId` próprio para
  não corromper dados de outra execução nem ativar idempotência cruzada acidental.

---

## D7. Pendências que dependem de decisão humana

1. **T4:** confirmar o alvo de lag de pico (proposta: < 5 min; teto 15 min). Bloqueia o
   threshold de SP-1/SK Kafka.
2. **Escala QA→produção:** há ambiente de QA com volume e stub de bureau fiéis? Sem isso,
   VT-1 e F3 degradam para aproximações.
3. **Observabilidade do pool Oracle:** há métrica exportável de uso do pool (20/pod) para
   correlacionar com o Δp95 de CT-1? Senão, T3 fica inferido, não medido.
4. **Capacidade-alvo máxima:** definir até qual taxa o stress deve ir (limite de 1100 req/s é
   proposta; o negócio pode querer o ponto de ruptura real).
5. **Retenção/expurgo da auditoria:** fora de escopo do plano, mas o crescimento sem expurgo
   afeta a validade futura dos thresholds de VT-1.

---

## D8. Matriz de rastreabilidade (acionável)

| Teste | Tipo | Carga/executor | Threshold-chave | Fluxo/Tensão | Requisito |
|-------|------|----------------|-----------------|--------------|-----------|
| E2E-1..5 | E2E | Gherkin HTTP+Kafka | veredito correto + SLA no F1/F3 | F1–F5 | RQ-E2E-1..5 |
| LT-1 | Load | arrival 300 req/s, 30 min | p95<1.5s, p99<2.5s, 5xx<0.5% | T1 | RQ-LOAD-1/3 |
| LT-2 | Load | Kafka 50 msg/s | lag<10s | — | RQ-LOAD-2 |
| ST-1 | Stress | arrival 300→1100 req/s | achar knee (p95=1.5s / 5xx=0.5%) | T1 | RQ-STRESS-1/3 |
| CT-1 | Stress/contention | API+Kafka concorrentes | Δp95 reportado (alerta >150ms) | T3 | RQ-STRESS-2, RQ-MEAS-1 |
| SP-1 | Spike | Kafka 2M/40min + API nominal | lag<5min, backlog<750k, drain<10min | T2/T4 | RQ-SPIKE-1, RQ-SOAK-1 |
| SP-2 | Spike | API 300→700 req/s | recuperação do p95, 5xx<0.5% | — | RQ-SPIKE-2 |
| VT-1 | Volume | 400M seed + escrita de pico | idempotência estável, SLA mantido | T3/T5 | RQ-VOL-1/2 |
| SK-1 | Soak | API 300 req/s, 2–4 h | sem deriva de latência/erro | — | RQ-SOAK-2 |

**Ordem de execução sugerida:** E2E (correção) → LT (baseline de SLA) → ST (knee) → CT
(acoplamento) → SP (picos) → VT (volume) → SK (estabilidade). Baselines isolados antes dos
concorrentes, para que o Δp95 tenha referência.
