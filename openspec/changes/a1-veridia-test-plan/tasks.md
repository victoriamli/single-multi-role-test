## 1. Preparação de ambiente e massa

- [ ] 1.1 Provisionar ambiente QA (4 pods) com observabilidade de latência, lag Kafka e pool Oracle
- [ ] 1.2 Configurar stubs parametrizáveis de `engine-score`, `engine-fraud`, `engine-bureau` (latência e taxa de erro/timeout ajustáveis) (D3)
- [ ] 1.3 Preparar seed da tabela de auditoria com volume representativo (~400M linhas) como pré-requisito de V-1/C-1 (D6)
- [ ] 1.4 Criar geração de massa com `traceId`/`customerId` únicos e prefixo de execução para isolamento (D6, T5)
- [ ] 1.5 Reservar tópicos/grupos de consumo dedicados ao teste e janela fora do pico real 23h-00h (D7)

## 2. E2E — fluxos críticos (gate funcional)

- [ ] 2.1 E2E-1 aprovação direta: 3 motores OK → `APPROVED` dentro do SLA + auditoria gravada
- [ ] 2.2 E2E-2 recusa por fraude: `engine-fraud` sinaliza risco → `DECLINED` mesmo com score bom
- [ ] 2.3 E2E-3 fallback do bureau: timeout 2s / 5xx → `APPROVED_WITH_REVIEW` com p95 < 1.5s (T1)
- [ ] 2.4 E2E-4 caminho assíncrono Kafka: `requested` → `completed` com mesmo `traceId` (xk6-kafka)
- [ ] 2.5 E2E-5 idempotência: mesmo `traceId` reprocessado → 1 auditoria, 1 resultado

## 3. Carga e stress da API síncrona

- [ ] 3.1 L-1 load nominal: `constant-arrival-rate` 300 rps, 15 min, bureau saudável; gates p95<1.5s, p99<2.5s, 5xx<0.5%
- [ ] 3.2 L-2 load com bureau instável (T1): 300 rps + injeção de timeout/5xx; mede % `APPROVED_WITH_REVIEW` por métrica customizada
- [ ] 3.3 SP-1 spike: `ramping-arrival-rate` 300→700→300; disponibilidade ≥99.9%, p99 volta <2.5s em ≤60s
- [ ] 3.4 ST-1 stress: rampa em degraus acima de 700 rps até degradar; registrar knee point (fase exploratória, sem gate de SLA)

## 4. Kafka — fora de pico, pico e spike

- [ ] 4.1 K-1 consumo fora de pico: `constant-arrival-rate` 50 msg/s, 10 min; lag<10s, 0 perda
- [ ] 4.2 K-2 pico noturno (T2): `constant-arrival-rate` ~830 msg/s por ~40 min (~2M msgs); lag≤5 min, erro<0.5%
- [ ] 4.3 K-2 drenagem pós-pico (T4): lag volta a <10s em ≤15 min após o fim do pico
- [ ] 4.4 K-3 spike de início de pico: 50→830 em ≤1 min; sem perda, lag decrescente em ≤5 min

## 5. Contenção Oracle e volume

- [ ] 5.1 C-1 concorrência (T3): rodar L-1 + K-2 simultâneos; p95 API<1.5s sob concorrência; reportar espera no pool vs. baseline isolado
- [ ] 5.2 V-1 volume de idempotência: SELECT por `traceId` sob ~400M linhas + escrita concorrente; latência dentro do orçamento p95<1.5s
- [ ] 5.3 Validar caminhos de massa (T5): ~100% escrita com `traceId` únicos; fração repetida confirma idempotência sem escrita extra

## 6. Consolidação e pendências

- [ ] 6.1 Consolidar relatório por teste: tipo, modelo de carga, thresholds, aceite, massa
- [ ] 6.2 Registrar knee point da API (ST-1) e o gargalo observado (bureau vs. pool Oracle)
- [ ] 6.3 **Pendência humana (T4):** confirmar com donos do SLA o threshold de lag no pico (≤5 min) e a drenagem (≤15 min)
- [ ] 6.4 **Pendência humana:** confirmar retention do Kafka, escala QA→produção (4→8 pods) e observabilidade do pool Oracle
