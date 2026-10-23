## ADDED Requirements

### Requirement: Carga sustentada da API no alvo de produção
O plano SHALL especificar um teste de carga da API `POST /v1/assessments` em modelo aberto
(arrival-rate), sustentando ~300 req/s, com thresholds ancorados no SLA.

#### Scenario: Carga sustentada a 300 req/s
- **WHEN** a API recebe ~300 req/s sustentados por pelo menos 15 min via
  `ramping-arrival-rate` (com warm-up)
- **THEN** `http_req_duration` mantém p95 < 1500ms e p99 < 2500ms e `http_req_failed`
  mantém taxa < 0.5%

### Requirement: Pico de carga da API
O plano SHALL especificar um teste de spike levando a API de ~300 para ~700 req/s em ~30s,
para avaliar o comportamento sob o pico declarado.

#### Scenario: Spike de 300 para 700 req/s
- **WHEN** a taxa de chegada sobe de 300 para 700 req/s em ~30s e se mantém no pico
- **THEN** as métricas de latência e erro são registradas contra o SLA e qualquer degradação
  (p95 > 1500ms ou 5xx > 0.5%) é reportada explicitamente como decisão de aceite do pico

### Requirement: Stress até o ponto de ruptura
O plano SHALL especificar um teste de stress que eleva a carga acima de 700 req/s até
identificar o joelho (knee point) de latência/erro, com threshold afrouxado para localizar a
quebra, não para passar.

#### Scenario: Busca do knee point
- **WHEN** a taxa de chegada cresce progressivamente além de 700 req/s
- **THEN** o teste registra a taxa na qual p95 e `http_req_failed` degradam de forma
  não-linear e identifica o recurso que satura primeiro (pool Oracle, Virtual Threads ou
  downstream)
