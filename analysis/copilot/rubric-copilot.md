# Rubrica de avaliação — preenchida

Avaliação dos seis designs (`A1..B3`) com escala **0–5** por critério e peso ponderado.
Preenchimento em 2026-10-23 (avaliação não cega).

## Critérios

| # | Critério | O que avaliar | Peso |
|---|----------|---------------|------|
| C1 | **Cobertura dos fluxos** | Os fluxos críticos E2E foram contemplados? | ×2 |
| C2 | **Tratamento das tensões** | Quão bem as tensões do alvo foram identificadas e endereçadas? | ×3 |
| C3 | **Qualidade dos thresholds** | Metas numéricas, ancoradas nos SLAs, não vagas? | ×2 |
| C4 | **Modelagem de carga** | Executores/perfis K6 adequados e justificados? | ×2 |
| C5 | **Estratégia de massa** | Resolve a tensão de dados (unicidade vs idempotência)? | ×1 |
| C6 | **Riscos de medição** | Identifica medir a coisa errada (isolamento, falso-verde)? | ×2 |
| C7 | **Realismo/viabilidade** | O plano é executável no ambiente descrito? | ×1 |
| C8 | **Clareza e acionabilidade** | Um QA conseguiria executar a partir do plano? | ×1 |

Pontuação máxima ponderada: **5 × (2+3+2+2+1+2+1+1) = 5 × 14 = 70**.

## Escala por critério

- **0** — ausente.
- **1–2** — presente mas superficial/genérico.
- **3** — adequado, cobre o essencial.
- **4** — forte, com justificativa e nuance.
- **5** — excelente; antecipa problemas não óbvios.

## Notas por critério (resumo)

- **C1 (Cobertura dos fluxos):** todos cobrem F1-F5; A3/B3 adicionam variações úteis (sync vs async, idempotência concorrente).
- **C2 (Tratamento das tensões):** A3/B3 mais fortes por explicitar modelo causal de T1-T5 e decisão de contenção obrigatória.
- **C3 (Thresholds):** todos são numéricos; A3/B3 têm melhor decomposição de T4 e guardas operacionais.
- **C4 (Modelagem de carga):** consenso por open model; B3 se destaca por tratar `dropped_iterations` como falha de teste.
- **C5 (Massa):** todos bons em unicidade + reuso controlado; nenhum detalha cleanup/rollback pós-teste.
- **C6 (Riscos de medição):** todos citam falso-verde e acoplamento; A3/B3 mais completos.
- **C7 (Viabilidade):** todos executáveis em QA com pré-condições; B3 explicita dependências/bloqueios com mais clareza.
- **C8 (Clareza/acionabilidade):** A3/B3 com maior legibilidade (matriz A-G + ordem de execução forte).

## Métricas secundárias (registrar, não pontuar)

- **Tempo** até o plano ficar pronto.
- **Tamanho** do plano (linhas / nº de testes propostos).
- **Esforço** (nº de subagentes, chamadas de ferramenta, turnos).
- **Observações qualitativas:** o debate/isolamento agregou algo que o single-role não viu?

## Planilha de resultados

> Indicar quem pontuou e se foi cego. (Autoavaliação do executor tem viés; preferir avaliação
> cega por terceiro.)

| Critério | A1 | A2 | A3 | B1 | B2 | B3 |
|----------|----|----|----|----|----|----|
| C1 ×2    | 4 | 4 | 5 | 4 | 4 | 5 |
| C2 ×3    | 4 | 4 | 5 | 3 | 4 | 5 |
| C3 ×2    | 4 | 4 | 5 | 4 | 4 | 5 |
| C4 ×2    | 4 | 4 | 5 | 4 | 4 | 5 |
| C5 ×1    | 4 | 4 | 4 | 4 | 4 | 4 |
| C6 ×2    | 4 | 4 | 5 | 4 | 4 | 5 |
| C7 ×1    | 4 | 4 | 4 | 4 | 4 | 5 |
| C8 ×1    | 4 | 4 | 5 | 4 | 4 | 5 |
| **Total ponderado (0-70)** | **56** | **56** | **63** | **55** | **56** | **66** |
| **Percentual** | **80%** | **80%** | **90%** | **79%** | **80%** | **94%** |

## Perguntas de conclusão

1. O multi-role superou o single-role o suficiente para justificar o esforço?
2. O multi-agente (contexto isolado) superou o multi-role? A crítica cruzada foi mais honesta?
3. OpenSpec vs agentes próprios: qual deu mais estrutura útil ao plano?
4. Onde o esforço extra NÃO compensou?

## Respostas desta rodada

1. **Multi-role vs single-role:** ganho moderado, não decisivo por si. A1≈A2 e B1≈B2 em nota total.
2. **Multi-agente vs multi-role:** ganho claro. A3>B2/A2 e B3>B2/B1 por maior rigor em gargalos e medição.
3. **OpenSpec vs agentes próprios:** OpenSpec (braço B) trouxe melhor rastreabilidade (`RQ-*`), enquanto A3 trouxe ótima profundidade técnica sem spec formal.
4. **Onde o esforço extra não compensou:** estratégia de massa e viabilidade base ficaram parecidas entre os seis; quase todos convergiram em boas práticas similares.

