## Context

**Veridia** é um gateway síncrono de decisão de crédito (Kotlin/Spring Boot, Virtual Threads,
K8s — 4 pods QA / 8 pods prod) com um caminho assíncrono por Kafka. O plano de testes K6
precisa validar os 5 fluxos de negócio e expor os pontos de ruptura reais do alvo. Os números
do alvo que ancoram todo o plano:

- API `POST /v1/assessments`: 3 motores downstream — `engine-score` (p95 ~80ms), `engine-fraud`
  (p95 ~250ms), `engine-bureau` (p95 ~1.2s, **instável**). Timeout 2s por downstream, **sem
  retry**. SLA p95 < 1.5s, p99 < 2.5s, 5xx < 0.5%, disponibilidade 99.9%. Throughput ~300 req/s
  sustentado, ~700 req/s de pico.
- Consumer Kafka `credit.assessment.requested` → `credit.assessment.completed`. Pico noturno
  ~2M msgs em ~40 min (~830 msg/s) vs. ~50 msg/s normal. 12 partições. Idempotência por
  `traceId` (SELECT Oracle antes de processar).
- Oracle: auditoria com ~400M linhas sem expurgo; escrita concorrente no pico compete com a API
  no **mesmo pool (20 conexões/pod)**.

Este design é produzido pela técnica **multi-role**: QA Planner, QA Performance Engineer e
Backend/Integration Engineer propõem num único contexto, e o Orchestrator encena a crítica
cruzada e consolida. O registro conciso do debate está na seção **Decisions** (cada decisão
traz a proposta, a divergência e a resolução).

## Goals / Non-Goals

**Goals:**
- Cobrir os 5 fluxos E2E com critérios de aceite mensuráveis.
- Endereçar explicitamente as tensões T1–T5 com decisões ancoradas no alvo.
- Entregar modelagem de carga K6 (executor, perfil, estágios) e thresholds numéricos
  derivados do SLA para cada teste.
- Transformar a meta vaga de lag no pico (T4) em threshold testável e justificado.
- Expor, por teste concorrente, a disputa do pool Oracle (T3) que testes isolados escondem.

**Non-Goals:**
- Implementar os scripts K6 completos (apenas estratégia e trechos curtos de `options`/
  `thresholds`).
- Redesenhar o alvo (não adicionar cache ao bureau, retry, ou segundo pool — testa-se como é).
- Testes de segurança/penetração ou funcionais de regra de negócio além dos 5 fluxos.
- Tuning de infraestrutura (sizing de pods, partições) — o plano mede, não ajusta.

## Decisions

### Resumo do debate multi-role (entrada para as decisões)

**QA Planner (o quê e por quê).** Os 5 fluxos viram cenários E2E; os críticos de risco são o
fallback do bureau (fluxo 3) e a idempotência (fluxo 5). Prioridade de risco: T1 (SLA vs
bureau) e T3 (pool) são os que podem derrubar produção; T5 é risco de *medição* (massa ruim
invalida o teste de volume). Aceite de negócio por fluxo, não só status HTTP.

**QA Performance Engineer (como testar em K6).** Load da API deve usar `constant-arrival-rate`
(não `ramping-vus`) para desacoplar carga de latência — se o bureau fica lento, VUs ficariam
presos e a carga cairia, mascarando o problema. Thresholds direto do SLA. Pico Kafka via
`ramping-arrival-rate` sobre xk6-kafka. Stress em degraus de arrival-rate até o knee point.

**Backend/Integration Engineer (gargalos e viabilidade).** O que satura primeiro quase nunca é
a CPU: é o pool Oracle (20/pod) ou o downstream. Testar caminhos isolados dá um número
otimista e falso — o acoplamento do pool só aparece com API+Kafka concorrentes. Mocks dos
motores precisam injetar latência/erro, senão T1 não é testável. Virtual Threads não "resolvem"
bloqueio em pool finito: sob pico, a espera por conexão vira o gargalo.

---

### D1 — Tipos de teste: incluir E2E, load, stress, spike, volume; soak parcial

Proposta (QA Perf): E2E + load + stress + spike + volume. QA Planner questionou se soak
(horas) seria necessário dado o vazamento potencial da tabela Oracle que só cresce.
Backend ponderou: o pico real dura ~40 min, não horas — soak clássico não reflete o alvo; o
risco de crescimento da tabela é estrutural (400M linhas já existem), não algo que um soak de
K6 de horas revelaria melhor que o teste de volume + seed realista.

**Resolução (Orchestrator):** incluir **E2E, load, stress, spike, volume**. **Soak** entra
apenas como *extensão opcional* do volume Kafka (sustentar ~300 msg/s por 2–4h) para detectar
degradação lenta de pool/GC, marcado como desejável, não bloqueante. Justificativa ancorada:
o pico é curto (volume+spike), o acoplamento é o risco dominante (concorrência), e soak longo
não é o modo de falha característico do alvo.

### D2 — Executor da API: `constant-arrival-rate`, não `ramping-vus` (T1)

Divergência: um teste orientado a VUs é mais simples, mas o Backend alertou que, com o bureau
lento, VUs presos reduzem a taxa efetiva e **escondem** a violação de SLA (o sistema parece
"aguentar" porque pára de pedir).

**Resolução:** load e stress da API usam executores de **arrival-rate**
(`constant-arrival-rate` para o sustentado, `ramping-arrival-rate` para pico/stress), com
`preAllocatedVUs`/`maxVUs` folgados para que a fila de chegada não seja limitada por VUs. Isso
mede a latência real sob taxa fixa — o comportamento que o SLA contratual descreve.

### D3 — T1 (SLA apertado vs bureau): stress em torno do downstream, não só da API

QA Planner: o risco é o veredito estourar o SLA por causa do bureau. QA Perf: stress de
arrival-rate encontra o knee point da API. Backend: mas o knee pode ser o bureau, não a
Veridia — é preciso **injetar latência/5xx no stub do bureau** para testar o modo de falha
real, senão medimos um bureau mock saudável.

**Resolução:** dois testes complementares — (a) `sync-api-load-stress` sobe a taxa com motores
"saudáveis" para achar a capacidade; (b) `bureau-instability-spike` mantém a taxa nominal e
**degrada o bureau** (latência > 2s e 5xx 10–30%) para medir: fração de fallback
`APPROVED_WITH_REVIEW`, manutenção de p95 < 1.5s (o timeout de 2s corta a espera) e, crucial,
que o 5xx do bureau **não** vire 5xx da API (deve virar fallback). Como não há retry, cada
falha do bureau tem efeito direto — isso é medido como a curva "falha do bureau × fallback".

### D4 — T2 (pico Kafka): é volume com rampa de spike, não soak

Debate: QA Planner chamou de "volume"; QA Perf notou a rampa abrupta (50→830 msg/s) como
spike; Backend classificou pela carga acumulada e drenagem.

**Resolução:** modelar como **volume** (massa total + tempo de drenagem) com **rampa inicial de
spike** de arrival-rate via `ramping-arrival-rate` sobre xk6-kafka. Medir lag por partição
(12) e tempo de drenagem após o fim da injeção. Não é soak (duração insuficiente).

### D5 — T3 (pool Oracle): teste concorrente é obrigatório, não opcional

Esta foi a divergência mais forte. QA Perf, por isolamento de medição, preferia testes
separados por caminho (API e Kafka). Backend foi incisivo: **testar isolado produz um número
falso** porque o pool de 20 conexões/pod é compartilhado e só satura quando API e consumer
competem no pico. QA Planner apoiou o Backend: o risco de negócio (API degradando à meia-noite)
só se reproduz concorrente.

**Resolução (Orchestrator):** manter os testes isolados como **linha de base** (para ter o p95
"limpo") e adicionar o teste `shared-pool-contention` como **cenário bloqueante**: um único run
K6 com dois `scenarios` — HTTP `constant-arrival-rate` 300 req/s + Kafka `ramping-arrival-rate`
até ~830 msg/s. O resultado-chave é **Δp95 = p95(concorrente) − p95(isolado)** atribuível à
disputa do pool, mais observabilidade do pool no lado do alvo (conexões ativas, fila de espera).
Medir só no K6 aqui seria medir a coisa errada.

### D6 — T4 (lag vago → threshold): propor número e marcar como pendente

QA Perf exigiu um número (restrição do papel: nada de meta vaga). Backend deu a âncora física:
se ~2M msgs chegam em 40 min e o consumer processa a X msg/s, o backlog e o tempo de drenagem
são deriváveis do teste. QA Planner lembrou que o número precisa de aval do dono do SLA.

**Resolução:** adotar como threshold de trabalho **lag ≤ 5 min durante o pico** e **backlog
drenado em ≤ 15 min após o fim da injeção** — mais rígido que o "15 min" vago original e
derivado da capacidade medida. Fica em **Open Questions** como pendente de homologação, pois
partiu de meta não contratualizada.

### D7 — T5 (massa): unicidade por padrão, repetição controlada só na idempotência

Consenso rápido, formalizado pelo Orchestrator: massa de volume usa `traceId` únicos
(`<runId>-<vu>-<iter>`) e pool amplo de `customerId`/documentos para exercer a escrita real;
um cenário separado de idempotência reusa `traceId` deliberadamente numa fração conhecida.
Reportar métricas de escrita por caminho para que a repetição não mascare o volume.

### D8 — Matriz consolidada de testes

| ID | Teste | Tipo | Alvo | Executor K6 | Perfil | Thresholds (ancorados no SLA) | Aceite | Tensão |
|----|-------|------|------|-------------|--------|-------------------------------|--------|--------|
| E2E-1 | Aprovação direta | E2E | API | `per-vu-iterations` (baixa conc.) | 1–5 VUs, motores OK | `http_req_duration p95<1500,p99<2500`; `http_req_failed rate<0.005` | veredito `APPROVED` via check de corpo | — |
| E2E-2 | Recusa por fraude | E2E | API | `per-vu-iterations` | fraud sinaliza risco | idem latência; `checks rate>0.99` | veredito `DECLINED` mesmo com score bom | — |
| E2E-3 | Fallback do bureau | E2E | API | `per-vu-iterations` | bureau stub timeout>2s | `http_req_duration p95<1500` | veredito `APPROVED_WITH_REVIEW` dentro do SLA | T1 |
| E2E-4 | Kafka async | E2E | Kafka | xk6-kafka produce/consume | 1 msg, traceId único | lag fora de pico `<10s` | `completed` com mesmo traceId | — |
| E2E-5 | Idempotência | E2E | Kafka+Oracle | xk6-kafka | mesmo traceId 2× | — | 1 resultado, auditoria count=1 | T5 |
| LD-1 | Load sustentado | load | API | `constant-arrival-rate` | 300 req/s, 10 min + warm-up | `p95<1500,p99<2500`; `http_req_failed rate<0.005` | SLA mantido no sustentado | — |
| LD-2 | Pico de produção | load | API | `ramping-arrival-rate` | →700 req/s, 5 min | mesmos thresholds de SLA | SLA no pico ou degradação registrada | — |
| ST-1 | Stress knee point | stress | API | `ramping-arrival-rate` | degraus 700→1100+ req/s | marca a taxa onde `p95≥1500` ou `5xx≥0.5%` | capacidade máx/pod e recurso saturado | T1,T3 |
| SP-1 | Spike instab. bureau | spike | API+bureau stub | `constant-arrival-rate` + injeção | 300 req/s, bureau lat>2s e 5xx 10–30% | `http_req_duration p95<1500`; **`http_req_failed rate<0.005`** (5xx do bureau não vira 5xx da API) | curva falha-bureau×fallback | T1 |
| VL-1 | Volume pico Kafka | volume(+spike) | Kafka+Oracle | `ramping-arrival-rate` (xk6-kafka) | 50→830 msg/s, 2M msgs | **lag pico ≤5min**, drenagem ≤15min; perdas=0 | consumer acompanha; integridade traceId | T2,T4 |
| CT-1 | Concorrência pool | load concorrente | API+Kafka+Oracle | 2 `scenarios` (HTTP+Kafka) | 300 req/s + →830 msg/s juntos | `Δp95` atribuível ao pool quantificado; API `p95<1500` sob concorrência (meta) | pool satura? degradação da API | T3 |
| SK-1 (opt.) | Soak | soak | Kafka+Oracle | `constant-arrival-rate` | ~300 msg/s por 2–4h | deriva de `p95`/lag < X% ao longo do tempo | sem degradação lenta (pool/GC) | T2 |

Trecho ilustrativo de thresholds (LD-1):
```js
thresholds: {
  http_req_duration: ['p(95)<1500', 'p(99)<2500'],
  http_req_failed:   ['rate<0.005'],
}
```

## Risks / Trade-offs

- **[Mock do bureau não reproduz a instabilidade real]** → stub com injeção configurável de
  latência (>2s) e 5xx (10–30%); varrer degraus para obter a curva falha×fallback (SP-1), não
  um único ponto.
- **[Teste isolado dá número otimista do pool (T3)]** → CT-1 concorrente é bloqueante; o número
  de referência é Δp95 vs. isolado, não o p95 isolado.
- **[Massa mal gerada mascara a escrita (T5)]** → `traceId` únicos por padrão; repetição só no
  cenário de idempotência; métricas de escrita reportadas por caminho.
- **[QA tem 4 pods, prod tem 8]** → não extrapolar linearmente; reportar resultados por pod e
  marcar a extrapolação QA→prod como premissa a validar.
- **[Seed Oracle de ~400M linhas indisponível em QA]** → idempotência e volume dependem de uma
  tabela realista; sem ela, o custo do SELECT de idempotência e da escrita concorrente é
  subestimado. Pré-condição explícita de VL-1/CT-1/E2E-5.
- **[K6 não enxerga saturação de pool/lag]** → exigir observabilidade do alvo (pool, lag por
  partição, latência por downstream, taxa de fallback); sem isso, o teste mede a coisa errada.
- **[Arrival-rate com downstream lento esgota VUs]** → dimensionar `preAllocatedVUs`/`maxVUs`
  com folga; monitorar `dropped_iterations` como sinal de que a carga não foi aplicada.

## Migration Plan

Não aplicável como migração de software (a entrega é um plano de testes). Sequência de
habilitação sugerida: (1) stubs dos 3 motores com injeção de latência/erro; (2) seed Oracle
realista; (3) rodar E2E (baixa conc.) para validar fluxos e massa; (4) LD-1/LD-2 isolados como
baseline; (5) ST-1 e SP-1; (6) VL-1; (7) CT-1 concorrente; (8) SK-1 opcional. Rollback = sem
efeito em produção (testes em QA).

## Open Questions

- **T4:** confirmar com o dono do SLA o threshold proposto (lag ≤ 5 min no pico, drenagem ≤ 15
  min). O "15 min aceitável" original não é contratual.
- **Escala QA→prod:** 4 vs 8 pods — como extrapolar capacidade e Δp95 do pool com confiança?
- **Retenção/partições Kafka:** 12 partições suportam ~830 msg/s com o consumer atual? Medir em
  VL-1; se o lag não fechar, é limite de partição ou de pool Oracle?
- **Disponibilidade da seed de 400M linhas em QA** e do stub de bureau com injeção — bloqueiam
  VL-1/CT-1/E2E-5 se ausentes.
- **Observabilidade do pool por pod** disponível no ambiente de teste? Sem ela, T3 fica
  parcialmente cego.
