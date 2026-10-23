# Como executar os seis braços

Executor em **todos**: Kiro. Mesmo `scenario/` para todos. Cada braço produz, na sua pasta
`runs/<braço>/`:

- **Braços A (OpenSpec):** `proposal.md` + `design.md` + `process.md`
- **Braços B (spec do Kiro):** `requirements.md` + `design.md` + `process.md`

O `design.md` é o artefato que contém o **planejamento de testes** e é o que a avaliação lê.
O `process.md` segue `output-format.md` e registra o processo (não o plano).

## Regras comuns (valem para os 6)

1. Usar **somente** o contexto do experimento: `scenario/scenario.md`,
   `scenario/context-base.md`, `scenario/target-app.md`. Ignorar steering de outros projetos.
2. Não implementar scripts K6; foco em planejamento (que vive no `design.md`).
3. Ordem de execução **aleatória** entre os braços, para reduzir viés de ordem.
4. Avaliação idealmente **cega**: quem preenche o checklist/rubrica não sabe qual braço gerou
   o design.

## Protocolo desta rodada (devagar, um braço por vez)

- Rodar **um braço por vez**; parar e validar com o usuário antes do próximo.
- Preservar o raciocínio de forma concisa no `process.md` (resumo de propostas, discussão,
  resolução — não transcrição palavra a palavra).
- Registrar os proxies de esforço factuais no `process.md`.

## Braços A — OpenSpec (CLI real, nunca simulado)

Fluxo real do CLI para cada braço A:
1. `openspec new change <nome>`
2. `openspec instructions <artifact> --json` (pega os templates reais)
3. preencher `proposal` → `specs` → `design` → `tasks` pelos templates
4. `openspec validate <nome> --strict`
5. copiar `proposal.md` e `design.md` resultantes para `runs/A*/`; escrever o `process.md`

- **A1 (single-role):** executor único preenche os artefatos, sem papéis.
- **A2 (multi-role):** os 4 papéis encenados num único contexto alimentam os artefatos.
- **A3 (multi-agente):** cada papel como subagente isolado; o Orchestrator consolida nos
  artefatos. (Ver nota de isolamento abaixo.)

## Braços B — Spec do Kiro / agentes próprios

Produzir `requirements.md` + `design.md` no estilo do modo spec do Kiro, guiados por
`agents/*.md`:
- **B1 (single-role):** sem papéis.
- **B2 (multi-role):** 4 papéis num único contexto, debate conduzido pelo `orchestrator.md`.
- **B3 (multi-agente):** cada papel como subagente isolado; Orchestrator consolida.

## Nota de isolamento (multi-agente A3/B3)

**Autopilot NÃO é pré-requisito.** Testado empiricamente (2026-10-23): com o Autopilot
desligado, a invocação de subagentes (`invoke_sub_agent`) funcionou normalmente e retornou
resultado. Portanto o mecanismo multi-agente está disponível independentemente do modo —
corrige a afirmação anterior de que "requer modo Autopilot".

O que permanece **não verificado** é o *grau de isolamento de contexto* de cada subagente:
confirmamos que a chamada executa e devolve resposta coerente, mas não há prova, por
inspeção, de que o subagente rodou com contexto genuinamente separado do principal. Portanto:
- Registrar no `process.md` de A3/B3 **como** os subagentes foram invocados.
- Tratar o isolamento como **declarado, não verificado**.
- Se em algum momento ficar claro que não houve isolamento real, A3/B3 degradam para
  multi-role — anotar, não esconder.

## Depois de rodar cada braço

1. Preencher o **checklist** (`rubric/checklist.md`) lendo o `design.md`.
2. Preencher a **rubrica** (`rubric/rubric.md`) — secundária.
3. Conferir os **proxies de esforço** no `process.md`.
4. Deixar tokens/tempo reais em branco, salvo medição instrumentada.
