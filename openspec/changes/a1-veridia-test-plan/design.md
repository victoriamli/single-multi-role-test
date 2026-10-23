## Context

Veridia é um gateway de decisão de crédito (Kotlin + Spring Boot, Virtual Threads, K8s: 4 pods
em QA, 8 em produção) com três superfícies de carga acopladas:

- **API síncrona** `POST /v1/assessments`, que orquestra três motores HTTP em paralelo
  (`engine-score` p95 ~80ms, `engine-fraud` p95 ~250ms, `engine-bureau` p95 ~1.2s e
  **instável**). O veredito só sai quando os três respondem ou quando o timeout de 2s por
  downstream expira (sem retry). SLA contratual: **p95 < 1.5s, p99 < 2.5s**, 5xx < 0.5%,
  disponibilidade 99.9%. Throughput sustentado ~300 req/s, pico ~700 req/s.
- **Consumer Kafka** (`credit.assessment.requested` → `credit.assessment.completed`, 12
  partições), com **pico diário** de ~2M mensagens em ~40 min (~830 msg/s) contra ~50 msg/s
  fora do pico. Lag < 10s fora do pico; no pico a meta é vaga ("aceitável até 15 min").
- **Oracle**: auditoria por avaliação, tabela ~400M linhas sem expurgo, **pool de 20 conexões
  por pod compartilhado** entre a API síncrona e o consumer — ponto de acoplamento central.

O plano deve exercer esses caminhos de forma ancorada nos 5 fluxos críticos e nas 5 tensões:

- **T1** SLA apertado vs. bureau lento/instável.
- **T2** pico Kafka 2M/40 min vs. 50 msg/s (classificar o tipo de teste).
- **T3** pool Oracle compartilhado (API + consumer) no pico.
- **T4** meta de lag no pico vaga → virar threshold testável.
- **T5** massa mal gerada mascara a carga real de escrita via idempotência.

A execução é feita pela aplicação interna de QA via K6 (binário ou Docker), scripts
versionados em pasta interna, com motor E2E declarativo (Gherkin, HTTP + Kafka). O ambiente
primário de execução é **QA (4 pods)**; números de produção (8 pods) são usados como
referência de extrapolação, não medidos diretamente.

## Goals / Non-Goals

**Goals:**

- Validar ponta-a-ponta os 5 fluxos de negócio (aprovação direta, recusa por fraude, fallback
  do bureau, caminho assíncrono Kafka, idempotência) com critérios de aceite mensuráveis.
- Medir a API síncrona sob carga nominal (~300 req/s) e localizar o ponto de ruptura (stress),
  com thresholds derivados do SLA.
- Caracterizar o pico Kafka e **substituir a meta vaga de lag** por um threshold testável.
- Expor a contenção do pool Oracle executando API + Kafka **concorrentemente** (não isolado).
- Definir a estratégia de massa para que o teste de volume exerça a escrita real, sem mascarar
  por idempotência.
- Entregar, para cada teste: tipo, alvo, modelo de carga (executor/estágios), thresholds
  numéricos, massa e critério de aceite.

**Non-Goals:**

- Implementar os scripts K6 completos (só trechos ilustrativos de `options`/`thresholds`).
- Redesenhar o alvo ou propor mudanças de arquitetura (ex.: adicionar cache ao bureau, retry,
  aumentar o pool). O plano testa Veridia como ele é; onde uma mudança resolveria o risco, o
  registro vira recomendação, não parte do teste.
- Testar diretamente o ambiente de produção (8 pods); QA (4 pods) é o alvo de execução.
- Validar correção funcional de negócio além dos vereditos observáveis nos 5 fluxos.

## Decisions

### D1 — Quais tipos de teste se aplicam (e quais não)

| Tipo | Aplica? | Por quê (ancorado no alvo) |
|------|---------|-----------------------------|
| **E2E** | Sim | Os 5 fluxos críticos precisam de validação funcional ponta-a-ponta, incluindo o fallback do bureau e a idempotência. |
| **Load** | Sim | Verificar o SLA no patamar sustentado de produção (~300 req/s) e o pico de ~700 req/s. |
| **Stress** | Sim | SLA apertado + bureau instável (T1) exigem localizar o knee point acima de 700 req/s. |
| **Spike** | Sim | A API tem pico declarado (300→700) e o consumer tem transição abrupta 50→830 msg/s; ambos precisam de teste de choque e recuperação. |
| **Volume** | Sim | Tabela de auditoria ~400M linhas + ~2M mensagens no pico Kafka (T3/T5). |
| **Soak** | Parcial | O pico Kafka de ~40 min funciona como soak curto para detectar crescimento monotônico de lag/memória; um soak longo (horas) fica como item opcional, pois o risco dominante é o pico, não a operação prolongada em regime baixo. |

**Alternativa considerada:** tratar o pico Kafka como spike puro. Rejeitada: 40 min de taxa
estável é carga sustentada de alta taxa, não um choque instantâneo — por isso o pico é modelado
como `constant-arrival-rate` longo (T2), e o *início* do pico é coberto por um spike separado.

### D2 — Modelagem de carga da API síncrona (open model)

Usar executores de **chegada** (`constant-arrival-rate` / `ramping-arrival-rate`), não
`ramping-vus`, porque o SLA é sobre taxa de requisições e a latência do bureau não deve
"segurar" a geração de carga (modelo fechado mascararia a saturação). Baseline de VUs
dimensionado para cobrir a latência de cauda (requisição pode durar até ~2s por causa do
bureau).

Perfis:

- **Load nominal:** `constant-arrival-rate` a 300 req/s por 10–15 min.
- **Spike:** `ramping-arrival-rate` 300→700 em ≤30s, mantém 2 min, volta a 300.
- **Stress:** `ramping-arrival-rate` em degraus a partir de 700 req/s até degradar SLA.

Trecho ilustrativo (apenas para explicitar a decisão, não é o script):

```javascript
export const options = {
  scenarios: {
    load_nominal: {
      executor: 'constant-arrival-rate',
      rate: 300, timeUnit: '1s', duration: '15m',
      preAllocatedVUs: 700, maxVUs: 1200, // cobre latência de cauda ~2s
    },
  },
  thresholds: {
    'http_req_duration': ['p(95)<1500', 'p(99)<2500'],
    'http_req_failed':   ['rate<0.005'], // 5xx < 0.5%
  },
};
```

### D3 — Perfil "bureau instável" (T1)

O risco #1 é o `engine-bureau`: p95 ~1.2s sozinho já consome quase todo o orçamento de 1.5s.
O teste de carga roda em **dois modos**: (a) bureau saudável e (b) bureau instável, com uma
fração configurável de chamadas sofrendo timeout (>2s) ou 5xx. Como Veridia não tem controle
sobre o bureau real em QA, a instabilidade é induzida por um **stub/mocked downstream** no
ambiente de QA (latência e taxa de erro parametrizáveis). O critério central: o fallback
`APPROVED_WITH_REVIEW` deve manter `http_req_duration` p95 < 1.5s e não propagar 5xx (<0.5%).
Métrica customizada conta a proporção de `APPROVED_WITH_REVIEW` para correlacionar com a
injeção de falha.

**Alternativa considerada:** usar o bureau real em QA. Rejeitada: instabilidade não é
reproduzível/controlável e contaminaria a medição; o stub torna o teste determinístico.

### D4 — Classificação e threshold do pico Kafka (T2, T4)

O pico é **carga sustentada de alta taxa**: `constant-arrival-rate` a ~830 msg/s por ~40 min
via `xk6-kafka`, totalizando ~2M mensagens. O *início* do pico (50→830) é um **spike**
separado para observar rebalance e recuperação de lag.

**Transformar a meta vaga em threshold (T4):** "aceitável até 15 min" é ambíguo (lag máximo?
tempo de drenagem?). Proposta do plano:

- **Lag máximo durante o pico ≤ 5 min** (mais estrito que 15 min; justificativa: 12 partições
  a ~830 msg/s significam ~70 msg/s por partição; um backlog de 5 min ≈ 250k mensagens, ainda
  recuperável sem esgotar memória/offset retention).
- **Tempo de drenagem pós-pico até lag < 10s ≤ 15 min.**
- Taxa de erro de processamento < 0.5%; zero perda de mensagem (todo `traceId` produzido tem
  `completed` correspondente).

Esses números são **autoria do plano** e ficam marcados como pendência de validação com os
donos do SLA (ver Open Questions), pois derivam de uma meta originalmente não quantificada.

### D5 — Contenção do pool Oracle: testar concorrente, não isolado (T3)

Testar API e Kafka separadamente **esconde** o acoplamento do pool (20 conexões/pod). O teste
decisivo roda os dois ao mesmo tempo: API a ~300 req/s + pico Kafka a ~830 msg/s contra o
mesmo cluster QA. Compara-se o p95 da API nesse cenário com o baseline isolado (D2) e
atribui-se a degradação ao tempo de aquisição de conexão. Onde a aplicação de QA/observabilidade
exponha, coletar o tempo de espera no pool como métrica; caso contrário, inferir pela diferença
de p95 entre isolado e concorrente. Critério: p95 da API < 1.5s mesmo sob concorrência; se
falhar, o relatório documenta a contenção do pool como gargalo.

### D6 — Estratégia de massa (T5)

- **Volume real de escrita:** gerar `traceId` e `customerId` **únicos** por iteração (milhões),
  garantindo ~100% das requisições no caminho de escrita. Geração: função determinística no K6
  (`uuid`/contador + prefixo de execução) para evitar colisão entre execuções e manter
  **isolamento** (nenhuma execução corrompe dados de outra).
- **Validação explícita do caminho de idempotência:** uma fração controlada (ex.: 5–10%) de
  `traceId` repetidos, verificando que não geram auditoria adicional. Métrica customizada
  separa os dois caminhos.
- **Pré-condição de volume:** a tabela de auditoria deve estar com volume representativo
  (~400M linhas) antes do teste de volume; caso o ambiente QA não tenha, registrar como
  pré-requisito de dados (seed) — senão a latência do SELECT de idempotência não reflete a
  produção.
- CPF/CNPJ sintéticos e `customerId` variados para refletir dispersão realista.

### D7 — Ambiente, isolamento e extrapolação

- Execução primária em **QA (4 pods)**. Os thresholds de SLA valem como estão (o SLA é do
  serviço, não do pod). Para raciocinar sobre produção (8 pods), documenta-se a premissa de
  escala linear como referência, sem medi-la.
- **Isolamento entre execuções:** prefixo de execução nos dados, tópicos/grupos de consumo
  dedicados ao teste quando possível, e janela de execução que não colida com o pico real de
  23h–00h.
- Downstreams (score/fraud/bureau) via stub parametrizável em QA para reprodutibilidade (D3).

### D8 — Matriz de testes (resumo executável)

| ID | Tipo | Alvo | Modelo de carga | Thresholds / Aceite | Massa |
|----|------|------|-----------------|---------------------|-------|
| E2E-1..5 | E2E | 5 fluxos | 1 req/cenário (Gherkin) | veredito correto; fallback < 1.5s p95; 1 auditoria por traceId | poucos casos determinísticos |
| L-1 | Load | API | const-arrival 300 rps, 15 min, bureau saudável | p95<1.5s, p99<2.5s, 5xx<0.5% | traceId únicos |
| L-2 | Load | API | const-arrival 300 rps, bureau instável (T1) | p95<1.5s, 5xx<0.5%; mede % APPROVED_WITH_REVIEW | traceId únicos |
| SP-1 | Spike | API | ramping-arrival 300→700→300 | disp.≥99.9%; p99 volta <2.5s em ≤60s | traceId únicos |
| ST-1 | Stress | API | ramping-arrival degraus >700 até quebrar | registra knee point (sem gate de SLA) | traceId únicos |
| K-1 | Load | Kafka | const-arrival 50 msg/s, 10 min | lag<10s; 0 perda | traceId únicos |
| K-2 | Volume/soak | Kafka | const-arrival 830 msg/s, 40 min (~2M) | lag≤5 min; drena <10s em ≤15 min; erro<0.5% | 2M traceId únicos |
| K-3 | Spike | Kafka | 50→830 em ≤1 min | sem perda; lag decrescente em ≤5 min | traceId únicos |
| C-1 | Volume+contention | API+Kafka+Oracle | L-1 + K-2 concorrentes (T3) | p95 API<1.5s sob concorrência; reporta espera no pool | mix único |
| V-1 | Volume | Oracle idempotência | SELECT sob ~400M linhas + escrita concorrente | latência de idempotência dentro do orçamento p95<1.5s | seed 400M + traceId únicos |

## Risks / Trade-offs

- **[Bureau real indisponível/instável em QA]** → Usar stub parametrizável (D3); registra-se
  que o teste mede o comportamento de Veridia frente à instabilidade, não o bureau em si.
- **[QA tem 4 pods, SLA pensado para produção/8 pods]** → Reportar resultados por serviço e
  marcar a extrapolação para 8 pods como premissa não medida (D7); evitar afirmar conformidade
  de produção a partir de QA.
- **[Ambiente QA sem 400M linhas de auditoria]** → A latência de idempotência não reflete a
  produção; seed de volume vira pré-requisito explícito (D6). Sem o seed, V-1 e C-1 perdem
  validade e devem ser marcados como inconclusivos.
- **[Threshold de lag de 5 min é autoral]** → Pode não refletir o apetite real do negócio;
  marcado como pendência (Open Questions / T4).
- **[Gerar 2M traceId únicos + massa de clientes]** → Custo de geração e de dados; mitigar com
  geração em streaming no K6 e prefixo de execução para isolamento (D6).
- **[Executar API + Kafka concorrentes pode saturar o cluster QA além do representativo]** →
  Dimensionar C-1 para refletir a razão real de produção; se QA não comportar, reduzir
  proporcionalmente e documentar o fator de escala.
- **[Teste colidir com o pico real de 23h-00h]** → Agendar fora dessa janela para não
  contaminar medições nem o ambiente (D7).
- **[Modelo fechado mascararia saturação]** → Mitigado pela escolha de executores de chegada
  (D2).

## Migration Plan

Não há migração de produção: o entregável é um plano de testes. Sequência de adoção sugerida:

1. Preparar ambiente QA: stubs dos três motores parametrizáveis, seed de auditoria
   representativa, tópicos/grupos de consumo dedicados ao teste.
2. Rodar E2E (E2E-1..5) como gate funcional antes dos testes de carga.
3. Estabelecer baseline isolado: L-1, K-1.
4. Rodar perfis de risco: L-2 (bureau instável), SP-1, K-2, K-3.
5. Rodar o teste de acoplamento C-1 e o volume V-1.
6. Rodar ST-1 (stress) por último para não contaminar os demais com o estado degradado.
7. Rollback: nenhum efeito em produção; limpar dados de teste por prefixo de execução.

## Open Questions

- **T4:** O threshold de lag no pico (≤ 5 min) e o tempo de drenagem (≤ 15 min) precisam de
  confirmação dos donos do SLA — hoje são proposta do plano.
- **Retenção/offset do Kafka:** qual o retention configurado? Define se um backlog de 5 min é
  seguro quanto a expiração de mensagens durante o pico.
- **Escala QA→produção:** há dados que confirmem escala ~linear de 4→8 pods, ou o SLA só vale
  medido em produção?
- **Observabilidade do pool Oracle:** a aplicação de QA expõe tempo de aquisição de conexão?
  Se não, C-1 depende de inferência por diferença de p95.
- **Soak longo:** o negócio quer validar operação prolongada (horas) além do pico de 40 min?
