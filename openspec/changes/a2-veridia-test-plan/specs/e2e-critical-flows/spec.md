## ADDED Requirements

### Requirement: Fluxo de aprovação direta
O plano SHALL incluir um cenário E2E que valide o fluxo de aprovação direta: cliente elegível,
os 3 motores (`engine-score`, `engine-fraud`, `engine-bureau`) retornam OK e o veredito
consolidado é `APPROVED` dentro do SLA síncrono.

#### Scenario: Cliente elegível aprovado
- **WHEN** um `POST /v1/assessments` é enviado para um cliente elegível e os 3 motores
  respondem OK
- **THEN** a resposta SHALL ser `APPROVED` na mesma requisição, com `http_req_duration`
  p95 < 1.5s e p99 < 2.5s e `http_req_failed` rate < 0.5%

### Requirement: Fluxo de recusa por fraude
O plano SHALL incluir um cenário E2E em que `engine-fraud` sinaliza risco e o veredito é
`DECLINED` mesmo com score bom, verificando que a regra de precedência de fraude é aplicada.

#### Scenario: Fraude tem precedência sobre score bom
- **WHEN** `engine-score` retorna score aprovável mas `engine-fraud` sinaliza risco
- **THEN** o veredito SHALL ser `DECLINED` e o resultado SHALL ser verificado por um `check`
  de corpo, não apenas por status HTTP 200

### Requirement: Fallback na degradação do bureau
O plano SHALL incluir um cenário E2E em que `engine-bureau` dá timeout (>2s) e a regra de
negócio responde `APPROVED_WITH_REVIEW`, validando que o fallback dispara dentro do SLA.

#### Scenario: Timeout do bureau aciona fallback
- **WHEN** o stub de `engine-bureau` é configurado para exceder o timeout de 2s
- **THEN** o veredito SHALL ser `APPROVED_WITH_REVIEW` e a latência total SHALL permanecer
  p95 < 1.5s (o fallback não espera além do timeout por downstream)

### Requirement: Caminho assíncrono Kafka ponta a ponta
O plano SHALL incluir um cenário E2E que produza em `credit.assessment.requested` e verifique
a publicação correlata em `credit.assessment.completed` com o mesmo `traceId`.

#### Scenario: Pedido assíncrono produz resultado correlacionado
- **WHEN** uma mensagem com `traceId` único é produzida em `credit.assessment.requested` via
  xk6-kafka
- **THEN** uma mensagem correspondente SHALL aparecer em `credit.assessment.completed` com o
  mesmo `traceId`, dentro do limite de lag fora de pico (< 10s), validado por `check`

### Requirement: Idempotência do consumer
O plano SHALL incluir um cenário E2E que reenvie o mesmo `traceId` e verifique que não há
auditoria duplicada no Oracle.

#### Scenario: Reprocessamento do mesmo traceId não duplica auditoria
- **WHEN** a mesma mensagem (mesmo `traceId`) é produzida duas vezes
- **THEN** o resultado SHALL ser publicado uma única vez e a contagem de registros de auditoria
  para aquele `traceId` SHALL permanecer igual a 1
