## ADDED Requirements

### Requirement: Spike de instabilidade do bureau sob o SLA apertado
O plano SHALL definir um teste que injete latência e erros (timeout/5xx) no `engine-bureau`
durante carga nominal, medindo o impacto no SLA da API e a taxa de disparo do fallback (T1).

#### Scenario: Injeção de latência elevada no bureau
- **WHEN** sob 300 req/s, o stub de `engine-bureau` passa a responder com p95 ~2s+ (acima do
  timeout de 2s) por uma janela de alguns minutos
- **THEN** o teste SHALL medir a fração de respostas `APPROVED_WITH_REVIEW` (fallback) e
  verificar que `http_req_duration` p95 permanece < 1.5s porque o timeout corta a espera

#### Scenario: Picos de 5xx do bureau
- **WHEN** o stub de `engine-bureau` injeta 5xx em uma fração configurável das chamadas
  (ex.: 10–30%)
- **THEN** a taxa de erro 5xx **da API** SHALL permanecer < 0.5% (o erro do downstream não
  deve propagar como 5xx do gateway; deve virar fallback), caso contrário o teste reprova

### Requirement: Ausência de retry como risco explícito
O plano SHALL registrar que, por não haver retry automático, cada instabilidade do bureau se
reflete direto no veredito, e o teste SHALL medir a sensibilidade do SLA a essa ausência.

#### Scenario: Correlação instabilidade → fallback
- **WHEN** a taxa de falha injetada no bureau varia em degraus
- **THEN** o teste SHALL reportar a curva "taxa de falha do bureau × taxa de fallback" como
  evidência de resiliência, sem alterar a arquitetura do alvo
