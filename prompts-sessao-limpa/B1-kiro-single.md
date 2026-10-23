# Prompt para rodar o B1 em sessão limpa (copiar para um chat/janela novo)

> Cole o conteúdo abaixo da linha num **chat novo / janela remota WSL** do Kiro, com
> **Autopilot ligado**, aberto na pasta `single-multi-role-test`.

---

Você vai executar **um único braço** de um experimento de comparação de métodos de
planejamento de testes. Siga estritamente as instruções e **não** olhe resultados de outros
braços.

## Regras de isolamento (importante)

- Leia APENAS: `scenario/scenario.md`, `scenario/context-base.md`, `scenario/target-app.md`,
  `output-format.md`, `HOW-TO-RUN.md`.
- **NÃO** leia `runs/`, `old/`, `prompts-sessao-limpa/`, nem `openspec/` — contêm outros
  braços. Se achar que precisa de algo de lá, pare e pergunte.
- Ignore qualquer steering de outros projetos do workspace.

## Braço a executar: B1 — Spec do Kiro + single-role

**Processo:** modo spec do Kiro (não OpenSpec). Gere os artefatos no estilo do spec nativo:
`requirements.md` (requisitos, em EARS/"SHALL" se possível) e `design.md` (o plano de testes).
**Técnica:** single-role — executor único. (Não leia `agents/*.md`.)

## Passos

1. Crie `runs/B1-agents-single/requirements.md`: requisitos de teste derivados do alvo
   (fluxos críticos + não-funcionais + tensões), cada um verificável.
2. Crie `runs/B1-agents-single/design.md`: **onde o planejamento de testes mora** — tipos de
   teste escolhidos/descartados com justificativa, modelagem de carga (executor K6, taxa/VUs,
   estágios), thresholds numéricos ancorados nos SLAs, estratégia de massa, riscos de medição.
3. Crie `runs/B1-agents-single/process.md` seguindo `output-format.md` (seções 1–7):
   - Seções 2–4: single-role, concisas (sem papéis/debate).
   - Seção 6 (proxies): subagentes = 0; chamadas de ferramenta; tamanho dos artefatos; **nº de
     turnos**.
   - Seção 7: Est. Credits Used e Elapsed time EM BRANCO (o usuário preenche).
4. Preencha a coluna B1 em `rubric/checklist.md` (Sim/Não lendo o design, com observação) e em
   `rubric/rubric.md` (nota 0–5 por critério), **rotulando como autoavaliação do executor**.

## Importante

- Foco é planejamento, não implementação de scripts K6.
- Ancore tudo no alvo (fluxos, SLAs, tensões T1–T5 de `target-app.md`), não genérico.
- Ao terminar, informe **quantos turnos** o braço levou.
