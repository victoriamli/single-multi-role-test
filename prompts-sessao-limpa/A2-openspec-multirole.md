# Prompt para rodar o A2 em sessão limpa (copiar para um chat novo)

> Cole o conteúdo abaixo da linha num **chat novo** do Kiro (sessão limpa), na mesma janela
> deste workspace. O objetivo é rodar o braço A2 sem o executor ter visto os outros braços.

---

Você vai executar **um único braço** de um experimento de comparação de métodos de
planejamento de testes. Siga estritamente as instruções e **não** olhe resultados de outros
braços.

## Regras de isolamento (importante)

- Leia APENAS estes arquivos como contexto:
  - `scenario/scenario.md`, `scenario/context-base.md`, `scenario/target-app.md`
  - `agents/orchestrator.md`, `agents/qa-planner.md`, `agents/qa-performance-engineer.md`, `agents/backend-integration-engineer.md`
  - `output-format.md`, `HOW-TO-RUN.md`
- **NÃO** leia a pasta `runs/` nem a pasta `old/`. Eles contêm outros braços do experimento e
  olhá-los contamina este. Se precisar de algo que acha que está lá, pare e pergunte.
- Ignore qualquer steering de outros projetos do workspace — não faz parte deste experimento.

## Braço a executar: A2 — OpenSpec (real) + multi-role

**Processo:** OpenSpec de verdade, via CLI (não simule a estrutura).
**Técnica:** multi-role — você assume os quatro papéis de `agents/*.md` num único contexto e
**encena o debate** entre eles, conduzido pelo `orchestrator.md`, antes de consolidar. Cada
papel propõe sua visão; o Orchestrator provoca a crítica cruzada e só então consolida.

## Passos

1. Rode o OpenSpec real nesta pasta:
   - `openspec new change a2-veridia-test-plan`
   - `openspec status --change a2-veridia-test-plan --json` (ver ordem dos artefatos)
   - `openspec instructions <artifact> --json` para cada artefato (use os templates reais)
2. Preencha, pelos templates reais, na ordem de dependência: `proposal` → `specs` (uma por
   capability, formato `### Requirement` SHALL + `#### Scenario` WHEN/THEN) → `design` →
   `tasks`. O **planejamento de testes mora no `design.md`**.
3. Conduza o conteúdo via debate multi-role: registre, de forma concisa, as propostas de cada
   papel, os pontos de divergência e a resolução do Orchestrator.
4. Valide: `openspec validate a2-veridia-test-plan --strict`.
5. Copie `proposal.md` e `design.md` do change para `runs/A2-openspec-multirole/`.
6. Escreva `runs/A2-openspec-multirole/process.md` seguindo `output-format.md` (seções 1–7).
   - Na seção 6 (proxies), conte subagentes (=0 aqui, é multi-role de contexto único),
     chamadas de ferramenta e tamanho dos artefatos, e **nº de turnos**.
   - Na seção 7, deixe Est. Credits Used e Elapsed time EM BRANCO (o usuário preenche da
     interface).
7. Preencha a coluna A2 em `rubric/checklist.md` (Sim/Não lendo o design, com observação
   concisa) e em `rubric/rubric.md` (nota 0–5 por critério), **rotulando como autoavaliação
   do executor**.

## Importante

- Foco é planejamento, não implementação de scripts K6.
- Todo critério/recomendação deve ser ancorado no alvo (fluxos, SLAs, tensões T1–T5 descritas
  em `target-app.md`), não genérico.
- Ao terminar, informe quantos turnos o braço levou, para o usuário somar crédito/tempo.
