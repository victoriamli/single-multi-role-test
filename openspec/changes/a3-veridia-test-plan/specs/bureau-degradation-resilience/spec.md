## ADDED Requirements

### Requirement: Instabilidade do bureau injetada sob carga (T1)
O plano SHALL especificar um teste que injeta latência e erros (timeout/5xx) no
`engine-bureau` enquanto a API recebe carga, para medir o efeito no SLA e na taxa de
fallback. O mock do bureau SHALL ser configurável (latência/erro injetados), não fixo.

#### Scenario: Bureau no limite do p95 sob carga nominal
- **WHEN** o `engine-bureau` responde com p95 ~1.2s e a API recebe ~300 req/s
- **THEN** a API mantém p95 < 1.5s e p99 < 2.5s, com a folga de SLA medida e reportada

#### Scenario: Picos de timeout do bureau elevam a taxa de fallback
- **WHEN** uma fração configurável das chamadas ao `engine-bureau` estoura o timeout de 2s
- **THEN** a `taxa de fallback` (APPROVED_WITH_REVIEW) é medida e a API mantém `http_req_failed`
  < 0.5% (o timeout do downstream não vira 5xx da API)

### Requirement: Efeito em cascata sem retry
O plano SHALL verificar que, sem retry automático, a latência do bureau não empilha além do
timeout de 2s por chamada e não degrada p99 além do SLA.

#### Scenario: Ausência de retry limita o custo por requisição
- **WHEN** o `engine-bureau` falha e não há retry configurado
- **THEN** o custo adicional por requisição é limitado ao timeout de 2s e o p99 da API
  permanece < 2.5s
