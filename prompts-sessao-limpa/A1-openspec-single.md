# Prompt para rodar o A1 em sessão limpa (copiar para um chat/janela novo)

> Cole o conteúdo abaixo da linha num **chat novo / janela nova** do Kiro, aberto na pasta
> `single-multi-role-test`. Objetivo: rodar o A1 nas mesmas condições limpas do A2.

---

Você vai executar **um único braço** de um experimento de comparação de métodos de
planejamento de testes. Siga estritamente as instruções e **não** olhe resultados de outros
braços.

## Regras de isolamento (importante)

- Leia APENAS estes arquivos como contexto:
  - `scenario/scenario.md`, `scenario/context-base.md`, `scenario/target-app.md`
  - `output-format.md`, `HOW-TO-RUN.md`
- **NÃO** leia a pasta `runs/` nem `old/` nem `prompts-sessao-limpa/` — contêm outros braços e
  olhá-los contamina este. Se achar que precisa de algo de lá, pare e pergunte.
- Ignore qualquer steering de outros projetos do workspace.

## Braço a executar: A1 — OpenSpec (real) + single-role

**Processo:** OpenSpec de verdade, via CLI (não simule a estrutura).
**Técnica:** resolva diretamente, num fluxo de raciocínio só. (Não leia `agents/*.md`.)

## Passos

1. Rode o OpenSpec real nesta pasta:
   - `openspec new change a1-veridia-test-plan`
   - `openspec status --change a1-veridia-test-plan --json`
   - `openspec instructions <artifact> --json` para cada artefato (use os templates reais)
2. Preencha, pelos templates reais, na ordem de dependência: `proposal` → `specs` (uma por
   capability, formato `### Requirement` SHALL + `#### Scenario` WHEN/THEN) → `design` →
   `tasks`. O **planejamento de testes mora no `design.md`**.
3. Valide: `openspec validate a1-veridia-test-plan --strict`.
4. Copie `proposal.md` e `design.md` do change para `runs/A1-openspec-single/`.
5. Escreva `runs/A1-openspec-single/process.md` seguindo `output-format.md` (seções 1–7).
   - Seção 2–4: single-role, portanto concisas (sem papéis/debate).
   - Seção 6 (proxies): subagentes = 0; conte chamadas de ferramenta, tamanho dos artefatos e
     **nº de turnos**.
   - Seção 7: deixe Est. Credits Used e Elapsed time EM BRANCO (o usuário preenche).
6. Preencha a coluna A1 em `rubric/checklist.md` (Sim/Não lendo o design, com observação
   concisa) e em `rubric/rubric.md` (nota 0–5 por critério), **rotulando como autoavaliação
   do executor**.

## Importante

- Foco é planejamento, não implementação de scripts K6.
- Ancore tudo no alvo (fluxos, SLAs, tensões T1–T5 de `target-app.md`), não genérico.
- Ao terminar, informe quantos turnos o braço levou, para o usuário somar crédito/tempo.
