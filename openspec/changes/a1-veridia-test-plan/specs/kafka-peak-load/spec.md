## ADDED Requirements

### Requirement: Consumo fora de pico

O plano SHALL validar o consumo do tópico `credit.assessment.requested` no volume normal
(~50 msg/s) usando `xk6-kafka` com `constant-arrival-rate`, confirmando o SLO de lag fora de
pico.

#### Scenario: 50 msg/s sustentados

- **WHEN** mensagens são produzidas a ~50 msg/s por ≥ 10 min
- **THEN** o lag de consumo do consumer group SHALL permanecer < 10s e todo `traceId`
  produzido SHALL ter resultado correspondente em `credit.assessment.completed`

### Requirement: Pico Kafka noturno (T2)

O plano SHALL reproduzir o pico diário de ~2 milhões de mensagens em ~40 min (~830 msg/s
sustentados) usando `constant-arrival-rate` ao longo de ~40 min, classificando-o como teste
de carga sustentada de alta taxa (não spike instantâneo) por ter duração longa e taxa estável.

#### Scenario: 830 msg/s por 40 min

- **WHEN** ~2M mensagens são produzidas a ~830 msg/s ao longo de ~40 min
- **THEN** o throughput de processamento SHALL acompanhar a produção sem crescimento
  monotônico de lag ao fim da janela e a taxa de erro de processamento SHALL ser < 0.5%

### Requirement: Threshold de lag no pico substituindo a meta vaga (T4)

O plano SHALL substituir a meta vaga "aceitável até 15 min" por um threshold testável de lag
máximo e de tempo de drenagem após o fim do pico, justificado pelo número de partições (12) e
pela taxa de pico.

#### Scenario: Lag no pico sob threshold quantificado

- **WHEN** o pico de ~830 msg/s está ativo
- **THEN** o lag de consumo SHALL permanecer ≤ 5 min (threshold proposto, mais estrito que a
  meta vaga de 15 min) e, após o término do pico, a fila SHALL drenar até lag < 10s em ≤ 15
  min

#### Scenario: Pendência de validação humana do threshold

- **WHEN** o threshold de 5 min de lag é aplicado
- **THEN** o plano SHALL marcar esse número como decisão a confirmar com os donos do SLA,
  por derivar de uma meta originalmente não quantificada

### Requirement: Spike de reconexão/rebalance

O plano SHALL avaliar o comportamento do consumer sob início abrupto do pico (de ~50 para
~830 msg/s), observando rebalance de partições e recuperação de lag.

#### Scenario: Transição abrupta de 50 para 830 msg/s

- **WHEN** a taxa de produção salta de ~50 para ~830 msg/s em ≤ 1 min
- **THEN** o consumer SHALL estabilizar o processamento sem perda de mensagens e o lag SHALL
  voltar a tendência decrescente dentro de 5 min após a estabilização
