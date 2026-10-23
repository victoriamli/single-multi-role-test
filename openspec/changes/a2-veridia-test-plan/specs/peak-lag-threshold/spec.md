## ADDED Requirements

### Requirement: Threshold numérico de lag no pico
O plano SHALL substituir a meta vaga de lag no pico ("aceitável até 15 min") por um threshold
numérico testável e justificado, derivado da capacidade de drenagem observada (T4).

#### Scenario: Definição do threshold de drenagem
- **WHEN** o teste de volume Kafka mede o tempo para drenar o backlog do pico após o fim da
  injeção
- **THEN** o plano SHALL fixar um limite explícito — proposto em **lag máximo ≤ 5 min durante
  o pico e backlog totalmente drenado em ≤ 15 min após o fim da injeção** — e reprovar se
  ultrapassado

#### Scenario: Pendência de homologação do número
- **WHEN** o threshold proposto (5 min / 15 min) é registrado
- **THEN** o plano SHALL marcá-lo como **pendente de validação com o dono do SLA**, por ter
  partido de uma meta originalmente vaga e não contratualizada
