# Prompt para rodar o B3 em sessão limpa (copiar para um chat/janela novo)

> Cole o conteúdo abaixo da linha num **chat novo / janela remota WSL** do Kiro, com
> **Autopilot ligado**, aberto na pasta `single-multi-role-test`.

---

Você vai executar **um único braço** de um experimento de comparação de métodos de
planejamento de testes. Siga estritamente as instruções e **não** olhe resultados de outros
braços.

## Regras de isolamento (importante)

- Leia APENAS: `scenario/scenario.md`, `scenario/context-base.md`, `scenario/target-app.md`,
  `agents/orchestrator.md`, `agents/qa-planner.md`, `agents/qa-performance-engineer.md`,
  `agents/backend-integration-engineer.md`, `output-format.md`, `HOW-TO-RUN.md`.
- **NÃO** leia `runs/`, `old/`, `prompts-sessao-limpa/`, nem `openspec/`. Se achar que precisa
  de algo de lá, pare e pergunte.
- Ignore qualquer steering de outros projetos do workspace.

## Braço a executar: B3 — Spec do Kiro + multi-agente

**Processo:** modo spec do Kiro (não OpenSpec). Gere `requirements.md` + `design.md`.
**Técnica:** multi-agente — cada papel especialista roda como **subagente com contexto
isolado**, invocado via a ferramenta de subagentes. Você atua como **Orchestrator**: delega,
provoca a crítica cruzada e consolida. NÃO encene os papéis você mesmo.

### Como conduzir o multi-agente

1. Invoque **QA Planner** e **Backend/Integration Engineer** como subagentes isolados (podem
   ser paralelos — independentes). Dê a cada um só o alvo e o seu papel; não repasse o
   raciocínio de um ao outro nesta fase.
2. Invoque **QA Performance Engineer** como subagente, passando os resultados dos dois.
3. Como Orchestrator, faça a crítica cruzada e consolide.

### Nota de honestidade (obrigatória)

- Se a invocação de subagentes **não funcionar** nesta sessão, **PARE e avise** — não finja
  multi-agente encenando os papéis (isso seria multi-role).
- No `process.md`, registre **como** os subagentes foram invocados e trate o isolamento como
  **declarado, não verificado**.

## Passos

1. Conduza o multi-agente acima.
2. Crie `runs/B3-agents-multiagent/requirements.md` (requisitos verificáveis) e
   `runs/B3-agents-multiagent/design.md` (**o plano de testes**: tipos com justificativa,
   modelagem de carga, thresholds numéricos, massa, riscos de medição).
3. Crie `runs/B3-agents-multiagent/process.md` seguindo `output-format.md` (seções 1–7):
   - Seção 2: resuma o que cada subagente propôs. Seção 3–4: divergências e consolidação.
   - Seção 6 (proxies): **nº de subagentes invocados**; chamadas; tamanho; **nº de turnos**.
   - Seção 7: Est. Credits Used e Elapsed time EM BRANCO.
4. Preencha a coluna B3 em `rubric/checklist.md` e `rubric/rubric.md`, **rotulando como
   autoavaliação do executor**.

## Importante

- Foco é planejamento, não implementação. Ancore tudo no alvo (T1–T5). Ao terminar, informe
  **nº de turnos** e **nº de subagentes**.
