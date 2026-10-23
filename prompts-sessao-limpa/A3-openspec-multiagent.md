# Prompt para rodar o A3 em sessão limpa (copiar para um chat/janela novo)

> Cole o conteúdo abaixo da linha num **chat novo / janela nova** do Kiro, aberto na pasta
> `single-multi-role-test`. Objetivo: rodar o A3 nas mesmas condições limpas dos demais.
> Observação: A3 usa subagentes (multi-agente). Se possível, rode em modo Autopilot — mas
> veja a nota sobre isso nos passos.

---

Você vai executar **um único braço** de um experimento de comparação de métodos de
planejamento de testes. Siga estritamente as instruções e **não** olhe resultados de outros
braços.

## Regras de isolamento (importante)

- Leia APENAS estes arquivos como contexto:
  - `scenario/scenario.md`, `scenario/context-base.md`, `scenario/target-app.md`
  - `agents/orchestrator.md`, `agents/qa-planner.md`, `agents/qa-performance-engineer.md`, `agents/backend-integration-engineer.md`
  - `output-format.md`, `HOW-TO-RUN.md`
- **NÃO** leia a pasta `runs/`, nem `old/`, nem `prompts-sessao-limpa/` — contêm outros braços
  e olhá-los contamina este. Se achar que precisa de algo de lá, pare e pergunte.
- Ignore qualquer steering de outros projetos do workspace.

## Braço a executar: A3 — OpenSpec (real) + multi-agente

**Processo:** OpenSpec de verdade, via CLI (não simule a estrutura).
**Técnica:** multi-agente — cada papel especialista roda como **subagente com contexto
isolado**, invocado via a ferramenta de subagentes (`invoke_sub_agent` / general-task-execution).
Você atua como **Orchestrator** (papel em `agents/orchestrator.md`): delega, provoca a crítica
cruzada e consolida. NÃO encene os papéis você mesmo — eles devem ser subagentes reais.

### Como conduzir o multi-agente

1. Invoque **QA Planner** e **Backend/Integration Engineer** como subagentes isolados (podem
   ser em paralelo — são independentes). Dê a cada um apenas o alvo (`target-app.md` resumido)
   e o seu papel (`agents/<papel>.md`). Não repasse o raciocínio de um para o outro nesta fase.
2. Invoque **QA Performance Engineer** como subagente, passando os resultados do Planner e do
   Backend (dependência real).
3. Como Orchestrator, faça a crítica cruzada: aponte convergências e divergências entre os três
   e consolide.

### Nota de honestidade (obrigatória)

- Se, nesta sessão, a invocação de subagentes **não funcionar** (for bloqueada), **PARE e
  avise** — não finja multi-agente encenando os papéis você mesmo (isso seria multi-role).
- No `process.md`, registre **como** os subagentes foram invocados e trate o isolamento de
  contexto como **declarado, não verificado** (não há prova de que o subagente rodou isolado).

## Passos (OpenSpec real)

1. `openspec new change a3-veridia-test-plan`
2. `openspec status --change a3-veridia-test-plan --json`
3. `openspec instructions <artifact> --json` por artefato (templates reais).
4. Alimente os artefatos com a análise dos subagentes, na ordem de dependência: `proposal` →
   `specs` (uma por capability, `### Requirement` SHALL + `#### Scenario` WHEN/THEN) → `design`
   → `tasks`. O **planejamento de testes mora no `design.md`**.
5. `openspec validate a3-veridia-test-plan --strict`.
6. Copie `proposal.md` e `design.md` do change para `runs/A3-openspec-multiagent/`.
7. Escreva `runs/A3-openspec-multiagent/process.md` seguindo `output-format.md` (seções 1–7).
   - Seção 2: resuma de forma concisa o que cada subagente propôs.
   - Seção 3–4: divergências e como o Orchestrator consolidou.
   - Seção 6 (proxies): **nº de subagentes invocados**, chamadas de ferramenta, tamanho dos
     artefatos e **nº de turnos**.
   - Seção 7: Est. Credits Used e Elapsed time EM BRANCO (o usuário preenche).
8. Preencha a coluna A3 em `rubric/checklist.md` (Sim/Não lendo o design, com observação) e em
   `rubric/rubric.md` (nota 0–5 por critério), **rotulando como autoavaliação do executor**.

## Importante

- Foco é planejamento, não implementação de scripts K6.
- Ancore tudo no alvo (fluxos, SLAs, tensões T1–T5 de `target-app.md`), não genérico.
- Ao terminar, informe **quantos turnos** o braço levou e **quantos subagentes** foram
  invocados, para o usuário somar crédito/tempo e para o registro de esforço.
