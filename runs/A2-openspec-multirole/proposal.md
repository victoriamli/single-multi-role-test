## Why

Veridia tem SLA contratual apertado (p95 < 1.5s) sobre um caminho síncrono cujo downstream
mais lento e instável (`engine-bureau`, p95 ~1.2s, picos de timeout/5xx) sozinho quase estoura
a meta, e um pico Kafka noturno (~2M msgs em ~40 min) que disputa o mesmo pool Oracle (20
conexões/pod) com a API. Hoje não há um plano de testes K6 que valide os fluxos críticos nem
que exponha esses pontos de ruptura antes da produção. Este change define esse plano — a
entrega é o **planejamento**, não a implementação dos scripts.

## What Changes

- Plano de testes K6 cobrindo os **5 fluxos de negócio E2E** (aprovação direta, recusa por
  fraude, fallback do bureau, caminho assíncrono Kafka, idempotência) com critérios de aceite
  mensuráveis.
- Estratégia de **load** na API `POST /v1/assessments` ancorada em ~300 req/s sustentado e
  ~700 req/s de pico, com thresholds derivados do SLA (p95<1.5s, p99<2.5s, 5xx<0.5%).
- Estratégia de **stress** até o ponto de ruptura em torno de T1 (bureau lento/instável e o
  timeout de 2s sem retry), buscando o *knee point* da degradação.
- Estratégia de **spike** reproduzindo instabilidade do bureau (injeção de latência/5xx) e a
  rampa do pico Kafka.
- Estratégia de **volume** no consumer Kafka reproduzindo o pico 23h–00h (~830 msg/s) sobre
  Oracle com ~400M linhas, incluindo o **teste concorrente API+Kafka** (T3) que expõe a
  disputa do pool.
- Definição de um **threshold testável para o lag no pico** (T4), substituindo a meta vaga de
  "aceitável até 15 min".
- Estratégia de **dados de massa** que garante unicidade de `traceId`/`customerId` para não
  mascarar a carga real de escrita nem ativar indevidamente a idempotência (T5).

## Capabilities

### New Capabilities
- `e2e-critical-flows`: cenários E2E de ponta a ponta para os 5 fluxos de negócio, incl.
  fallback do bureau e idempotência, agnósticos de protocolo (HTTP + Kafka).
- `sync-api-load-stress`: load e stress da API síncrona ancorados em throughput de produção e
  no SLA; busca do ponto de ruptura (T1).
- `bureau-instability-spike`: spike/resiliência em torno da instabilidade do `engine-bureau`
  (timeout 2s sem retry) e seu efeito no SLA e no fallback (T1).
- `kafka-peak-volume`: volume/soak do consumer reproduzindo o pico noturno e medindo lag e
  throughput de consumo (T2).
- `shared-pool-contention`: teste concorrente API + consumer Kafka que expõe a disputa do pool
  Oracle no pico (T3).
- `peak-lag-threshold`: definição e validação de um threshold numérico de lag no pico,
  substituindo a meta vaga (T4).
- `mass-data-strategy`: estratégia de geração de massa única e realista, medindo os dois
  caminhos de escrita sem mascaramento por idempotência (T5).

### Modified Capabilities
<!-- Nenhuma: openspec/specs/ está vazio; não há requisitos existentes sendo alterados. -->

## Impact

- **Alvo sob teste:** API `POST /v1/assessments`, consumer `credit.assessment.requested` /
  producer `credit.assessment.completed`, persistência Oracle (auditoria + idempotência).
- **Ferramenta:** K6 (executors `ramping-vus`, `constant-arrival-rate`,
  `ramping-arrival-rate`), extensão `xk6-kafka`; scripts versionados em pasta interna do
  projeto de QA.
- **Dependências de ambiente:** stubs/mocks dos 3 motores com injeção de latência/erro; seed
  Oracle próximo de produção (~400M linhas) para idempotência/volume; ambiente QA de 4 pods
  (prod = 8) a ser considerado na extrapolação.
- **Observabilidade necessária no alvo:** lag por partição, uso do pool por pod, latência por
  downstream, taxa de fallback do bureau — medições que o K6 sozinho não enxerga.
