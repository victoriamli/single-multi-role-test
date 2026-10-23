## ADDED Requirements

### Requirement: Aprovação direta ponta-a-ponta

O plano SHALL validar, via E2E declarativo (Gherkin/HTTP), que um cliente elegível cujos três
motores (`engine-score`, `engine-fraud`, `engine-bureau`) retornam OK resulta em veredito
`APPROVED` dentro do SLA síncrono (p95 < 1.5s, p99 < 2.5s), com registro de auditoria gravado
no Oracle.

#### Scenario: Cliente elegível com os três motores OK

- **WHEN** um `POST /v1/assessments` é enviado para um cliente elegível e os três motores
  respondem OK
- **THEN** a resposta SHALL ser `APPROVED` na mesma requisição, com latência individual
  dentro de p95 < 1.5s, e um registro de auditoria correspondente ao `traceId` SHALL existir
  no Oracle

### Requirement: Recusa por fraude

O plano SHALL validar que, quando `engine-fraud` sinaliza risco, o veredito é `DECLINED`
mesmo com `engine-score` bom, confirmando a precedência da regra de fraude.

#### Scenario: Fraude sinalizada com score bom

- **WHEN** `engine-fraud` retorna risco para um pedido cujo `engine-score` é favorável
- **THEN** o veredito SHALL ser `DECLINED` e o motivo registrado na auditoria SHALL indicar
  fraude

### Requirement: Fallback na degradação do bureau

O plano SHALL validar que, quando `engine-bureau` estoura o timeout de 2s (sem retry), a regra
de negócio responde `APPROVED_WITH_REVIEW` e que o fallback ocorre dentro do SLA do endpoint
(p95 < 1.5s), já que o bureau não pode fazer a requisição exceder o contrato (T1).

#### Scenario: Timeout do bureau dispara fallback dentro do SLA

- **WHEN** `engine-bureau` não responde dentro de 2s enquanto `engine-score` e `engine-fraud`
  retornam OK
- **THEN** o veredito SHALL ser `APPROVED_WITH_REVIEW` e a latência total da requisição SHALL
  permanecer < 1.5s no p95

#### Scenario: Bureau retorna 5xx

- **WHEN** `engine-bureau` responde 5xx antes do timeout
- **THEN** o veredito SHALL ser `APPROVED_WITH_REVIEW` e a resposta NÃO SHALL propagar erro
  5xx ao chamador

### Requirement: Caminho assíncrono via Kafka

O plano SHALL validar que um pedido publicado em `credit.assessment.requested` gera um
resultado em `credit.assessment.completed` preservando o mesmo `traceId`, usando `xk6-kafka`
para produzir e consumir.

#### Scenario: Pedido assíncrono processado com traceId preservado

- **WHEN** uma mensagem com `traceId` T é produzida em `credit.assessment.requested`
- **THEN** uma mensagem de resultado com o mesmo `traceId` T SHALL aparecer em
  `credit.assessment.completed` dentro do limite de lag fora de pico (< 10s)

### Requirement: Idempotência por traceId

O plano SHALL validar que reprocessar o mesmo `traceId` não gera registro de auditoria
duplicado, confirmando a verificação de idempotência contra o Oracle.

#### Scenario: Reprocessamento do mesmo traceId

- **WHEN** a mesma mensagem com `traceId` T é entregue duas vezes ao consumer
- **THEN** o Oracle SHALL conter exatamente um registro de auditoria para o `traceId` T e um
  único resultado SHALL ser publicado em `credit.assessment.completed`
