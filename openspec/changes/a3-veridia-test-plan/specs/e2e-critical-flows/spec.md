## ADDED Requirements

### Requirement: Fluxo de aprovação direta (F3)
O plano SHALL validar E2E que um pedido elegível, com os 3 motores retornando OK, produz
veredito `APPROVED` dentro do SLA da API.

#### Scenario: Três motores OK
- **WHEN** um `POST /v1/assessments` é enviado com cliente elegível e os 3 motores
  (`engine-score`, `engine-fraud`, `engine-bureau`) respondem OK
- **THEN** o veredito consolidado é `APPROVED` e a resposta ocorre com p95 < 1.5s e
  p99 < 2.5s

### Requirement: Fluxo de recusa por fraude (F1)
O plano SHALL validar E2E que um sinal de risco do `engine-fraud` produz veredito `DECLINED`
mesmo quando o score é bom, por representar risco de decisão incorreta.

#### Scenario: Fraude sinalizada com score bom
- **WHEN** o `engine-fraud` sinaliza risco e o `engine-score` retorna score aprovável
- **THEN** o veredito consolidado é `DECLINED` e a decisão é registrada na auditoria

### Requirement: Fluxo de degradação do bureau com fallback (F2)
O plano SHALL validar E2E que, quando o `engine-bureau` estoura o timeout de 2s, a regra de
negócio responde `APPROVED_WITH_REVIEW` dentro do SLA.

#### Scenario: Timeout do bureau dispara fallback
- **WHEN** o `engine-bureau` não responde dentro do timeout de 2s por downstream
- **THEN** o veredito é `APPROVED_WITH_REVIEW` e a resposta ainda respeita p95 < 1.5s e
  p99 < 2.5s

#### Scenario: Bureau retorna 5xx
- **WHEN** o `engine-bureau` retorna erro 5xx (sem retry configurado)
- **THEN** o fallback `APPROVED_WITH_REVIEW` é aplicado e o erro do downstream NÃO é
  propagado como 5xx da API

### Requirement: Fluxo assíncrono via Kafka (F5)
O plano SHALL validar E2E que um pedido publicado em `credit.assessment.requested` gera um
resultado em `credit.assessment.completed` com o mesmo `traceId`.

#### Scenario: Requested gera Completed com mesmo traceId
- **WHEN** uma mensagem é produzida em `credit.assessment.requested` com um `traceId` único
- **THEN** uma mensagem correspondente aparece em `credit.assessment.completed` com o mesmo
  `traceId` e a taxa de correspondência (`completed/requested`) é 100% (`rate == 1`)

### Requirement: Idempotência de reprocessamento (F4)
O plano SHALL validar E2E que o reprocessamento de um mesmo `traceId` não gera registro de
auditoria duplicado.

#### Scenario: Mesmo traceId reprocessado
- **WHEN** uma mensagem com um `traceId` já processado é reenviada ao consumer
- **THEN** nenhum novo registro de auditoria é criado para aquele `traceId` (contagem de
  auditoria permanece 1)

#### Scenario: Reprocessamento concorrente do mesmo traceId
- **WHEN** duas mensagens com o mesmo `traceId` são consumidas concorrentemente (partições
  ou threads distintas)
- **THEN** no máximo um registro de auditoria é persistido, sem violação de unicidade não
  tratada
