## 1. Preparação de ambiente e massa

- [ ] 1.1 Configurar mock do `engine-bureau` com injeção de latência (p95 ~1.2s) e erros (timeout/5xx) em fração configurável; `engine-score`/`engine-fraud` como mocks fixos (ref. D6).
- [ ] 1.2 Preparar seed realista da tabela de auditoria Oracle (~400M linhas) excluindo os `traceId` de teste (ref. D8, `mass-data-strategy`).
- [ ] 1.3 Implementar gerador de massa: `traceId` via `uuidv4`, `customerId` `cust-<VU>-<ITER>-<timestamp>`, ≥95% únicos + ~5% reuso deliberado tagueado à parte (ref. D8).
- [ ] 1.4 Instrumentar observabilidade no alvo: lag por partição, `oracle_pool_wait_ms`, `active_connections`, latência+timeout por downstream, taxa de fallback, in-flight/VT ativas (ref. D5/Risks).

## 2. E2E dos fluxos críticos (F1–F5)

- [ ] 2.1 E2E F3 aprovação direta: 3 motores OK → `APPROVED`, p95<1.5s (executor `per-vu-iterations`).
- [ ] 2.2 E2E F1 recusa por fraude: fraud=risco + score=bom → `DECLINED`.
- [ ] 2.3 E2E F2 fallback do bureau: timeout/5xx do bureau → `APPROVED_WITH_REVIEW`, não-5xx, dentro do SLA.
- [ ] 2.4 E2E F5 Kafka assíncrono: produzir em `requested`, verificar `completed` com mesmo `traceId`, `rate(completed/requested)==1`.
- [ ] 2.5 E2E F4 idempotência: reenviar `traceId` (simples e concorrente) → auditoria não duplica.

## 3. Carga e stress da API síncrona

- [ ] 3.1 Teste A (Load): `ramping-arrival-rate` 50→300 req/s (2m) + 300 sustentado (15m); thresholds `p(95)<1500`, `p(99)<2500`, `rate<0.005` (ref. D2/D3).
- [ ] 3.2 Teste B (Spike): 300→700 req/s em 30s + 5m no pico; medir SLA no pico; reportar degradação como decisão (pendência Q1).
- [ ] 3.3 Teste C (Stress): elevar além de 700 req/s até o knee point com Kafka concorrente; threshold afrouxado; registrar recurso que satura primeiro (pool/VT/downstream).

## 4. Pico Kafka e lag (T2, T4)

- [ ] 4.1 Teste D (Pico Kafka): `constant-arrival-rate` ~833 msg/s por 40m via `xk6-kafka`; thresholds `lag<180s`, `backlog<750k`, `drain<900s` (ref. D4).
- [ ] 4.2 Medir `C_max` (capacidade máxima de consumo, msg/s) e marcar as metas de backlog/drenagem como provisórias até confirmar (pendência Q2).
- [ ] 4.3 Validar lag fora do pico (~50 msg/s): `lag<10s` (SLO declarado).

## 5. Sobreposição e contenção do pool Oracle (T3)

- [ ] 5.1 Teste F (Sobreposição): rodar Teste A (300 req/s) + Teste D (833 msg/s) simultaneamente.
- [ ] 5.2 Medir `Δp95_api` = p95(sobreposição) − p95(solo); aceite `Δp95_api ≤ 150ms`; reportar `oracle_pool_wait_ms` e `active_connections` (ref. D5).
- [ ] 5.3 Registrar que testes solo são insuficientes para aceitar o pico (falso verde) e marcar extrapolação QA(4 pods)→prod(8 pods) como não-linear (pendência Q3).

## 6. Consolidação e pendências

- [ ] 6.1 Consolidar matriz de rastreabilidade teste→fluxo/tensão (D9) e os thresholds atingidos vs SLA.
- [ ] 6.2 Registrar pendências de decisão humana: Q1 (SLA no pico), Q2 (`C_max`), Q3 (QA→prod), Q4 (veredito multi-downstream), Q5 (atomicidade/constraint de idempotência), Q6 (fração de reuso de massa).
