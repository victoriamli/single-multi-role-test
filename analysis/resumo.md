# Resumo das avaliações

Consolidação das tabelas das três avaliações independentes dos seis planos (A1–A3, B1–B3).
Cada tabela é mantida **como foi montada na conclusão original**, com suas colunas próprias
(que diferem entre si). A tabela-síntese no final apenas alinha as notas de qualidade de cada
avaliador lado a lado.

> Convenção: **A = OpenSpec, B = Spec do Kiro / agentes próprios**, consistente nas três
> avaliações e com a estrutura de pastas das runs.

---

## 1. Kiro (auto-avaliação)

| Braço | Processo | Técnica | Checklist | Rubrica (autoaval.) | Créditos | Tempo | Subagentes |
|-------|----------|---------|:---------:|:-------------------:|---------:|------:|:----------:|
| A1 | OpenSpec  | single-role  | 8/8 | 67 | 6.74  | 7m04  | 0 |
| A2 | OpenSpec  | multi-role   | 8/8 | 69 | 6.87  | 7m15  | 0 |
| A3 | OpenSpec  | multi-agente | 8/8 | 69 | 12.73 | 12m22 | 3 |
| B1 | Spec Kiro | single-role  | 8/8 | 67 | 4.5   | 4m34  | 0 |
| B2 | Spec Kiro | multi-role   | 8/8 | 69 | 4.63  | 4m37  | 0 |
| B3 | Spec Kiro | multi-agente | 8/8 | 69 | 11.51 | 13m25 | 3 |

---

## 2. Copilot

| Design | Qualidade (0-70) | Tokens | Tempo | Qualidade/tokens | Qualidade/min |
|--------|-----------------:|-------:|------:|-----------------:|--------------:|
| A1 | 56 | 6.74  | 7m04  | 8.31  | 7.93  |
| A2 | 56 | 6.87  | 7m15  | 8.15  | 7.72  |
| A3 | 63 | 12.73 | 12m22 | 4.95  | 5.09  |
| B1 | 55 | 4.50  | 4m34  | 12.22 | 12.04 |
| B2 | 56 | 4.63  | 4m37  | 12.10 | 12.13 |
| B3 | 66 | 11.51 | 13m25 | 5.73  | 4.92  |

---

## 3. Kiro (avaliação em contexto separado)

| Braço | Família / colaboração | Rubrica | % (de 70) | Tokens | Tempo | Qualidade/token |
|-------|-----------------------|--------:|----------:|-------:|-------|----------------:|
| B3 | agentes · multi-agente  | 68 | 97% | 11.51 | 13m25s | 5.9      |
| A3 | OpenSpec · multi-agente | 63 | 90% | 12.73 | 12m22s | 4.9      |
| B2 | agentes · multi-role    | 56 | 80% | 4.63  | 4m37s  | **12.1** |
| A2 | OpenSpec · multi-role   | 55 | 79% | 6.87  | 7m15s  | 8.0      |
| A1 | OpenSpec · single-role  | 53 | 76% | 6.74  | 7m04s  | 7.9      |
| B1 | agentes · single-role   | 49 | 70% | 4.50  | 4m34s  | 10.9     |

---

## 4. Síntese das notas de qualidade (por avaliador)

Notas de qualidade de cada avaliador lado a lado. As colunas vêm, respectivamente, de:
"Rubrica (autoaval.)" da tabela 1, "Qualidade (0-70)" da tabela 2 e "Rubrica" da tabela 3.
Todas na escala 0–70.

| Braço | Kiro (auto-avaliação) | Copilot | Kiro (contexto separado) |
|-------|:---------------------:|:-------:|:------------------------:|
| A1 | 67 | 56 | 53 |
| A2 | 69 | 56 | 55 |
| A3 | 69 | 63 | 63 |
| B1 | 67 | 55 | 49 |
| B2 | 69 | 56 | 56 |
| B3 | 69 | 66 | 68 |
