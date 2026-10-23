# Aplicação alvo: "Veridia" — gateway de decisão de crédito

Aplicação **fictícia**, autocontida. Todo o conhecimento necessário para planejar os testes
está aqui. **Não** procurar código real; não existe código para este alvo.

> Projetada com pontos de tensão deliberados (marcados como ⚠️ **TENSÃO**) para que o
> planejamento precise ponderar trade-offs reais.

## Visão geral

**Veridia** é um gateway que recebe pedidos de avaliação de crédito e orquestra chamadas a
múltiplos motores de decisão externos, consolidando um veredito. Stack: Kotlin + Spring
Boot sobre JVM com **Virtual Threads**. Roda em Kubernetes, tipicamente **4 pods** em QA e
**8 pods** em produção.

## Interfaces

### 1. API síncrona de avaliação — `POST /v1/assessments`

- Recebe o pedido (documento do cliente, produto, valor) e responde com o veredito
  consolidado **na mesma requisição** (síncrono).
- Internamente chama **3 motores de decisão externos** via HTTP:
  - `engine-score` — rápido, p95 ~80ms.
  - `engine-fraud` — médio, p95 ~250ms.
  - `engine-bureau` — ⚠️ **TENSÃO**: consulta um bureau externo de terceiros, p95 ~1.2s e
    **instável** (picos de timeout e 5xx em horários de pico). Não tem cache.
- As 3 chamadas são feitas e o veredito só sai quando as 3 retornam (ou expira timeout).
- ⚠️ **TENSÃO**: o SLA contratual do endpoint é **p95 < 1.5s** e **p99 < 2.5s** — muito
  próximo do próprio p95 do `engine-bureau`, deixando pouca folga.
- Timeout configurado por chamada downstream: 2s. Sem retry automático hoje.

### 2. Consumer Kafka — tópico `credit.assessment.requested`

- Caminho **assíncrono**: pedidos também chegam por Kafka para processamento em lote.
- Publica o resultado em `credit.assessment.completed`.
- ⚠️ **TENSÃO**: há um **pico diário** entre 23h e 00h quando o batch noturno de um sistema
  parceiro despeja ~2 milhões de mensagens em ~40 min. Fora disso, o volume é ~50 msg/s.
- Consumer group com 12 partições. Idempotência por `traceId` (verifica Oracle antes de
  processar).

### 3. Persistência — Oracle

- Cada avaliação grava um registro de auditoria (request, veredito, latências dos motores).
- ⚠️ **TENSÃO**: a tabela de auditoria acumula sem expurgo automático; já tem ~400M linhas.
  Consultas de idempotência por `traceId` usam índice, mas a escrita concorrente durante o
  pico Kafka compete com a carga da API síncrona no mesmo pool de conexões (tamanho 20 por
  pod).

## SLAs / SLOs declarados

| Métrica                                   | Alvo                        |
|-------------------------------------------|-----------------------------|
| Latência API `POST /v1/assessments`       | p95 < 1.5s, p99 < 2.5s      |
| Disponibilidade API                       | 99.9%                       |
| Taxa de erro API (5xx)                     | < 0.5%                      |
| Lag de consumo Kafka fora do pico          | < 10s                       |
| Lag de consumo Kafka durante o pico        | ⚠️ **TENSÃO**: "aceitável até 15 min" (meta vaga, não quantificada formalmente) |
| Throughput sustentado API em produção      | ~300 req/s (pico ~700 req/s)|

## Fluxos de negócio críticos (E2E)

1. **Aprovação direta:** cliente elegível → 3 motores retornam OK → veredito `APPROVED`.
2. **Recusa por fraude:** `engine-fraud` sinaliza risco → veredito `DECLINED` mesmo com
   score bom.
3. **Degradação do bureau:** ⚠️ `engine-bureau` dá timeout → regra de negócio atual
   responde `APPROVED_WITH_REVIEW` (fallback). Testar que o fallback dispara corretamente e
   dentro do SLA.
4. **Caminho assíncrono (Kafka):** pedido entra por `credit.assessment.requested` →
   resultado em `credit.assessment.completed` com o mesmo `traceId`.
5. **Idempotência:** mesmo `traceId` reprocessado não gera auditoria duplicada.

## Dados de massa

- Clientes identificados por `customerId` e documento (CPF/CNPJ).
- ⚠️ **TENSÃO**: gerar massa realista para teste de **volume** exige milhões de `traceId`
  únicos e clientes variados; reusar o mesmo `customerId` ativa o caminho de idempotência e
  mascara a carga real de escrita.

## Pontos de tensão resumidos (para o debate)

- **T1 — SLA apertado vs downstream lento/instável:** o `engine-bureau` sozinho quase
  estoura o SLA; como desenhar stress/spike em torno disso?
- **T2 — Pico Kafka noturno:** 2M msgs em 40 min vs. 50 msg/s normal — isso é volume, soak,
  ou spike? Como modelar?
- **T3 — Pool Oracle compartilhado:** API síncrona e consumer Kafka disputam o mesmo pool
  durante o pico — teste de carga isolado por caminho esconde esse acoplamento.
- **T4 — Meta vaga de lag no pico:** "aceitável até 15 min" não é threshold testável — o
  plano precisa propor um número e justificá-lo.
- **T5 — Massa vs idempotência:** dados mal gerados mascaram a carga real de escrita.
