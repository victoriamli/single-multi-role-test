## ADDED Requirements

### Requirement: Unicidade de massa para não mascarar a carga de escrita
O plano SHALL definir uma estratégia de geração de massa que garanta `traceId` únicos e
`customerId`/documentos variados, para que o teste de volume exerça a escrita real de auditoria
e não ative indevidamente o caminho de idempotência (T5).

#### Scenario: Geração de massa única em escala
- **WHEN** o teste de volume/concorrência precisa de milhões de requisições
- **THEN** cada iteração SHALL usar um `traceId` único (ex.: prefixo de execução + VU +
  contador) e `customerId`/documento de um pool amplo, garantindo que a escrita de auditoria
  ocorra de fato

### Requirement: Fração controlada de repetição para exercitar idempotência
O plano SHALL incluir uma fração conhecida e controlada de `traceId` repetidos apenas no
cenário de idempotência, separada da carga de volume, para medir os dois caminhos sem
contaminar um ao outro.

#### Scenario: Caminhos de escrita e idempotência medidos separadamente
- **WHEN** o cenário de idempotência reusa `traceId` deliberadamente enquanto o de volume usa
  apenas únicos
- **THEN** as métricas de latência/throughput de escrita SHALL ser reportadas por caminho, de
  modo que a repetição não mascare a carga real de escrita do volume
