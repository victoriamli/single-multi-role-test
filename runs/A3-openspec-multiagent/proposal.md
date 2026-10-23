## Why

Veridia é um gateway de decisão de crédito com um SLA apertado (p95 < 1.5s) que depende de
um downstream lento e instável (`engine-bureau`, p95 ~1.2s), e com um pico Kafka noturno
(~2M msgs em ~40 min) que disputa o mesmo pool Oracle (20 conexões/pod) com a API síncrona.
Hoje não existe um plano de testes que exercite esses pontos de ruptura de forma ancorada
nos SLAs — teste isolado por caminho dá falso verde e esconde o acoplamento real. Precisamos
de um plano K6 que transforme as tensões T1–T5 em testes com critérios de aceite numéricos.

## What Changes

- Novo plano de testes K6 para Veridia cobrindo **E2E, load, spike, stress, volume e soak**,
  com inclusões e exclusões justificadas por fluxo/tensão do alvo.
- Cobertura dos **5 fluxos críticos** E2E (aprovação direta, recusa por fraude, fallback do
  bureau, caminho Kafka assíncrono, idempotência) com asserções de veredito, não só de
  latência.
- Modelagem de carga **open-model** (arrival-rate) para a API, alinhada a ~300 req/s
  sustentado e ~700 req/s de pico, evitando que VUs presos mascarem o SLA (T1).
- Cenário obrigatório de **sobreposição API + consumer Kafka** para expor a contenção do
  pool Oracle (T3) — medição diferencial de Δp95 da API com e sem carga Kafka concorrente.
- Modelagem do **pico Kafka** como arrival-rate sustentado de ~833 msg/s por 40 min
  (spike + volume + soak no mesmo evento) (T2).
- **Transformação da meta vaga de lag** ("aceitável até 15 min") em thresholds testáveis:
  lag operacional, backlog máximo em mensagens e tempo de drenagem (T4).
- **Estratégia de massa** com `traceId`/`customerId` majoritariamente únicos e fração
  controlada de reuso para exercitar idempotência sem mascarar a carga de escrita (T5).
- Thresholds mensuráveis por teste, ancorados nos SLAs (p95<1.5s, p99<2.5s, 5xx<0.5%).

Escopo é **planejamento**, não implementação de scripts: trechos curtos de
`options`/`thresholds` ilustram decisões, sem scripts completos.

## Capabilities

### New Capabilities
- `e2e-critical-flows`: validação ponta a ponta dos 5 fluxos de negócio (aprovação, fraude,
  fallback do bureau, Kafka assíncrono, idempotência), com aceite por veredito.
- `sync-api-load-stress`: carga e stress da API síncrona `POST /v1/assessments` alinhados a
  ~300/~700 req/s, cobrindo T1 (SLA vs bureau) e a busca do ponto de ruptura.
- `bureau-degradation-resilience`: comportamento sob instabilidade injetada do
  `engine-bureau` (timeout/5xx) e o fallback `APPROVED_WITH_REVIEW` dentro do SLA (T1).
- `kafka-peak-throughput`: pico noturno (~833 msg/s por 40 min) e sua caracterização como
  spike+volume+soak, com metas de lag e drenagem (T2, T4).
- `oracle-pool-contention`: sobreposição API + consumer Kafka para expor a disputa do pool
  Oracle (20/pod) e medir o impacto diferencial no SLA da API (T3).
- `mass-data-strategy`: geração de massa com unicidade controlada de `traceId`/`customerId`
  e fração de reuso proposital para idempotência (T5).

### Modified Capabilities
<!-- Nenhuma: não há specs existentes cujos requisitos mudem. Todas são novas capabilities de teste. -->

## Impact

- **Alvo sob teste:** Veridia (API síncrona, consumer Kafka, Oracle). Sem mudança de
  arquitetura — o plano testa o alvo como descrito.
- **Ferramenta:** K6 (JavaScript) com executors de arrival-rate e `xk6-kafka` para o caminho
  Kafka; métricas customizadas e checks para vereditos e lag.
- **Ambiente:** QA com 4 pods (80 conexões Oracle) vs prod com 8 pods (160) — extrapolação
  não-linear a sinalizar. Depende de mock do `engine-bureau` com injeção de latência/erro e
  de seed realista da tabela de auditoria (~400M linhas) excluindo os `traceId` de teste.
- **Pendências para decisão humana:** quantificação final do lag no pico (T4), capacidade
  máxima de consumo do consumer (`C_max`), SLA durante o pico de 700 req/s e regra de
  veredito com múltiplos downstream degradados.
