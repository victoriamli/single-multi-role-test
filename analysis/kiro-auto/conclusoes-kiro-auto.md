# Conclusões — Experimento single/multi-role/multi-agente × OpenSpec/Spec-Kiro

Rodada definitiva: seis braços, todos na **mesma condição** (janela remota WSL, Autopilot
ligado, sessão limpa por braço, OpenSpec real na linha A). Executor: Kiro. Alvo: Veridia.

## Matriz de resultados

| Braço | Processo | Técnica | Checklist | Rubrica (autoaval.) | Créditos | Tempo | Subagentes |
|-------|----------|---------|:---------:|:-------------------:|---------:|------:|:----------:|
| A1 | OpenSpec  | single-role  | 8/8 | 67 | 6.74  | 7m04  | 0 |
| A2 | OpenSpec  | multi-role   | 8/8 | 69 | 6.87  | 7m15  | 0 |
| A3 | OpenSpec  | multi-agente | 8/8 | 69 | 12.73 | 12m22 | 3 |
| B1 | Spec Kiro | single-role  | 8/8 | 67 | 4.5   | 4m34  | 0 |
| B2 | Spec Kiro | multi-role   | 8/8 | 69 | 4.63  | 4m37  | 0 |
| B3 | Spec Kiro | multi-agente | 8/8 | 69 | 11.51 | 13m25 | 3 |

## Leitura por eixo

### Técnica (dentro do mesmo processo)
- **single → multi-role:** qualidade +2 na rubrica (dentro do ruído), **custo ~igual**
  (A: 6.74→6.87; B: 4.5→4.63). O multi-role "sai de graça": os papéis discutem num contexto
  só, sem processar muito mais token.
- **multi-role → multi-agente:** qualidade **+0**, **custo quase dobra** (A: 6.87→12.73;
  B: 4.63→11.51). O salto é o preço dos 3 subagentes (cada um reprocessa o contexto).

### Processo (dentro da mesma técnica)
- **OpenSpec vs Spec-Kiro:** mesma qualidade (notas idênticas por quadrante), mas o OpenSpec
  custa **~33–50% mais crédito** (A1 6.74 vs B1 4.5; A2 6.87 vs B2 4.63; A3 12.73 vs B3 11.51).
  O overhead é a cerimônia do CLI: gerar specs no formato rígido + `validate --strict`.

## Achado principal (confiável)

**Neste cenário, a cerimônia extra não se pagou — nem a de técnica nem a de processo.**
- O multi-agente custou ~85% mais que o multi-role sem ganhar qualidade medível.
- O OpenSpec custou mais que o Spec-Kiro sem ganhar qualidade medível.
- O combo mais barato que entregou o mesmo resultado: **B2 (Spec-Kiro + multi-role), 4.63
  créditos**. O mais caro: **A3 (OpenSpec + multi-agente), 12.73** — ~2,75× o custo do B2.

A métrica que sustenta isso é o **custo (crédito)**, que é confiável: não sofre com o ruído de
tempo (ver ressalvas) nem com o viés de autoavaliação.

## Ressalvas (essenciais para interpretar)

1. **A rubrica saturou e NÃO discrimina.** Todos os braços: checklist 8/8 e rubrica 67–69.
   A diferença de 2 pontos (single vs multi) é **menor que o ruído de autoavaliação** já
   medido (o mesmo A1 oscilou 57↔67 entre rodadas). Portanto: pela nota, os seis são **empate
   técnico em qualidade**. Não concluir "multi-role é melhor" a partir de +2.
2. **Autoavaliação, não cega.** Todas as notas foram dadas pelo executor que gerou os planos.
   Viés pró-resultado. **A comparação cega dos `design.md` por um humano é o passo pendente que
   daria peso real às diferenças de qualidade** (que, se existem, estão na textura dos planos,
   não no placar).
3. **Tempo é ruidoso.** Mesmo na janela remota, "elapsed time" inclui espera entre turnos.
   Usar como ordinal grosseiro, não como medida fina. Confiar no crédito.
4. **Isolamento dos subagentes (A3/B3): declarado, não verificado.** O mecanismo roda sem
   Autopilot (testado), mas não há prova de que cada subagente teve contexto genuinamente
   separado. Se não houve, A3/B3 degradam para multi-role.
5. **N=1 cenário, 1 rodada por braço.** Estudo de caso, não estatística. Em um alvo com
   tensões mais independentes, o multi-agente (paralelo) poderia render mais — aqui o gargalo
   era central (pool Oracle) e todos convergiram nele barato, o que favorece os métodos simples.

## Recomendação prática (para este tipo de tarefa, com Kiro)

- **Padrão:** multi-role (ganho de profundidade sem custo extra relevante). Processo conforme
  a necessidade de rastreabilidade: Spec-Kiro se o custo importa; OpenSpec se você precisa dos
  artefatos formais/validação.
- **Multi-agente:** reservar para tarefas com subtarefas genuinamente independentes ou quando
  o isolamento de contexto for essencial (ex.: revisão crítica honesta). Para planejamento com
  um gargalo central, o custo extra não compensou.

## Respostas às perguntas de conclusão

1. **Multi-role vs single-role justifica o esforço?** Sim, porque o esforço é ~nulo (mesmo
   custo). O ganho de qualidade é pequeno/ruidoso, mas o custo não aumenta — então vale como
   padrão.
2. **Multi-agente superou multi-role?** Não neste cenário: mesma qualidade medível, ~85% mais
   caro. A diferença qualitativa (isolamento preservando divergências) existe nos designs mas
   não apareceu na rubrica — depende da avaliação cega para confirmar.
3. **OpenSpec vs Spec-Kiro?** Mesma qualidade; OpenSpec mais caro mas com rastreabilidade
   formal e gate de validação. Escolha por necessidade de auditoria, não por qualidade do plano.
4. **Onde o esforço extra NÃO compensou?** Multi-agente (custo dobrou, qualidade igual) e
   OpenSpec no single-role (custo maior, qualidade igual).
