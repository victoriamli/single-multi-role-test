## ADDED Requirements

### Requirement: Contenção do pool Oracle compartilhado (T3)

O plano SHALL executar simultaneamente a carga da API síncrona (~300 req/s) e o pico Kafka
(~830 msg/s) para expor a disputa pelo pool de conexões Oracle compartilhado (20 por pod), já
que testar cada caminho isoladamente esconde o acoplamento.

#### Scenario: API e consumer competindo pelo pool no pico

- **WHEN** a carga da API (~300 req/s) e o pico Kafka (~830 msg/s) rodam concorrentemente
  contra o mesmo cluster
- **THEN** o p95 de `http_req_duration` da API SHALL ser medido e comparado ao baseline
  isolado, e a degradação atribuível à espera por conexão (tempo de aquisição no pool) SHALL
  ser reportada; o teste SHALL falhar se o p95 da API ultrapassar 1500ms

### Requirement: Volume sobre a tabela de auditoria

O plano SHALL exercer a escrita e a consulta de idempotência contra a tabela de auditoria com
volume representativo (~400M linhas), para verificar que o índice por `traceId` sustenta a
latência de SELECT sob carga de escrita concorrente.

#### Scenario: Consulta de idempotência sob volume alto

- **WHEN** a verificação de idempotência por `traceId` executa contra a tabela de ~400M linhas
  durante escrita concorrente de auditoria
- **THEN** a latência da verificação de idempotência SHALL permanecer dentro de um orçamento
  que não faça a requisição síncrona exceder p95 < 1.5s, e esse orçamento SHALL ser reportado
  por métrica customizada

### Requirement: Estratégia de massa que não mascara escrita (T5)

O plano SHALL gerar massa com `traceId` e `customerId` únicos em volume de milhões, de modo
que a carga de escrita real seja exercida e o caminho de idempotência não absorva
indevidamente as requisições.

#### Scenario: Massa única força caminho de escrita

- **WHEN** o teste de volume usa `traceId`/`customerId` únicos por iteração
- **THEN** a proporção de requisições que seguem o caminho de escrita (não curto-circuitadas
  por idempotência) SHALL ser ~100%, verificada por métrica customizada

#### Scenario: Massa repetida valida o caminho de idempotência

- **WHEN** uma fração controlada de `traceId` é intencionalmente repetida
- **THEN** essas requisições SHALL seguir o caminho de idempotência e NÃO SHALL gerar escrita
  de auditoria adicional, isolando a medição dos dois caminhos
