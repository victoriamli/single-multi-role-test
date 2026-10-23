## Why

Veridia é um gateway de decisão de crédito com um SLA síncrono apertado (p95 < 1.5s)
que depende de um downstream lento e instável (`engine-bureau`, p95 ~1.2s), e com um
pico Kafka noturno de ~2M mensagens em ~40 min que disputa o mesmo pool Oracle (20
conexões/pod) usado pela API síncrona. Hoje não existe um plano de testes K6 que exerça
esses pontos de ruptura de forma mensurável; sem ele, não há como afirmar que a aplicação
sustenta a carga real nem onde ela quebra. Este change define esse plano de testes
(E2E, carga, volume, stress) ancorado nos fluxos, SLAs e tensões reais do alvo.

## What Changes

- Introduz um **plano de testes K6** (planejamento, não implementação de scripts) cobrindo
  os 5 fluxos de negócio críticos e as 5 tensões (T1–T5) do alvo.
- Define cobertura **E2E** declarativa (Gherkin, agnóstica de protocolo) para aprovação
  direta, recusa por fraude, fallback do bureau (`APPROVED_WITH_REVIEW`), caminho assíncrono
  Kafka e idempotência.
- Define testes de **carga** da API `POST /v1/assessments` no patamar de produção (~300
  req/s) com thresholds derivados do SLA (p95 < 1.5s, p99 < 2.5s, erro 5xx < 0.5%),
  incluindo um perfil com `engine-bureau` instável (T1).
- Define testes de **stress** (acima da capacidade, até ruptura) e **spike** sobre a API
  e sobre o consumer Kafka para localizar o ponto de saturação (T1, T2).
- Define o teste do **pico Kafka** (~830 msg/s por ~40 min) classificando-o e transformando
  a meta vaga de lag ("aceitável até 15 min") em um threshold testável (T2, T4).
- Define teste de **volume/contention** que exercita a tabela de auditoria de ~400M linhas
  e a disputa do pool Oracle entre API síncrona e consumer sob pico concorrente (T3).
- Define a **estratégia de massa de dados** (traceId/customerId únicos) para não mascarar a
  carga real de escrita via idempotência (T5).
- Define **thresholds mensuráveis** e critérios de aceite por teste; o detalhamento da
  modelagem de carga vive no `design.md`.

## Capabilities

### New Capabilities

- `e2e-decision-flows`: cobertura ponta-a-ponta dos 5 fluxos de negócio críticos (aprovação
  direta, recusa por fraude, fallback do bureau no timeout, caminho assíncrono Kafka,
  idempotência), com critérios de aceite por fluxo.
- `sync-api-load-stress`: carga, stress e spike sobre `POST /v1/assessments`, com thresholds
  derivados do SLA e um perfil com `engine-bureau` instável (T1).
- `kafka-peak-load`: modelagem e execução do pico noturno (~2M msgs/40 min) e do throughput
  fora de pico, com um threshold de lag quantificado em substituição à meta vaga (T2, T4).
- `oracle-contention-volume`: volume sobre a tabela de auditoria (~400M linhas) e contenção
  do pool Oracle compartilhado (API + consumer) sob pico concorrente, com massa que não
  dispara idempotência (T3, T5).

### Modified Capabilities

<!-- Nenhuma. Não há specs existentes em openspec/specs/ cujos requisitos mudem. -->

## Impact

- **Alvo sob teste:** Veridia — API `POST /v1/assessments`, consumer Kafka
  `credit.assessment.requested`/`credit.assessment.completed`, persistência Oracle.
- **Downstreams exercitados indiretamente:** `engine-score`, `engine-fraud`, `engine-bureau`
  (inclusive seu comportamento instável) e o bureau externo de terceiros.
- **Ferramenta de execução:** K6 (JS), executors `constant-arrival-rate`,
  `ramping-arrival-rate`, `ramping-vus`; extensão `xk6-kafka` para o caminho assíncrono;
  motor E2E declarativo (Gherkin) da aplicação de QA.
- **Ambientes:** QA (4 pods) como ambiente primário de execução; extrapolação para produção
  (8 pods) documentada como premissa, não testada diretamente.
- **Dados:** geração de massa com milhões de `traceId`/`customerId` únicos; dependência de
  ambiente Oracle com volume de auditoria representativo.
- **Sem mudança de arquitetura do alvo:** o plano testa Veridia como descrito.
