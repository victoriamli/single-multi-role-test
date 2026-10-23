# Checklist de qualidade do planejamento (objetivo)

Avaliação factual e de alto nível, preenchida lendo o **`design.md`** de cada braço (é onde o
plano de testes mora, tanto nos braços OpenSpec quanto nos do modo spec do Kiro).

Marcação **Sim / Não**. Se houver ressalva, anotar em **Obs.** de forma concisa. O critério é:
o item está presente e tratado no design? — não "quão bem", para evitar subjetividade.

## Os 8 critérios

1. **Cobriu os fluxos críticos do alvo?** Os fluxos de negócio essenciais foram contemplados.
2. **Cobriu os tipos de teste relevantes?** Os tipos aplicáveis (carga, stress, volume, etc.)
   foram escolhidos e os descartados, justificados.
3. **Critérios de aceite mensuráveis?** Metas numéricas e verificáveis, não vagas.
4. **Identificou os principais riscos/gargalos do alvo?** Os pontos de ruptura foram nomeados.
5. **Tratou a estratégia de dados de massa?** Como gerar/obter a massa e seu impacto.
6. **Apontou riscos de medição?** Onde o teste poderia medir a coisa errada / dar falso resultado.
7. **Levantou o que precisa de decisão humana?** Pendências/ambiguidades explicitadas.
8. **O plano é acionável?** Um QA conseguiria executar a partir do design.

## Planilha (preencher após cada run)

> Preenchimento por **autoavaliação do executor** (Kiro) lendo o `design.md`. Não é cego.

| # | Critério | A1 | A2 | A3 | B1 | B2 | B3 |
|---|----------|----|----|----|----|----|----|
| 1 | Fluxos críticos         | Sim | Sim | Sim | Sim | Sim | Sim |
| 2 | Tipos de teste          | Sim | Sim | Sim | Sim | Sim | Sim |
| 3 | Aceite mensurável       | Sim | Sim | Sim | Sim | Sim | Sim |
| 4 | Riscos/gargalos         | Sim | Sim | Sim | Sim | Sim | Sim |
| 5 | Dados de massa          | Sim | Sim | Sim | Sim | Sim | Sim |
| 6 | Riscos de medição       | Sim | Sim | Sim | Sim | Sim | Sim |
| 7 | Decisão humana          | Sim | Sim | Sim | Sim | Sim | Sim |
| 8 | Acionável               | Sim | Sim | Sim | Sim | Sim | Sim |
| **Sim (total)** |     | 8/8 | 8/8 | 8/8 | 8/8 | 8/8 | 8/8 |

## Observações por braço

<!-- Anotar aqui ressalvas concisas quando um Sim/Não precisar de contexto. -->

**A1 (OpenSpec single-role) — autoavaliação do executor, não cega:**
1. Fluxos: E2E-1..5 cobrem os 5 fluxos (aprovação, fraude, fallback bureau, Kafka async, idempotência).
2. Tipos: D1 traz tabela escolhendo E2E/load/stress/spike/volume e justifica soak como parcial.
3. Aceite: thresholds numéricos ancorados no SLA (p95<1.5s, p99<2.5s, 5xx<0.5%, lag≤5min).
4. Riscos: bureau instável (T1) e pool Oracle compartilhado (T3) nomeados como gargalos dominantes; ST-1 busca o knee point.
5. Massa: D6 define traceId/customerId únicos + fração repetida p/ idempotência (T5).
6. Medição: isolado vs. concorrente (T3/C-1), QA(4 pods) vs. prod(8), seed de 400M como pré-condição de V-1/C-1.
7. Decisão humana: Open Questions lista T4, retention Kafka, escala QA→prod, observabilidade do pool.
8. Acionável: matriz D8 + tasks.md por teste (tipo, carga, threshold, aceite, massa).
   Ressalva: thresholds dependem de stub do bureau e seed de volume disponíveis em QA.

**A2 (OpenSpec multi-role) — autoavaliação do executor, não cega:**
1. Fluxos: specs `e2e-critical-flows` (5 requisitos) + E2E-1..5 na matriz D8 cobrem os 5 fluxos.
2. Tipos: D1 escolhe E2E/load/stress/spike/volume e justifica soak como opcional (SK-1).
3. Aceite: thresholds numéricos ancorados no SLA (p95<1.5s, p99<2.5s, 5xx<0.5%, lag≤5min).
4. Riscos: bureau instável (T1) e pool Oracle compartilhado (T3) nomeados; ST-1 busca o knee point.
5. Massa: spec `mass-data-strategy` + D7 — unicidade por padrão, repetição só na idempotência (T5).
6. Medição: D5/CT-1 isolado vs concorrente (Δp95), QA 4 pods vs prod 8, observabilidade do pool.
7. Decisão humana: Open Questions lista T4, extrapolação QA→prod, partições Kafka, seed 400M, observabilidade.
8. Acionável: matriz D8 (12 testes) + tasks.md agrupadas com pré-condições e pendências.
   Ressalva: igual a A1 — depende de stub do bureau com injeção e seed de 400M em QA (marcados como pré-condição).

**A3 (OpenSpec multi-agente) — autoavaliação do executor, não cega:**
1. Fluxos: spec `e2e-critical-flows` (F1–F5) + matriz D7/D9 cobrem os 5 fluxos (aprovação, fraude, fallback bureau, Kafka async, idempotência), com asserção de veredito.
2. Tipos: D1 traz tabela escolhendo E2E/load/spike/stress/volume/soak e justifica fusões (volume Kafka com soak; stress isolado despriorizado p/ sobreposição).
3. Aceite: thresholds numéricos ancorados no SLA (p95<1500, p99<2500, 5xx<0.005; lag<180s, backlog<750k, drain<900s; Δp95≤150ms).
4. Riscos: bureau (#1), Virtual Threads (#2) e pool Oracle (#3) nomeados como gargalos dominantes; Teste C busca o knee point.
5. Massa: spec `mass-data-strategy` + D8 — ≥95% únicos + 5% reuso tagueado, seed 400M excluindo traceIds de teste (T5).
6. Medição: D5/Teste F sobreposição vs solo (Δp95), falso verde nomeado, QA(80)→prod(160) não-linear.
7. Decisão humana: Open Questions Q1–Q6 (SLA no pico, C_max, QA→prod, veredito multi-downstream, atomicidade idempotência, fração de reuso).
8. Acionável: matriz D9 + tasks.md por grupo com pré-condições e pendências.
   Ressalva: igual a A1/A2 — depende de mock do bureau com injeção e seed de 400M em QA (marcados como pré-condição). `C_max` é bloqueante para as metas de lag (marcado como pendência).

**B1 (Spec do Kiro single-role) — autoavaliação do executor, não cega:**
1. Fluxos: E2E-1..5 cobrem os 5 fluxos (aprovação F1, fraude F2, fallback bureau F3, Kafka async F4, idempotência F5) com asserção de veredito, não só status HTTP.
2. Tipos: D1 traz tabela escolhendo E2E/load/stress/spike/soak/volume; soak "puro" e stress isolado do bureau despriorizados com justificativa (nada integralmente descartado pois o alvo exercita todos).
3. Aceite: D4 com thresholds numéricos ancorados no SLA (p95<1500ms, p99<2500ms, 5xx<0.005, avail>0.999, lag<10s normal, lag pico<5min).
4. Riscos: bureau instável (T1) e pool Oracle compartilhado (T3) nomeados como gargalos dominantes; ST-1 busca o knee point; Virtual Threads movem o limite para o pool/bureau.
5. Massa: D6 define traceId/customerId únicos + fração marcada repetida para idempotência + seed 400M em namespace separado (T5, RQ-DATA-1..3).
6. Medição: D5 lista 6 riscos — falso verde por closed model (T1), acoplamento escondido por teste isolado (T3/CT-1), massa que mascara escrita (T5), volume ausente, extrapolação QA(4)→prod(8), fidelidade do stub do bureau.
7. Decisão humana: D7 lista T4 (alvo de lag), ambiente QA com volume/stub, observabilidade do pool, capacidade-alvo máxima, retenção da auditoria.
8. Acionável: matriz D8 por teste (tipo, executor/carga, threshold-chave, fluxo/tensão, requisito) + ordem de execução sugerida.
   Ressalva: thresholds dependem de stub do bureau com injeção de latência/erro e de seed de 400M disponíveis em QA (marcados como pré-condição/pendência); T4=5min é proposta a confirmar.

**B2 (Spec do Kiro multi-role) — autoavaliação do executor, não cega:**
1. Fluxos: E2E-1..5 (D3) cobrem os 5 fluxos (aprovação F1, fraude F2, fallback bureau F3, Kafka async F4, idempotência F5) com asserção de veredito, não só HTTP; E2E também rodam como checks sob carga.
2. Tipos: D1 escolhe E2E/load/spike/volume/stress e descarta soak puro com justificativa (sem SLA de duração; o pico de 40 min age como soak parcial). Nada essencial omitido.
3. Aceite: D4 com thresholds numéricos ancorados no SLA (p95<1500, p99<2500, 5xx<0.005, avail>0.999, lag<10s baseline, lag pico<180s, dreno<900s).
4. Riscos: bureau instável (T1) e pool Oracle compartilhado (T3) nomeados como gargalos dominantes em D0/D7; ST-1 busca o knee point; Virtual Threads movem o limite p/ pool/downstream.
5. Massa: D6 define unicidade ≥95% + fatia de ~5% reuso tagueada + seed de 400M excluindo traceIds de teste (T5, RQ-DATA-1..3).
6. Medição: D7 nomeia 7 riscos — falso verde por modelo fechado (T1), acoplamento escondido por teste isolado (T3/CT-1), massa que mascara escrita (T5), stub fixo, extrapolação QA(4)→prod(8) não-linear, lag medido só pelo K6, volume ausente no índice.
7. Decisão humana: D9 lista P1–P5 (T4 numérico, capacidade-alvo, stub+seed em QA bloqueantes, fator QA→prod, retenção auditoria).
8. Acionável: matriz D8 por teste (tipo, executor/carga, threshold-chave, fluxo/tensão, requisito) + ordem de execução sugerida.
   Ressalva: thresholds dependem de stub do bureau injetável e seed de 400M em QA (marcados como pré-condição/pendência P3); T4 (180s/900s) é proposta a confirmar (P1). Paridade substantiva com A2 (ambos multi-role).

**B3 (Spec do Kiro multi-agente) — autoavaliação do executor, não cega:**
1. Fluxos: E2E-1..7 (D3) cobrem os 5 fluxos (aprovação F1, fraude F2, fallback bureau F3, Kafka async F4, idempotência F5) com asserção de veredito, não só HTTP; E2E também rodam como checks sob carga nos Testes B/E/F.
2. Tipos: D1 escolhe E2E/load/stress/spike e funde soak+volume nos cenários que os tornam significativos (soak dentro do pico Kafka; volume como pré-condição de dados 400M). Nada essencial omitido, fusões justificadas.
3. Aceite: D4 com thresholds numéricos ancorados no SLA (p95<1500, p99<2500, 5xx<0.005, dropped<1; lag<900000ms, dreno<600000ms, perda==0).
4. Riscos: D0 nomeia os gargalos dominantes — bureau (#1), pool Oracle 20/pod sob Virtual Threads (#2), folga estrutural do SLA (#3), teto do consumer (#4), tabela 400M (#5), fallback silencioso (#6); Teste C busca o knee point.
5. Massa: D6 define unicidade por padrão + fração ~1–2% reuso tagueada e excluída dos thresholds de SLA + seed de 400M não-colidente (T5, RQ-DATA-1..3).
6. Medição: D5 nomeia 9 riscos — falso verde por modelo fechado (T1), acoplamento escondido por teste isolado (T3/Teste F + Δp95), massa que mascara escrita (T5), mock fixo, tabela vazia, extrapolação QA(4)→prod(8) não-linear, fallback silencioso, pinning de VT, HPA lag no spike.
7. Decisão humana: requirements §7 (RQ-OPEN-1..8) + D10 listam T4 numérico, SLA do fallback, teto de fallback, janela de disponibilidade/5xx, paridade sync/async, escopo de idempotência, C_max bloqueante, seed+stub em QA.
8. Acionável: matriz D8 (teste×carga×threshold×fluxo/tensão×requisito) + ordem de execução D9 + executores/estágios concretos.
   Ressalva: thresholds dependem de stub injetável do bureau e seed de 400M em QA (marcados como pré-condição/pendência bloqueante); T4 (15/10/5 min) é proposta a ratificar; C_max é bloqueante para stress e metas de lag. **Isolamento dos subagentes: declarado, não verificado** (ver process.md §4) — se tivesse ficado claro que não houve isolamento real, B3 degradaria para multi-role; nenhum sinal disso foi observado. Paridade substantiva com B2/A3 (mesma cobertura de plano); diferença é de processo (crítica cruzada entre subagentes isolados por caminhos independentes), não de conteúdo do design.
