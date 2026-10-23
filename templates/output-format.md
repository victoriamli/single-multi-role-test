# Formato do registro de processo (`process.md`)

Cada braço produz um `process.md` neste formato, registrando **como** o plano foi produzido
(o processo e o raciocínio), não o plano em si. O plano de testes mora no `design.md` de cada
run.

> O debate não precisa ser transcrito palavra por palavra. Um resumo conciso das propostas de
> cada papel, os pontos de discussão e o resultado já bastam. Nos braços single-role, as
> seções de papéis/debate ficam naturalmente curtas (não há papéis distintos) — não force um
> debate artificial.

## 1. Entendimento do Problema
Resumo do alvo e do objetivo do plano de testes, como o braço o interpretou.

## 2. Propostas por Papel
Resumo conciso do que cada papel propôs. (Single-role: consolidado num parágrafo.)
- **QA Planner:** o que levantou (fluxos, riscos, prioridade).
- **QA Performance Engineer:** tipos de teste, modelagem, thresholds propostos.
- **Backend/Integration Engineer:** gargalos e viabilidade.

## 3. Pontos de Discussão
As divergências que apareceram entre os papéis (ou, no single-role, os trade-offs ponderados).

## 4. Resolução e Papel do Orquestrador
Como cada divergência foi resolvida e o que o Orquestrador consolidou. (Single-role: como as
decisões foram tomadas sem debate.)

## 5. Pendências
Itens que ficaram em aberto e dependem de decisão humana.

## 6. Proxies de Esforço (factual)
Preenchido com o que é contável nesta execução:
- **Subagentes invocados:** <n>
- **Chamadas de ferramenta (aprox.):** <n>
- **Tamanho dos artefatos:** proposal/requirements <linhas>, design <linhas>, process <linhas>

> São *proxies* de custo/esforço, não medidas exatas. Servem para ordenar os braços por
> esforço relativo, não para cravar valores.

## 7. Custo e Tempo Reais (preenchido pelo usuário, da interface)
A interface do Kiro mostra, ao fim de cada turno, "Est. Credits Used" e "Elapsed time". Estes
são os números reais a registrar aqui — o executor (Kiro) NÃO os vê e não deve estimá-los.

- **Nº de turnos deste braço:** <n>  (quantos turnos a interface somou para este braço)
- **Est. Credits Used (soma dos turnos):** <preencher da interface>
- **Elapsed time (soma dos turnos):** <preencher da interface>

> Se o braço ocupou vários turnos (típico no multi-agente), somar os valores de todos os
> turnos dele. O Kiro anota quantos turnos o braço levou para facilitar a soma.
