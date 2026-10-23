# Rubrica de avaliação

Definida **antes** de rodar qualquer braço, para evitar viés na hora de pontuar. Cada
critério vale **0–5**. Pontue cada um dos seis planos (A1, A2, A3, B1, B2, B3) de forma
independente, idealmente sem saber qual braço gerou qual plano (avaliação cega).

## Critérios

| # | Critério | O que avaliar | Peso |
|---|----------|---------------|------|
| C1 | **Cobertura dos fluxos** | Os 5 fluxos críticos E2E foram contemplados? | ×2 |
| C2 | **Tratamento das tensões** | Quão bem T1–T5 foram identificadas e endereçadas? | ×3 |
| C3 | **Qualidade dos thresholds** | Metas numéricas, ancoradas nos SLAs, não vagas? | ×2 |
| C4 | **Modelagem de carga** | Executores/perfis K6 adequados e justificados? | ×2 |
| C5 | **Estratégia de massa** | Resolve T5 (unicidade vs idempotência)? | ×1 |
| C6 | **Riscos de medição** | Identifica medir a coisa errada (T3, isolamento)? | ×2 |
| C7 | **Realismo/viabilidade** | O plano é executável no ambiente descrito? | ×1 |
| C8 | **Clareza e acionabilidade** | Um QA conseguiria executar a partir do plano? | ×1 |

Pontuação máxima ponderada: **5 × (2+3+2+2+1+2+1+1) = 5 × 14 = 70**.

## Escala por critério

- **0** — ausente.
- **1–2** — presente mas superficial/genérico.
- **3** — adequado, cobre o essencial.
- **4** — forte, com justificativa e nuance.
- **5** — excelente; antecipa problemas não óbvios.

## Métricas secundárias (registrar, não pontuar)

- **Tempo** até o plano ficar pronto (percepção).
- **Tamanho** do plano (palavras / nº de testes propostos).
- **Esforço de orquestração** (nº de interações/subagentes no braço).
- **Observações qualitativas:** o debate agregou algo que o single-role não viu?

## Planilha de resultados (nova rodada)

> **Autoavaliação do executor** (Kiro), não cega. Secundária em relação ao checklist. Notas
> 0–5 por critério. Total = soma de nota×peso (máx 70).

| Critério | A1 | A2 | A3 | B1 | B2 | B3 |
|----------|----|----|----|----|----|----|
| C1 ×2    | 5  | 5  | 5  | 5  | 5  | 5  |
| C2 ×3    | 5  | 5  | 5  | 5  | 5  | 5  |
| C3 ×2    | 5  | 5  | 5  | 5  | 5  | 5  |
| C4 ×2    | 4  | 5  | 5  | 4  | 5  | 5  |
| C5 ×1    | 5  | 5  | 5  | 5  | 5  | 5  |
| C6 ×2    | 5  | 5  | 5  | 5  | 5  | 5  |
| C7 ×1    | 4  | 4  | 4  | 4  | 4  | 4  |
| C8 ×1    | 5  | 5  | 5  | 5  | 5  | 5  |
| **Total**| 67 | 69 | 69 | 67 | 69 | 69 |

> **A1 — justificativa da autoavaliação (executor, não cega):**
> C1=5 os 5 fluxos cobertos com aceite por fluxo. C2=5 T1–T5 todas endereçadas com decisão
> (T4 virou threshold testável de 5 min; T3 testado concorrente). C3=5 thresholds numéricos
> ancorados no SLA. C4=4 modelo aberto justificado e executor por teste, mas sem tuning fino
> de `preAllocatedVUs`/`maxVUs` por cenário. C5=5 unicidade + fração repetida medindo os dois
> caminhos. C6=5 isolado vs. concorrente, QA×prod, seed, modelo fechado. C7=4 executável, mas
> depende de stub do bureau e seed de 400M em QA (riscos honestamente marcados). C8=5 matriz
> D8 + tasks acionáveis.

> **A2 — justificativa da autoavaliação (executor, não cega):**
> C1=5 os 5 fluxos cobertos na spec `e2e-critical-flows` + E2E-1..5 com aceite por fluxo.
> C2=5 T1–T5 endereçadas com decisão explícita no debate D1–D8 (T4 virou lag ≤5min testável;
> T3 testado concorrente em CT-1). C3=5 thresholds numéricos ancorados no SLA. C4=5 executores
> de arrival-rate justificados por papel (D2: VUs presos mascarariam o SLA) e `preAllocatedVUs`/
> `maxVUs` + `dropped_iterations` tratados — ganho do multi-role sobre A1 neste critério.
> C5=5 unicidade + fração repetida medindo os dois caminhos (spec `mass-data-strategy`). C6=5
> isolado vs concorrente (Δp95), QA×prod, observabilidade do pool. C7=4 executável mas depende
> de stub do bureau e seed de 400M em QA (riscos honestamente marcados). C8=5 matriz D8 + tasks
> acionáveis. Diferença vs A1: o debate encenado expôs e justificou a escolha do executor
> (C4→5), sem alterar a cobertura substantiva do plano.

> **A3 — justificativa da autoavaliação (executor, não cega):**
> C1=5 os 5 fluxos cobertos na spec `e2e-critical-flows` + F1–F5 na matriz D7/D9 com asserção
> de veredito. C2=5 T1–T5 endereçadas com decisão explícita consolidada dos subagentes (T4
> virou lag<180s + backlog<750k + drain<900s; T3 sobreposição obrigatória em Teste F). C3=5
> thresholds numéricos ancorados no SLA. C4=5 executores de arrival-rate justificados (D2: VUs
> presos no closed-model mascarariam o SLA sob bureau lento) + dimensionamento por Little's Law
> (~390 in-flight) — paridade com A2 neste critério. C5=5 unicidade ≥95% + 5% reuso tagueado +
> seed 400M excluindo traceIds de teste (spec `mass-data-strategy`). C6=5 solo vs sobreposição
> (Δp95≤150ms), falso verde nomeado, QA(80)→prod(160) não-linear, observabilidade do pool.
> C7=4 executável mas depende de mock injetável do bureau, seed de 400M em QA e de medir
> `C_max` (bloqueante, marcado como pendência). C8=5 matriz D9 + tasks acionáveis por grupo.
> Diferença vs A2: a crítica cruzada entre subagentes isolados expôs as mesmas decisões-chave
> (open-model, sobreposição, T4 numérico) por caminhos independentes — convergência que reforça
> a confiança, sem alterar a cobertura substantiva. O custo de orquestração foi maior (3
> subagentes, mais turnos) para um plano de qualidade equivalente a A2.

> **B1 — justificativa da autoavaliação (executor, não cega):**
> C1=5 os 5 fluxos cobertos (E2E-1..5) com asserção de veredito e aceite por fluxo (F1/F3 sob
> SLA). C2=5 T1–T5 todas endereçadas com decisão: T1 via open model + ST-1/RQ-STRESS-3; T2 via
> SP-1 (perfil 2M/40min); T3 via CT-1 concorrente + Δp95; T4 virou threshold testável de 5 min
> (teto 15 min) com racional; T5 via D6 (unicidade + fração marcada + seed separado). C3=5
> thresholds numéricos ancorados no SLA em D4 (p95<1500, p99<2500, 5xx<0.005, lag). C4=4 open
> model justificado por Little's Law e estágios definidos por cenário, mas sem tuning fino de
> `preAllocatedVUs`/`maxVUs`/`dropped_iterations` por cenário — o single-role não encenou o
> papel de performance que forçaria esse detalhamento (mesma lacuna de A1). C5=5 unicidade por
> padrão + fração tagueada + seed de 400M em namespace separado, medindo os dois caminhos.
> C6=5 D5 nomeia 6 riscos de medição incl. falso verde por closed model, acoplamento escondido
> por teste isolado e extrapolação QA→prod não-linear. C7=4 executável mas depende de stub do
> bureau injetável e seed de 400M em QA (honestamente marcados como pré-condição/pendência).
> C8=5 matriz D8 + ordem de execução acionáveis. Paridade com A1 (ambos single-role): mesma
> cobertura substantiva e mesma lacuna em C4; o formato spec do Kiro (requirements EARS +
> design) deu estrutura equivalente à do OpenSpec para este alvo.

> **B2 — justificativa da autoavaliação (executor, não cega):**
> C1=5 os 5 fluxos cobertos (E2E-1..5) com asserção de veredito e aceite por fluxo (F1/F3 sob
> SLA), rodando também como checks sob carga. C2=5 T1–T5 endereçadas com decisão no debate: T1
> via modelo aberto + ST-1/ST-2/SP-2; T2 via SP-1 (perfil 50→833 msg/s) classificado como
> volume+spike; T3 via CT-1 concorrente + Δp95; T4 decomposto em lag<180s/backlog<750k/dreno<900s;
> T5 via D6 (unicidade + fatia marcada + seed separado). C3=5 thresholds numéricos ancorados no
> SLA em D4. C4=5 modelo aberto (arrival-rate) justificado explicitamente pela objeção do Backend
> (VUs fechados dão falso verde com bureau lento), com `preAllocatedVUs/maxVUs` por Little's Law e
> `dropped_iterations` — ganho do multi-role sobre B1, que não encenou o papel de performance com
> esse detalhe. C5=5 unicidade ≥95% + reuso tagueado medindo os dois caminhos. C6=5 D7 nomeia 7
> riscos de medição incl. falso verde por modelo fechado, acoplamento escondido (T3) e extrapolação
> QA→prod não-linear. C7=4 executável mas depende de stub do bureau injetável e seed de 400M em QA
> (honestamente marcados como pré-condição/pendência bloqueante P3). C8=5 matriz D8 + ordem de
> execução acionáveis. Paridade com A2 (ambos multi-role): o debate encenado expôs e justificou a
> escolha do executor aberto e o número de T4 (C4→5), sem alterar a cobertura substantiva; o
> formato spec do Kiro (requirements EARS + design) deu estrutura equivalente à do OpenSpec.
> Diferença vs B1 (single-role, 67): a crítica cruzada elevou C4 de 4→5 ao forçar a justificativa
> do modelo de carga.

> **B3 — justificativa da autoavaliação (executor, não cega):**
> C1=5 os 5 fluxos cobertos (E2E-1..7, D3) com asserção de veredito e aceite por fluxo (F1/F3
> sob SLA), rodando também como checks sob carga. C2=5 T1–T5 endereçadas com decisão consolidada
> dos subagentes: T1 via modelo aberto + Teste C/presets bureau-slow/flaky; T2 via Teste E
> (50→833 msg/s, spike+soak); T3 via Teste F combinado + Δp95 (bloqueante de release); T4
> decomposto em lag≤900000ms/dreno≤600000ms/alvo 300000ms; T5 via D6 (unicidade + fração tagueada
> + seed 400M). C3=5 thresholds numéricos ancorados no SLA em D4. C4=5 modelo aberto (arrival-rate)
> justificado explicitamente por coordinated omission + Little's Law (≈390 em voo → preAlloc 500/
> maxVUs 800) e `dropped_iterations<1` como guarda — paridade com A2/A3/B2, acima de B1 (single-role
> não forçou essa justificativa). C5=5 unicidade por padrão + ~1–2% reuso tagueado excluído dos
> thresholds + seed não-colidente. C6=5 D5 nomeia 9 riscos de medição incl. falso verde por modelo
> fechado (T1), acoplamento escondido por teste isolado (T3), extrapolação QA(4)→prod(8) não-linear,
> mock fixo, tabela vazia, fallback silencioso, pinning de VT. C7=4 executável mas depende de stub
> injetável do bureau, seed de 400M em QA e de medir C_max (bloqueantes, honestamente marcados).
> C8=5 matriz D8 + ordem de execução D9 acionáveis. Paridade com A3/B2: a crítica cruzada entre
> subagentes isolados chegou às mesmas decisões-chave (open-model, Teste F obrigatório, T4 numérico)
> por caminhos independentes — convergência que reforça a confiança sem alterar a cobertura
> substantiva. O **isolamento dos subagentes é declarado, não verificado** (process.md §4); nenhum
> sinal de degradação para multi-role foi observado. Custo de orquestração maior (3 subagentes, 4
> chamadas) para qualidade equivalente a B2 — mesmo padrão observado em A3 vs A2.

> Rodada de A anterior (A1/A2/A3) descartada: rodou em janela local lendo da WSL, o que
> contaminou a métrica de tempo. Toda a nova rodada roda em **janela remota WSL** com
> **Autopilot ligado**, condição uniforme para os seis braços.

## Perguntas de conclusão (responder após a nova rodada)

1. O multi-role superou o single-role o suficiente para justificar o esforço?
2. O multi-agente (contexto isolado) superou o multi-role? A crítica cruzada foi mais honesta?
3. OpenSpec vs agentes próprios: qual deu mais estrutura útil ao plano?
4. Onde o esforço extra NÃO compensou?
