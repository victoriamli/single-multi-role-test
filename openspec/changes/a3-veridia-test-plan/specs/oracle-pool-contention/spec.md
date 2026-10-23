## ADDED Requirements

### Requirement: Sobreposição API + consumer Kafka (T3)
O plano SHALL especificar um teste que executa a carga da API síncrona e o pico do consumer
Kafka simultaneamente, para expor a disputa do pool Oracle compartilhado (20 conexões/pod).

#### Scenario: Carga concorrente nos dois caminhos
- **WHEN** a API recebe ~300 req/s e o consumer processa ~833 msg/s ao mesmo tempo
- **THEN** o uso e o tempo de espera do pool Oracle são medidos e o teste captura se a
  contenção degrada a API, o consumer ou ambos

### Requirement: Impacto diferencial no SLA da API
O plano SHALL medir o impacto da carga Kafka concorrente sobre a latência da API de forma
diferencial (solo vs sobreposição), com meta numérica.

#### Scenario: Δp95 da API com e sem Kafka concorrente
- **WHEN** compara-se o p95 da API isolada com o p95 da API durante a sobreposição
- **THEN** o aumento `Δp95_api` permanece ≤ 150ms (≤ ~10% do orçamento de SLA) e qualquer
  valor acima é reportado como falha de acoplamento

### Requirement: Risco de medição por teste isolado
O plano SHALL registrar que testar os caminhos isoladamente pode dar falso verde e que a
sobreposição é obrigatória para validar o pool.

#### Scenario: Teste isolado esconde o acoplamento
- **WHEN** apenas a API OU apenas o consumer é testado isoladamente
- **THEN** o resultado é marcado como insuficiente para aceitar o comportamento sob pico, por
  não exercitar a contenção do pool
