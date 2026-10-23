# Prompt para rodar o B2 em sessão limpa (copiar para um chat/janela novo)

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

## Braço a executar: B2 — Spec do Kiro + multi-role

**Processo:** modo spec do Kiro (não OpenSpec). Gere `requirements.md` + `design.md`.
**Técnica:** multi-role — você assume os quatro papéis de `agents/*.md` num **único contexto**
e **encena o debate** conduzido pelo `orchestrator.md` antes de consolidar. Cada papel propõe;
o Orchestrator provoca a crítica cruzada e consolida.

## Passos

1. Conduza o debate multi-role (propostas, divergências, resolução).
2. Crie `runs/B2-agents-multirole/requirements.md`: requisitos de teste verificáveis.
3. Crie `runs/B2-agents-multirole/design.md`: **o plano de testes** — tipos escolhidos/
   descartados com justificativa, modelagem de carga, thresholds numéricos, massa, riscos de
   medição.
4. Crie `runs/B2-agents-multirole/process.md` seguindo `output-format.md` (seções 1–7):
   - Seção 2: resuma de forma concisa o que cada papel propôs.
   - Seção 3–4: divergências e resolução do Orchestrator.
   - Seção 6 (proxies): subagentes = 0 (multi-role de contexto único); chamadas; tamanho; **nº
     de turnos**.
   - Seção 7: Est. Credits Used e Elapsed time EM BRANCO.
5. Preencha a coluna B2 em `rubric/checklist.md` e `rubric/rubric.md`, **rotulando como
   autoavaliação do executor**.

## Importante

- Foco é planejamento, não implementação. Ancore tudo no alvo (T1–T5). Informe nº de turnos.
