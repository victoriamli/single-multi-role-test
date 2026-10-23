# Requisitos de teste — Veridia (gateway de decisão de crédito)

> Braço **B1 — Spec do Kiro + single-role**. Artefato de requisitos no estilo do modo spec
> nativo do Kiro (EARS / "SHALL"). Todos os requisitos derivam de `scenario/target-app.md`
> (fluxos, SLAs, tensões T1–T5). Cada requisito é verificável por um threshold numérico ou
> uma asserção de resultado observável.
>
> Escopo: **planejamento** de testes K6 (E2E, load, stress, spike, soak, volume). Não inclui
> implementação de scripts nem redesenho do alvo.

## Convenções

- **WHEN / WHILE** descreve a condição de disparo; **THE test suite SHALL** descreve o
  comportamento verificável esperado.
- Cada requisito referencia o(s) fluxo(s) crítico(s) (F1–F5) e/ou tensão (T1–T5) de origem.
- IDs: `RQ-E2E-*` (ponta a ponta), `RQ-LOAD-*`, `RQ-STRESS-*`, `RQ-SPIKE-*`, `RQ-SOAK-*`,
  `RQ-VOL-*`, `RQ-DATA-*` (massa), `RQ-MEAS-*` (integridade de medição).

---

## 1. Requisitos funcionais / E2E (fluxos críticos)

### RQ-E2E-1 — Aprovação direta (F1)
WHEN um pedido é enviado a `POST /v1/assessments` para um cliente elegível e os três motores
(`engine-score`, `engine-fraud`, `engine-bureau`) retornam OK, THE test suite SHALL verificar
que a resposta é `APPROVED` na mesma requisição (síncrona) e que `http_req_duration` do fluxo
fica dentro de p95 < 1.5s / p99 < 2.5s.

### RQ-E2E-2 — Recusa por fraude (F2)
WHEN `engine-fraud` sinaliza risco, THE test suite SHALL verificar que o veredito é `DECLINED`
mesmo quando `engine-score` retorna score bom, confirmando a precedência da regra de fraude.

### RQ-E2E-3 — Fallback de degradação do bureau (F3, T1)
WHEN `engine-bureau` excede o timeout de 2s (estímulo de falha injetado), THE test suite SHALL
verificar que o veredito retornado é `APPROVED_WITH_REVIEW` e que o fluxo de fallback completa
dentro do SLA do endpoint (p95 < 1.5s / p99 < 2.5s), sem propagar 5xx ao cliente.

### RQ-E2E-4 — Caminho assíncrono Kafka (F4)
WHEN uma mensagem é publicada em `credit.assessment.requested`, THE test suite SHALL verificar
que um resultado correspondente aparece em `credit.assessment.completed` com o **mesmo
`traceId`** e veredito coerente com o pedido.

### RQ-E2E-5 — Idempotência (F5, T5)
WHEN o mesmo `traceId` é reprocessado (via Kafka ou API), THE test suite SHALL verificar que
**não** há registro de auditoria duplicado no Oracle e que o resultado permanece consistente
entre as repetições.

---

## 2. Requisitos de carga (load) — carga esperada de produção

### RQ-LOAD-1 — API síncrona em throughput sustentado (SLA p95/p99)
WHILE a API recebe a taxa sustentada de produção (~300 req/s) por ≥ 30 min, THE test suite
SHALL verificar que `http_req_duration` mantém p95 < 1.5s e p99 < 2.5s e que a taxa de erro
5xx permanece < 0.5%.

### RQ-LOAD-2 — Consumer Kafka fora do pico
WHILE o tópico `credit.assessment.requested` recebe ~50 msg/s (volume normal), THE test suite
SHALL verificar que o **lag de consumo** do consumer group permanece < 10s de forma sustentada.

### RQ-LOAD-3 — Mix de veredito representativo
WHILE a carga nominal é aplicada, THE test suite SHALL distribuir os pedidos entre os fluxos
F1/F2/F3 em proporção declarada (ex.: maioria aprovação, fração de fraude, fração de
degradação do bureau) para que os percentis de latência reflitam o caminho crítico real, e não
apenas o caso rápido.

---

## 3. Requisitos de stress — acima da capacidade até a ruptura

### RQ-STRESS-1 — Ponto de ruptura da API síncrona (T1)
WHILE a taxa de chegada à API é elevada em estágios acima do pico de produção (~700 req/s e
além), THE test suite SHALL identificar o ponto (**knee**) em que p95 cruza 1.5s ou 5xx cruza
0.5%, reportando a taxa de chegada e o nº de requisições em voo nesse ponto.

### RQ-STRESS-2 — Saturação do pool Oracle sob concorrência (T3)
WHILE a API síncrona e o consumer Kafka geram carga **simultânea** disputando o pool de
conexões Oracle (20 por pod), THE test suite SHALL medir o impacto no p95 da API comparado ao
mesmo teste isolado (Δp95) e verificar se a saturação do pool antecipa a ruptura.

### RQ-STRESS-3 — Amplificação da instabilidade do bureau (T1)
WHILE a taxa de timeout/5xx injetada no `engine-bureau` é aumentada sob carga de stress, THE
test suite SHALL verificar que a taxa de `APPROVED_WITH_REVIEW` cresce de forma controlada e
que o endpoint não degrada além do SLA de erro (5xx < 0.5%) nem esgota os virtual threads.

---

## 4. Requisitos de spike — pico abrupto

### RQ-SPIKE-1 — Pico noturno Kafka (T2, T4)
WHEN ~2 milhões de mensagens são injetadas em `credit.assessment.requested` em ~40 min
(perfil do batch noturno 23h–00h), THE test suite SHALL verificar que o **backlog máximo** e o
**tempo de drenagem** permanecem dentro dos thresholds propostos (ver RQ-SOAK-1 / design) e
que nenhuma mensagem é perdida.

### RQ-SPIKE-2 — Pico abrupto na API síncrona
WHEN a taxa de chegada à API salta abruptamente de ~300 para ~700 req/s, THE test suite SHALL
verificar que a recuperação ocorre sem erros persistentes (5xx < 0.5% após estabilização) e
que o p95 retorna ao SLA dentro de uma janela definida.

---

## 5. Requisitos de soak — carga prolongada

### RQ-SOAK-1 — Drenagem sustentada do pico Kafka (T2, T4)
WHILE o backlog gerado pelo pico noturno é drenado em condição de carga concorrente da API,
THE test suite SHALL verificar que o lag retorna a < 10s dentro do limite proposto para o pico
(número a ser ancorado no design, derivado da meta vaga "aceitável até 15 min") sem crescimento
monotônico de memória/conexões.

### RQ-SOAK-2 — Estabilidade de recursos sob carga nominal prolongada
WHILE a carga nominal da API é sustentada por período prolongado, THE test suite SHALL
verificar ausência de vazamento observável (crescimento de latência/erros ao longo do tempo
sem aumento de carga).

---

## 6. Requisitos de volume — grande massa acumulada

### RQ-VOL-1 — Idempotência com tabela de auditoria grande (T3, T5)
WHILE a tabela de auditoria contém volume realista (~400M linhas) e a carga de escrita
concorrente está ativa, THE test suite SHALL verificar que a consulta de idempotência por
`traceId` (via índice) mantém latência estável e não se torna o gargalo dominante da escrita.

### RQ-VOL-2 — Escrita concorrente sob volume
WHILE o volume acumulado está presente, THE test suite SHALL verificar que a gravação do
registro de auditoria sob carga de pico não viola o SLA de latência da API nem esgota o pool
de conexões.

---

## 7. Requisitos de dados de massa (T5)

### RQ-DATA-1 — Unicidade por padrão
THE test suite SHALL gerar `traceId` e `customerId` **únicos** por padrão para refletir a
carga real de escrita, evitando que a reutilização acione inadvertidamente o caminho de
idempotência e mascare a carga.

### RQ-DATA-2 — Fração controlada de repetição
THE test suite SHALL incluir uma **fração explícita e marcada** de `traceId` repetidos,
exclusivamente para exercitar e medir o caminho de idempotência (RQ-E2E-5 / RQ-VOL-1), sem
contaminar as medições de carga de escrita.

### RQ-DATA-3 — Seed de volume separado da massa de teste
THE test suite SHALL tratar o seed de ~400M linhas de auditoria como **pré-condição** dos
testes de volume, com `traceId` de seed distintos dos `traceId` gerados em runtime, de modo
que o seed não colida com a massa de teste.

---

## 8. Requisitos de integridade de medição (T3)

### RQ-MEAS-1 — Teste isolado vs. concorrente
THE test suite SHALL executar os caminhos síncrono e assíncrono tanto **isoladamente** quanto
em **concorrência**, reportando ambos, para que o acoplamento via pool Oracle (T3) não fique
oculto por medições isoladas.

### RQ-MEAS-2 — Modelo de chegada aberto
THE test suite SHALL usar executores K6 de **taxa de chegada** (open model) para os testes de
carga/stress da API, de modo que VUs presos em respostas lentas do bureau não mascarem a taxa
real oferecida nem os percentis sob o SLA.

### RQ-MEAS-3 — Escala QA vs. produção explícita
THE test suite SHALL registrar que os testes rodam em QA (4 pods) e SHALL marcar a
extrapolação para produção (8 pods) como não-linear, evitando concluir conformidade de
produção a partir de medições de QA sem ajuste.

### RQ-MEAS-4 — Thresholds como critério de parada
THE test suite SHALL expressar todos os critérios de aceite como `thresholds` K6
mensuráveis (p95/p99 de `http_req_duration`, taxa de erro, métricas customizadas de lag),
abortando/falhando a execução quando violados, sem metas vagas.
