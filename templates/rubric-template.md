# Rubrica de avaliação (template em branco)

Cópia zerada para reuso em outra rodada/cenário. Cada critério vale **0–5**. Pontue cada braço
de forma independente, idealmente **cego** (sem saber qual método gerou qual plano).

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
| C1 ×2    |    |    |    |    |    |    |
| C2 ×3    |    |    |    |    |    |    |
| C3 ×2    |    |    |    |    |    |    |
| C4 ×2    |    |    |    |    |    |    |
| C5 ×1    |    |    |    |    |    |    |
| C6 ×2    |    |    |    |    |    |    |
| C7 ×1    |    |    |    |    |    |    |
| C8 ×1    |    |    |    |    |    |    |
| **Total**|    |    |    |    |    |    |

## Perguntas de conclusão

1. O multi-role superou o single-role o suficiente para justificar o esforço?
2. O multi-agente (contexto isolado) superou o multi-role? A crítica cruzada foi mais honesta?
3. OpenSpec vs agentes próprios: qual deu mais estrutura útil ao plano?
4. Onde o esforço extra NÃO compensou?
