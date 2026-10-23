# Checklist de qualidade do planejamento — avaliação preenchida

Avaliação factual de presença dos itens (Sim/Não) nos seis designs (`designA1.md` a `designB3.md`).
Critério: o item está presente e tratado no plano, independentemente de profundidade.

## Os 8 critérios

1. **Cobriu os fluxos críticos do alvo?**
2. **Cobriu os tipos de teste relevantes?**
3. **Critérios de aceite mensuráveis?**
4. **Identificou os principais riscos/gargalos do alvo?**
5. **Tratou a estratégia de dados de massa?**
6. **Apontou riscos de medição?**
7. **Levantou o que precisa de decisão humana?**
8. **O plano é acionável?**

## Planilha

> Preenchido em 2026-10-23 (avaliação não cega).

| # | Critério | A1 | A2 | A3 | B1 | B2 | B3 | Obs. |
|---|----------|----|----|----|----|----|----|------|
| 1 | Fluxos críticos | Sim | Sim | Sim | Sim | Sim | Sim | Todos cobrem F1-F5; A3/B3 expandem com cenários extras (ex.: sync vs async). |
| 2 | Tipos de teste | Sim | Sim | Sim | Sim | Sim | Sim | Todos cobrem E2E/load/stress/spike/volume; soak é dedicado em alguns e fundido em outros. |
| 3 | Aceite mensurável | Sim | Sim | Sim | Sim | Sim | Sim | Todos têm thresholds numéricos de SLA e metas de lag/dreno no pico Kafka. |
| 4 | Riscos/gargalos | Sim | Sim | Sim | Sim | Sim | Sim | T1-T5 aparecem em todos; A3/B3 detalham melhor o modelo de gargalos. |
| 5 | Dados de massa | Sim | Sim | Sim | Sim | Sim | Sim | Consenso: alta unicidade + fração controlada de reuso para idempotência. |
| 6 | Riscos de medição | Sim | Sim | Sim | Sim | Sim | Sim | Todos discutem falso-verde e acoplamento; B3 explicita guarda anti-omission. |
| 7 | Decisão humana | Sim | Sim | Sim | Sim | Sim | Sim | Todos listam pendências abertas (threshold de lag, C_max, extrapolacao QA->prod, etc.). |
| 8 | Acionável | Sim | Sim | Sim | Sim | Sim | Sim | Todos trazem matriz de testes/executores/ordem de execução. |
| **Sim (total)** |  | **8/8** | **8/8** | **8/8** | **8/8** | **8/8** | **8/8** | Checklist saturado (esperado para critério de presenca). |

## Observações por braço

- **A1:** muito acionável e organizado; boa matriz. Menos enfático em guardas operacionais de carga (ex.: `dropped_iterations` como gate explícito).
- **A2:** mantém cobertura completa e boa rastreabilidade de debate multi-role; forte em acoplamento T3.
- **A3:** melhor profundidade no braço A (modelo mental + testes A-F + rastreabilidade); pendências continuam explícitas.
- **B1:** sólido e consistente com `RQ-*`; boa estrutura de execução, mas menos profundidade de gargalos que A3/B3.
- **B2:** equilíbrio entre estrutura de spec e pragmatismo; cobertura completa, sem grandes lacunas de presença.
- **B3:** plano mais robusto em rigor de medição e observabilidade (guardas explícitas, presets de bureau, contenção bloqueante).

## Nota de uso

O checklist mede **presença**, não profundidade. Para ranquear qualidade técnica entre designs,
usar a rubrica ponderada em `copilot/rubric-copilot.md`.
