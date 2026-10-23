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
| **OpenSpec** (A)   | A1          | A2                      | A3                                |
| **Spec do Kiro** (B)| B1         | B2                      | B3                                |

- **Linhas (processo):** OpenSpec (CLI + templates reais, via `openspec new change` →
  `instructions` → `validate --strict`) vs o modo spec do Kiro (`requirements.md` + `design.md`).
- **Colunas (técnica):**
  - **Single-role:** executor único, sem papéis distintos.
  - **Multi-role:** quatro papéis (`agents/*.md`) num **único contexto**, debate encenado.
  - **Multi-agente:** cada papel como **subagente com contexto isolado** (`invoke_sub_agent`);
    o Orchestrator delega, provoca a crítica cruzada e consolida.

> Nota de convenção: a estrutura de pastas usa **A = OpenSpec, B = Spec do Kiro**. Documentos
> em `analysis/` podem inferir a técnica pela leitura do próprio design e eventualmente rotular
> as famílias de forma diferente — ao cruzar, confie no nome da pasta da run.

## Alvo do teste

Aplicação fictícia **"Veridia"** (`scenario/target-app.md`): um gateway de decisão de crédito
(Kotlin/Spring, Virtual Threads, Kafka, Oracle) desenhado com **5 pontos de tensão deliberados**
(T1 SLA vs bureau instável; T2 pico Kafka 16x; T3 pool Oracle compartilhado; T4 meta de lag
vaga; T5 massa que mascara escrita). As tensões servem para diferenciar os métodos — um alvo
trivial faria todos convergirem igual.

## Artefatos por braço (`runs/<braço>/`)

| | "why/what" | "how" (o plano) | Registro de processo |
|---|---|---|---|
| **Braços A (OpenSpec)** | `proposal.md` | `design.md` | `process.md` |
| **Braços B (Spec Kiro)** | `requirements.md` | `design.md` | `process.md` |

- `design.md`: **onde o planejamento de testes mora** — é o artefato comum lido na avaliação.
- `process.md`: registro conciso do processo (propostas por papel, discussão, resolução, papel
  do Orquestrador, proxies de esforço e custo/tempo reais). Segue `templates/output-format.md`.

## Metodologia de execução (rodada definitiva)

Para que os seis braços sejam comparáveis, todos rodaram na **mesma condição**:

- **Uma sessão limpa por braço** (chat/janela novo), lendo apenas `scenario/`, `agents/`,
  `templates/output-format.md` e `HOW-TO-RUN.md` — **sem** olhar `runs/` de outros braços, para
  não contaminar. Prompts prontos em `prompts-sessao-limpa/`.
- **Janela remota WSL + Autopilot ligado** nas sessões de execução (o Autopilot não é
  pré-requisito dos subagentes — testado — mas foi padronizado para uniformidade).
- **OpenSpec real** nos braços A (CLI + `validate --strict`), nunca simulado.
- **Custo e tempo reais** coletados da interface (Est. Credits Used / Elapsed time) e
  registrados no `process.md` — o executor não os estima.

## Avaliação

A avaliação dos seis `design.md` é feita de forma **independente e por mais de um avaliador**,
para atacar o viés de autoavaliação. Fica em `analysis/`, um subdiretório por avaliador:

- `analysis/kiro/` — avaliação pelo Kiro.
- `analysis/kiro-auto/` — avaliação pelo Kiro em modo auto.
- `analysis/copilot/` — avaliação por um terceiro (Copilot).

Cada um contém `checklist-*.md`, `rubric-*.md` e `conclusoes-*.md`. Instrumentos:

1. **Checklist objetivo** — 8 critérios Sim/Não lidos do `design.md`. Mede presença, não
   profundidade; **satura** (tende a 8/8 em todos) — serve de porta de entrada, não de ranking.
2. **Rubrica numérica** (0–5 por critério, peso, máx 70) — discrimina profundidade; o peso
   recai em C2 (tensões) e C6 (riscos de medição), onde os braços se separam.
3. **Proxies de esforço + custo real** — subagentes, chamadas, tamanho; crédito/tempo da
   interface.

Templates em branco para novas rodadas: `templates/rubric-template.md`,
`templates/checklist-template.md`, `templates/output-format.md`.

## Alcance e limites (honestidade experimental)

- Mede o efeito do **processo** e da **técnica**, *dado o Kiro como executor* — não compara
  modelos de IA diferentes.
- **N=1 cenário, 1 rodada por braço:** estudo de caso, não estatística. Diferenças pequenas na
  rubrica (1–2 pontos) estão dentro do ruído; confie em diferenças de família e no **custo**.
- **Isolamento dos subagentes (A3/B3): declarado, não verificado** — o mecanismo roda, mas não
  há prova de contexto genuinamente separado.
- **Tempo é ruidoso** (inclui espera entre turnos); o **crédito** é a métrica de custo confiável.

## Estrutura do repositório

```
single-multi-role-test/
  README.md  HOW-TO-RUN.md
  scenario/              scenario.md  context-base.md  target-app.md
  agents/                orchestrator.md  qa-planner.md  qa-performance-engineer.md  backend-integration-engineer.md
  prompts-sessao-limpa/  A1..A3-openspec-*.md  B1..B3-kiro-*.md   (um prompt por braço)
  runs/                  A1..A3 (proposal+design+process)  B1..B3 (requirements+design+process)
  openspec/              config.yaml  changes/  specs/   (gerados pelo CLI nas runs A)
  analysis/              kiro/  kiro-auto/  copilot/   (checklist + rubric + conclusoes por avaliador)
  templates/             output-format.md  rubric-template.md  checklist-template.md
```
