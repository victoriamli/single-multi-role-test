## ADDED Requirements

### Requirement: Unicidade de massa por padrão (T5)
O plano SHALL gerar `traceId` e `customerId` majoritariamente únicos para não ativar o
caminho de idempotência e não mascarar a carga real de escrita no Oracle.

#### Scenario: Geração de massa única
- **WHEN** os cenários de carga/volume geram requisições
- **THEN** cada requisição usa um `traceId` único (ex.: `uuidv4`) e um `customerId` variado
  (ex.: `cust-<VU>-<ITER>-<timestamp>`), de forma que ≥ 95% das chamadas sejam de identidades
  novas

### Requirement: Fração controlada de reuso para idempotência
O plano SHALL incluir uma fração deliberada e pequena de `traceId` repetidos para exercitar o
caminho de idempotência sem contaminar a medição de escrita.

#### Scenario: Reuso proposital medido à parte
- **WHEN** ~5% das mensagens reutilizam `traceId` já enviados
- **THEN** o caminho de idempotência é exercitado e suas métricas (SELECT, deduplicação) são
  medidas separadamente das métricas de escrita nova

### Requirement: Seed da tabela de auditoria
O plano SHALL assumir um seed realista da tabela de auditoria (~400M linhas) para que o custo
do `SELECT` de idempotência seja representativo, excluindo os `traceId` usados nos testes.

#### Scenario: Seed não colide com a massa de teste
- **WHEN** a tabela é pré-carregada com ~400M linhas
- **THEN** os `traceId` de teste não existem previamente na tabela, evitando falso positivo de
  idempotência
