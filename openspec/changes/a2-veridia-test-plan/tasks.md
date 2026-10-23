## 1. Pré-condições de ambiente
- [ ] 1.1 Provisionar stubs dos 3 motores (`engine-score`, `engine-fraud`, `engine-bureau`) com injeção configurável de latência e 5xx (necessário para E2E-3, SP-1)
- [ ] 1.2 Preparar seed Oracle realista (~400M linhas de auditoria) para idempotência e volume (pré-condição de VL-1, CT-1, E2E-5)
- [ ] 1.3 Habilitar observabilidade do alvo: lag por partição, uso do pool por pod, latência por downstream, taxa de fallback (pré-condição de medição de T3/T4)
- [ ] 1.4 Configurar gerador de massa: `traceId` únicos (`<runId>-<vu>-<iter>`) + pool amplo de `customerId`/documentos (T5)

## 2. Testes E2E dos fluxos críticos
- [ ] 2.1 E2E-1 aprovação direta: motores OK → `APPROVED`, checar corpo e `p95<1500/p99<2500`
- [ ] 2.2 E2E-2 recusa por fraude: `engine-fraud` sinaliza risco → `DECLINED` mesmo com score bom
- [ ] 2.3 E2E-3 fallback do bureau: stub timeout>2s → `APPROVED_WITH_REVIEW` dentro de `p95<1500` (T1)
- [ ] 2.4 E2E-4 Kafka async: produzir em `credit.assessment.requested`, verificar `completed` com mesmo `traceId`, lag `<10s`
- [ ] 2.5 E2E-5 idempotência: mesmo `traceId` 2× → 1 resultado e auditoria count=1 (T5)

## 3. Load e stress da API síncrona
- [ ] 3.1 LD-1 load sustentado: `constant-arrival-rate` 300 req/s, 10 min + warm-up; thresholds `p95<1500,p99<2500,http_req_failed<0.005`
- [ ] 3.2 LD-2 pico de produção: `ramping-arrival-rate` →700 req/s por 5 min; mesmos thresholds de SLA
- [ ] 3.3 ST-1 stress knee point: degraus 700→1100+ req/s; registrar taxa de violação de `p95≥1500`/`5xx≥0.5%` e recurso saturado (T1, T3)
- [ ] 3.4 Dimensionar `preAllocatedVUs`/`maxVUs` com folga e monitorar `dropped_iterations` em todos os testes de arrival-rate

## 4. Resiliência à instabilidade do bureau (T1)
- [ ] 4.1 SP-1 injetar latência>2s no bureau sob 300 req/s; medir fração de fallback e manter `p95<1500`
- [ ] 4.2 SP-1 injetar 5xx 10–30% no bureau; verificar que 5xx da API permanece `<0.5%` (vira fallback, não erro do gateway)
- [ ] 4.3 Produzir a curva "taxa de falha do bureau × taxa de fallback" variando a injeção em degraus

## 5. Volume do pico Kafka e threshold de lag (T2, T4)
- [ ] 5.1 VL-1 injetar pico via `ramping-arrival-rate` (xk6-kafka) 50→830 msg/s até ~2M msgs; medir lag por partição e tempo de drenagem
- [ ] 5.2 Verificar integridade: nº de `completed` = nº de pedidos únicos, sem perdas/duplicatas
- [ ] 5.3 Validar threshold proposto: lag ≤ 5 min no pico e backlog drenado ≤ 15 min após a injeção
- [ ] 5.4 **[decisão humana]** homologar o threshold de lag com o dono do SLA (substitui a meta vaga "15 min") (T4)

## 6. Concorrência do pool Oracle (T3)
- [ ] 6.1 CT-1 rodar um único K6 com 2 `scenarios`: HTTP `constant-arrival-rate` 300 req/s + Kafka `ramping-arrival-rate` →830 msg/s
- [ ] 6.2 Calcular `Δp95 = p95(concorrente) − p95(isolado)` atribuível à disputa do pool
- [ ] 6.3 Coletar métricas do pool (conexões ativas, fila de espera) e correlacionar saturação com a degradação da API

## 7. Soak opcional e consolidação
- [ ] 7.1 SK-1 (opcional) soak `constant-arrival-rate` ~300 msg/s por 2–4h; checar deriva de `p95`/lag (pool/GC)
- [ ] 7.2 Consolidar baseline isolado vs. concorrente e reportar resultados por pod (premissa de extrapolação QA 4 pods → prod 8 pods)
- [ ] 7.3 Registrar pendências remanescentes (T4, seed 400M, partições Kafka, observabilidade do pool) para decisão humana
