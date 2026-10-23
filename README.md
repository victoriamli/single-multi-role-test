# single-multi-role-test

Experimento controlado para comparar **como o Kiro planeja testes** sob diferentes
combinações de **processo** e **técnica de prompting**.

## Pergunta do experimento

> Usando o Kiro como executor, qual combinação de **processo** (OpenSpec vs modo spec do
> Kiro / agentes próprios) e **técnica** (single-role / multi-role / multi-agente) produz o
> melhor **planejamento de testes K6** (E2E, carga, volume e stress) para uma aplicação?

O produto de cada braço é um **planejamento de testes** (não há implementação de código nem
execução real). Comparamos os planos entre si. O planejamento mora no `design.md` de cada run.

## Matriz 2×3

| Processo \ Técnica | Single-role | Multi-role (1 contexto) | Multi-agente (contextos isolados) |
|--------------------|-------------|-------------------------|-----------------------------------|
| **OpenSpec**       | A1          | A2                      | A3                                |
| **Spec do Kiro**   | B1          | B2                      | B3                                |

- **Linhas (processo):** OpenSpec (CLI + templates reais) vs o modo spec do Kiro / agentes
  próprios (`agents/*.md`). Em ambos o executor é o Kiro.
- **Colunas (técnica):**
  - **Single-role:** sem papéis distintos.
  - **Multi-role:** vários papéis num **único contexto** (debate encenado).
  - **Multi-agente:** cada papel como **subagente com contexto isolado**; o Orchestrator
    coordena. (Depende do modo de execução — ver HOW-TO-RUN.)

## Artefatos por braço

Cada run produz três documentos, respeitando os nomes nativos de cada processo:

| | Documento "why/what" | Documento "how" (o plano) | Registro de processo |
|---|---|---|---|
| **Braços A (OpenSpec)** | `proposal.md` | `design.md` | `process.md` |
| **Braços B (spec Kiro)** | `requirements.md` | `design.md` | `process.md` |

- `proposal.md` / `requirements.md`: gerados pelo processo (OpenSpec gera de verdade via CLI;
  o spec do Kiro gera requirements+design).
- `design.md`: **onde o planejamento de testes mora** — é o artefato comum lido na avaliação.
- `process.md`: registro conciso do processo (propostas por papel, discussão, resolução,
  papel do Orquestrador, proxies de esforço). Segue `output-format.md`.

## Avaliação (três camadas)

1. **Checklist objetivo** (`rubric/checklist.md`) — 8 critérios de alto nível, Sim/Não,
   preenchidos lendo o `design.md`. É a camada principal.
2. **Rubrica numérica** (`rubric/rubric.md`) — mantida pela tabela visual; secundária.
3. **Proxies de esforço** — nº de subagentes, chamadas, tamanho dos artefatos (no `process.md`).
   Tokens/tempo reais só se coletados por instrumentação (não estimados).

## Alcance e limites (honestidade experimental)

- Mede: efeito do **processo** e da **técnica**, *dado o Kiro como executor*.
- NÃO mede: diferença entre modelos de IA.
- N pequeno é estudo de caso, não estatística. Avaliação idealmente **cega** (quem pontua não
  sabe qual braço gerou o design).

## Estrutura

```
single-multi-role-test/
  README.md  HOW-TO-RUN.md  output-format.md
  scenario/     scenario.md  context-base.md  target-app.md
  agents/       orchestrator.md  qa-planner.md  qa-performance-engineer.md  backend-integration-engineer.md
  rubric/       checklist.md (principal)  rubric.md (secundária)
  runs/         A1.. A2.. A3.. B1.. B2.. B3..   (cada um: proposal|requirements + design + process)
  openspec/     config.yaml  (changes/ e specs/ gerados nas runs A)
  old/          resultados arquivados das rodadas anteriores
```
