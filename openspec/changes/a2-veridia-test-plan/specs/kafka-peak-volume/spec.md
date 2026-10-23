## ADDED Requirements

### Requirement: Reprodução do pico Kafka noturno
O plano SHALL definir um teste de volume que reproduza o pico 23h–00h (~2M mensagens em ~40
min, ~830 msg/s) no tópico `credit.assessment.requested` via xk6-kafka, medindo lag e
throughput de consumo (T2).

#### Scenario: Injeção do pico de arrival-rate
- **WHEN** o produtor K6 (`ramping-arrival-rate` sobre xk6-kafka) sobe de ~50 msg/s até ~830
  msg/s e sustenta o volume alvo
- **THEN** o teste SHALL medir o lag de consumo por partição e o tempo total de drenagem após
  o fim da injeção, registrando se o consumer acompanha a produção

### Requirement: Classificação do pico como volume com componente de spike
O plano SHALL justificar o pico como teste de **volume** (massa acumulada + drenagem) com
rampa inicial de **spike** de arrival-rate, e não como soak (duração insuficiente para soak).

#### Scenario: Justificativa de tipo de teste registrada
- **WHEN** o design classifica o teste do pico Kafka
- **THEN** SHALL documentar por que volume+spike se aplicam e por que soak é apenas parcial
  (40 min não caracteriza soak de horas)

### Requirement: Correlação de resultado no pico
O plano SHALL verificar que cada `traceId` produzido no pico gera exatamente um resultado em
`credit.assessment.completed`, mesmo sob lag elevado.

#### Scenario: Integridade sob volume
- **WHEN** o volume alvo é injetado com `traceId` únicos
- **THEN** a contagem de resultados em `credit.assessment.completed` SHALL igualar a de
  pedidos únicos, sem perdas nem duplicatas, após a drenagem completa
