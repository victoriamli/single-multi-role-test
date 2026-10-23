## ADDED Requirements

### Requirement: Load da API no throughput de produção
O plano SHALL definir um teste de carga da API `POST /v1/assessments` que sustente ~300 req/s
e valide o pico de ~700 req/s, usando um executor orientado a taxa de chegada para desacoplar a
carga da latência do downstream.

#### Scenario: Carga sustentada dentro do SLA
- **WHEN** o teste aplica `constant-arrival-rate` de 300 req/s por pelo menos 10 min após
  warm-up, com mix realista de vereditos
- **THEN** SHALL atender `http_req_duration: p(95)<1500, p(99)<2500` e
  `http_req_failed: rate<0.005`; violação de qualquer threshold reprova o teste

#### Scenario: Pico de produção
- **WHEN** a taxa sobe para ~700 req/s por uma janela curta (ex.: 5 min) via
  `ramping-arrival-rate`
- **THEN** os mesmos thresholds de SLA SHALL ser avaliados e qualquer degradação acima do SLA
  SHALL ser registrada como resultado

### Requirement: Stress até o ponto de ruptura
O plano SHALL definir um teste de stress que aumente a taxa de chegada além de 700 req/s até
identificar o *knee point* (onde p95 cruza 1.5s ou 5xx cruza 0.5%), reportando a capacidade
máxima por pod.

#### Scenario: Busca do knee point
- **WHEN** `ramping-arrival-rate` sobe em degraus (ex.: 700→900→1100→… req/s) com
  `preAllocatedVUs`/`maxVUs` dimensionados para não limitar artificialmente a carga
- **THEN** o teste SHALL registrar a taxa em que p95 ultrapassa 1.5s ou 5xx ultrapassa 0.5% e
  identificar qual recurso saturou primeiro (CPU, pool Oracle ou downstream)
