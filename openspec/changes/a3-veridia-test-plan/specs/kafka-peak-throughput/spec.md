## ADDED Requirements

### Requirement: Pico Kafka noturno como arrival-rate sustentado (T2)
O plano SHALL especificar um teste do pico noturno modelando ~833 msg/s por ~40 min em
`credit.assessment.requested` via `xk6-kafka`, caracterizado como spike + volume + soak no
mesmo evento.

#### Scenario: 2M mensagens em ~40 min
- **WHEN** ~833 msg/s são produzidas sustentadamente por ~40 min (≈2M mensagens) em
  `credit.assessment.requested`
- **THEN** o consumer processa o fluxo e publica em `credit.assessment.completed`, com lag,
  backlog e tempo de drenagem medidos por partição

### Requirement: Metas testáveis de lag no pico (T4)
O plano SHALL substituir a meta vaga "aceitável até 15 min" por thresholds numéricos de lag
operacional, backlog máximo e tempo de drenagem.

#### Scenario: Lag e backlog durante o pico
- **WHEN** o pico está ativo
- **THEN** o lag operacional de consumo permanece < 180s, o backlog permanece < 750.000
  mensagens e o tempo de drenagem após o fim do pico é < 900s (15 min, como hard cap)

#### Scenario: Lag fora do pico
- **WHEN** o volume é o de baseline (~50 msg/s)
- **THEN** o lag de consumo permanece < 10s, conforme SLO declarado

### Requirement: Capacidade de consumo é pré-condição explícita
O plano SHALL registrar a capacidade máxima de consumo (`C_max`, msg/s) como variável
bloqueante para a validade das metas de backlog e drenagem.

#### Scenario: C_max desconhecido
- **WHEN** `C_max` ainda não foi medido
- **THEN** as metas de backlog/drenagem são tratadas como provisórias e marcadas como
  pendência de decisão humana
