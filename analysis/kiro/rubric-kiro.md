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

> Pontuado por: Kiro. **Não-cego:** cada design revela a própria técnica no texto (menção a
> "subagentes isolados", "debate multi-role", "single-role", OpenSpec/`requirements.md`). Peso
> aplicado na linha. Nota bruta 0–5 à esquerda; abaixo, o total já ponderado.

Legenda de braços (inferida do texto de cada design):
A1 = single-role, agentes próprios · A2 = multi-role (debate em contexto único) ·
A3 = multi-agente isolado + OpenSpec · B1 = Spec single-role · B2 = Spec multi-role ·
B3 = Spec multi-agente.

| Critério | A1 | A2 | A3 | B1 | B2 | B3 |
|----------|----|----|----|----|----|----|
| C1 ×2    | 4  | 4  | 5  | 4  | 4  | 5  |
| C2 ×3    | 4  | 4  | 5  | 3  | 4  | 5  |
| C3 ×2    | 4  | 4  | 5  | 4  | 4  | 5  |
| C4 ×2    | 4  | 4  | 4  | 4  | 4  | 5  |
| C5 ×1    | 4  | 4  | 5  | 4  | 4  | 5  |
| C6 ×2    | 3  | 4  | 4  | 3  | 4  | 5  |
| C7 ×1    | 4  | 4  | 4  | 4  | 4  | 5  |
| C8 ×1    | 5  | 4  | 4  | 4  | 4  | 4  |
| **Total**| **53** | **55** | **63** | **49** | **56** | **68** |

Conta dos totais (peso × nota, somado):

- **A1 = 53:** C1 8 + C2 12 + C3 8 + C4 8 + C5 4 + C6 6 + C7 4 + C8 5.
- **A2 = 55:** C1 8 + C2 12 + C3 8 + C4 8 + C5 4 + C6 8 + C7 4 + C8 4.
- **A3 = 63:** C1 10 + C2 15 + C3 10 + C4 8 + C5 5 + C6 8 + C7 4 + C8 4.
- **B1 = 49:** C1 8 + C2 9 + C3 8 + C4 8 + C5 4 + C6 6 + C7 4 + C8 4.
- **B2 = 56:** C1 8 + C2 12 + C3 8 + C4 8 + C5 4 + C6 8 + C7 4 + C8 4.
- **B3 = 68:** C1 10 + C2 15 + C3 10 + C4 10 + C5 5 + C6 10 + C7 5 + C8 4.

Normalizado (sobre 70): A1 76% · A2 79% · A3 90% · B1 70% · B2 80% · B3 97%.

## Métricas secundárias (registrar, não pontuar)

Tempo e tokens informados pelo usuário (tokens em unidade relativa fornecida). Tamanho medido
por leitura do `design.md` de cada braço.

| Braço | Tokens | Tempo | Tamanho aprox. | Nº testes na matriz | Total rubrica |
|-------|--------|-------|----------------|---------------------|---------------|
| A1 | 6.74 | 7m04s | médio | 10 (E2E-1..5 + 5 carga) | 53 |
| A2 | 6.87 | 7m15s | médio-grande | 12 | 55 |
| A3 | 12.73 | 12m22s | grande | 6 testes + transversais + rastreab. | 63 |
| B1 | 4.50 | 4m34s | médio | 9 | 49 |
| B2 | 4.63 | 4m37s | médio | 9 | 56 |
| B3 | 11.51 | 13m25s | grande | 7 (A–G) | 68 |

Observações qualitativas:

- **O isolamento/multi-agente agregou algo real.** B3 e A3 (contexto isolado por subagente)
  trouxeram itens que nenhum single-role viu: coordinated omission nomeado, pinning de Virtual
  Threads, HPA lag no início do spike, idempotência concorrente como teste próprio (B3-G),
  distinção bureau-timeout vs bureau-5xx (B3 E2E-3/E2E-4), e a decomposição de T4 em três
  números com cálculo de backlog.
- **Custo não é linear com qualidade.** Dobrar/triplicar tokens (A3, B3) subiu ~15–20 pontos
  percentuais de qualidade sobre o melhor barato (B2). O salto de B1→B2 (quase mesmo custo)
  rendeu +7 pontos só trocando single-role por multi-role.
- **B2 é o ponto doce de custo-benefício:** 80% da qualidade por ~1/3 dos tokens de B3.
- **A1 lidera C8 (clareza):** o plano mais enxuto é também o mais fácil de ler e executar de
  cabo a rabo — a matriz D8 é autossuficiente. Profundidade não é clareza.

## Perguntas de conclusão

1. **Multi-role superou single-role o suficiente?** Sim, e barato. A2 > A1 (+2) e, sobretudo,
   B2 > B1 (+7) com custo de tokens praticamente idêntico. A crítica cruzada encenada rende
   principalmente em C2 (tensões) e C6 (riscos de medição), que é onde o single-role é mais
   raso. É o upgrade de melhor relação custo-benefício do conjunto.
2. **Multi-agente (isolado) superou o multi-role?** Sim em qualidade absoluta (A3 > A2, B3 > B2),
   mas ao custo de ~2–3× os tokens e ~2–3× o tempo. O isolamento produziu crítica mais honesta:
   em vez de um Orchestrator encenando papéis que concordam, os subagentes trouxeram riscos
   genuinamente não óbvios. O ganho é real, porém marginal por token.
3. **OpenSpec vs agentes próprios?** Dentro da mesma técnica, OpenSpec (B) rendeu um pouco mais
   de estrutura útil que agentes próprios (A): B2 > A2, B3 > A3, puxado por rastreabilidade
   RQ-*→teste e separação de pré-condições de ambiente. A exceção é single-role, onde OpenSpec
   não ajudou (B1 < A1): a estrutura extra vira cerimônia sem um segundo papel que a explore.
4. **Onde o esforço extra NÃO compensou?** (a) B1: OpenSpec sozinho, sem segundo papel, custou
   organização sem elevar profundidade — ficou abaixo de A1, que é mais barato. (b) O salto
   multi-role→multi-agente (A2→A3, B2→B3): +8 a +12 pontos por ~3× tokens; vale para o plano de
   referência, não para rodar sempre. (c) Nenhum braço converteu seu custo em mais clareza — a
   acionabilidade (C8) saturou cedo e o braço mais caro (B3) nem lidera nela.
