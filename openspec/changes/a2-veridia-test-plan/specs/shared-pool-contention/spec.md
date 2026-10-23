## ADDED Requirements

### Requirement: Teste concorrente API + consumer Kafka
O plano SHALL definir um teste que execute simultaneamente a carga da API síncrona (~300 req/s)
e o pico Kafka (~830 msg/s), para expor a disputa do pool Oracle compartilhado (20 conexões por
pod) que testes isolados por caminho escondem (T3).

#### Scenario: Carga concorrente sobre o pool compartilhado
- **WHEN** dois cenários K6 rodam no mesmo teste — um `constant-arrival-rate` HTTP a 300 req/s
  e um `ramping-arrival-rate` Kafka até ~830 msg/s — contra o mesmo ambiente
- **THEN** o teste SHALL comparar o p95 da API sob concorrência com o p95 da API isolada, e a
  degradação atribuível à disputa do pool SHALL ser quantificada (ex.: aumento de p95 e de
  tempo de espera por conexão)

### Requirement: Observabilidade do pool como critério de medição
O plano SHALL exigir medição do uso do pool Oracle por pod durante o teste concorrente, já que
o K6 sozinho não enxerga saturação de pool.

#### Scenario: Saturação de pool detectada
- **WHEN** o teste concorrente roda com coleta de métricas do pool (conexões ativas, fila de
  espera) no lado do alvo
- **THEN** o resultado SHALL indicar se o pool saturou (fila de espera > 0 sustentada) e
  correlacionar isso com a degradação de latência observada na API
