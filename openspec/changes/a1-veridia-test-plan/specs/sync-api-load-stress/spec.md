## ADDED Requirements

### Requirement: Carga nominal da API síncrona

O plano SHALL exercer `POST /v1/assessments` no patamar sustentado de produção (~300 req/s)
por tempo suficiente para estabilizar percentis, usando o executor `constant-arrival-rate`
(carga aberta, independente da latência), e SHALL aprovar apenas se os thresholds derivados
do SLA forem respeitados.

#### Scenario: Carga sustentada a 300 req/s com downstream saudável

- **WHEN** a API recebe ~300 req/s por ≥ 10 min com os três motores em latência nominal
- **THEN** `http_req_duration` SHALL ter p95 < 1500ms e p99 < 2500ms e a taxa de erro 5xx
  (`http_req_failed` sobre respostas 5xx) SHALL ser < 0.5%

### Requirement: Carga com bureau instável (T1)

O plano SHALL incluir um perfil de carga em que `engine-bureau` apresenta a instabilidade
descrita (picos de timeout/5xx), para verificar que o fallback `APPROVED_WITH_REVIEW` mantém
o SLA do endpoint mesmo quando o downstream degrada.

#### Scenario: 300 req/s com injeção de timeouts do bureau

- **WHEN** a API recebe ~300 req/s e uma fração configurável das chamadas ao `engine-bureau`
  estoura o timeout de 2s ou retorna 5xx
- **THEN** o p95 de `http_req_duration` SHALL permanecer < 1500ms, a taxa de 5xx propagada ao
  chamador SHALL ser < 0.5%, e a proporção de vereditos `APPROVED_WITH_REVIEW` SHALL ser
  mensurada por métrica customizada

### Requirement: Spike na API síncrona

O plano SHALL aplicar um spike (subida abrupta para o pico declarado de ~700 req/s) usando
`ramping-arrival-rate`, para avaliar recuperação e estabilidade de percentis após o choque.

#### Scenario: Salto de 300 para 700 req/s

- **WHEN** a taxa salta de ~300 para ~700 req/s em ≤ 30s e se mantém por ≥ 2 min
- **THEN** a disponibilidade SHALL permanecer ≥ 99.9% durante a janela e o p99 SHALL retornar
  a < 2500ms em até 60s após o fim do spike

### Requirement: Stress até ruptura da API

O plano SHALL aumentar a taxa de chegada além do pico (rampa crescente via
`ramping-arrival-rate`) até degradar os SLAs, registrando o ponto de ruptura (taxa em que p95
ultrapassa 1.5s ou erro 5xx ultrapassa 0.5%).

#### Scenario: Rampa crescente além de 700 req/s

- **WHEN** a taxa de chegada cresce em degraus a partir de 700 req/s até a degradação
- **THEN** o teste SHALL registrar o knee point (taxa de ruptura) e as métricas de saturação
  (p95/p99, erro 5xx, saturação observável) no relatório, sem exigir aprovação de threshold
  de SLA nesta fase exploratória
