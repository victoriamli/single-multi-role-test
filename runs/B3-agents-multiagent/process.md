# Registro de processo — B3 (spec do Kiro, multi-agente)

> Registra **como** o plano foi produzido (processo e raciocínio), não o plano. O plano mora
> em `design.md`. Técnica deste braço: **multi-agente** — cada papel especialista rodou como
> **subagente com contexto isolado**, invocado via a ferramenta de subagentes. O executor
> atuou como **Orchestrator**: delegou, provocou a crítica cruzada e consolidou. Os papéis
> **não** foram encenados no mesmo contexto (isso seria multi-role / B2).

## 1. Entendimento do Problema

Alvo: **Veridia**, gateway síncrono+assíncrono de decisão de crédito (Kotlin/Spring Boot,
Virtual Threads, K8s 4 pods QA / 8 prod). Objetivo do plano: provar E2E (F1–F5), carga,
volume e stress com **K6**, com thresholds mensuráveis ancorados no SLA (p95<1.5s, p99<2.5s,
5xx<0.5%, lag Kafka), tratando as cinco tensões: T1 (SLA apertado vs bureau lento/instável),
T2 (pico Kafka 2M/40min), T3 (pool Oracle compartilhado), T4 (meta de lag vaga), T5 (massa vs
idempotência). Foco é **planejamento**, não implementação.

## 2. Propostas por Papel (subagentes isolados)

Cada subagente recebeu **apenas** o alvo e a descrição do seu papel. Na primeira fase, o
raciocínio de um **não** foi repassado ao outro (Planner e Backend em paralelo). O Performance
Engineer recebeu os resultados dos dois.

- **QA Planner** (subagente 1): levantou os 5 fluxos F1–F5 com critérios de aceite de negócio
  (asserção de veredito, não só HTTP), matriz de cenários S1–S16, e priorização por risco
  (críticos: fallback do bureau dentro do SLA, contenção combinada sync+Kafka, pico Kafka,
  idempotência concorrente, baseline). Para cada tensão T1–T5 apontou **qual risco de negócio**
  representa, deixando o "como" para o Performance. Levantou 9 pendências (T4 numérico, SLA do
  fallback, janela de disponibilidade/5xx, gatilho do fallback, paridade sync/async, escopo de
  idempotência, intenção do "sem retry", tamanho representativo da auditoria, dono da massa).

- **Backend/Integration Engineer** (subagente 2): ranqueou os gargalos reais — #1 bureau
  (veredito preso no mais lento; p95 1.2s = 80% do orçamento), #2 **pool Oracle 20/pod** como
  limitador real sob Virtual Threads (threads são baratas; o pool não), #3 folga estrutural do
  SLA, #4 teto do consumer (833 msg/s vs 50; 12 partições → ~70/partição), #5 custo de escrita
  da tabela de 400M, #6 taxa de fallback silenciosa. Classificou o pico Kafka como **spike de
  chegada + soak** (modelo aberto; não despejar batch fixo). Exigiu mock **programável** do
  bureau (presets healthy/slow/flaky/brownout) e alertou que QA(4)→prod(8) é **não-linear**.
  Marcou o cenário **combinado (T3)** como obrigatório e levantou o modelo aberto-vs-fechado
  como item de debate.

- **QA Performance Engineer** (subagente 3): transformou tudo em estratégia K6 concreta —
  seleção de tipos (todos os 6, com soak e volume **fundidos** nos cenários que os tornam
  significativos), **modelo aberto** (arrival-rate) para todos os testes de SLA justificado por
  Little's Law (≈390 em voo a 300 req/s → `preAllocatedVUs 500/maxVUs 800`) e por *coordinated
  omission*; 7 testes A–G com executores, estágios e durações; thresholds por teste
  (`p(95)<1500`, `p(99)<2500`, `rate<0.005`, `dropped_iterations<1`); e um **número para T4**
  (linha vermelha 15 min / dreno 10 min / alvo interno 5 min). Reservou modelo fechado só para
  a corrida de idempotência.

## 3. Pontos de Discussão (divergências entre subagentes)

- **Modelo aberto vs fechado:** o Backend levantou explicitamente como item de debate; o
  Performance tomou posição firme pelo **aberto** com argumento de coordinated omission +
  Little's Law. Convergiram (o Backend já suspeitava que VUs mascarariam o pool/bureau).
- **Classificação do pico Kafka:** Backend disse "spike de chegada + soak, não só volume"; o
  Performance operacionalizou isso **reclassificando volume** como pré-condição de dados
  (400M + massa única), não como forma de tráfego. Sem conflito, mas foi uma decisão de
  enquadramento que o Orchestrator teve de arbitrar.
- **T4 (meta de lag):** Planner e Backend concordaram que é **intestável como está** e
  pediram decisão humana; o Performance **propôs um número concreto** (15/10/5 min). Divergência
  de postura (levantar pendência vs cravar número) resolvida mantendo **ambos**: número
  proposto no design + pendência de ratificação no requirements.
- **SLA do fallback:** nenhum subagente tinha autoridade para decidir se `APPROVED_WITH_REVIEW`
  cumpre o mesmo p95/p99 — ficou como pendência cruzada (Planner levantou, Performance repetiu).
- **Extrapolação QA→prod:** Backend recusou projeção linear; o Performance acatou (reportar por
  pod). Sem divergência real, mas registrado como restrição de medição.

## 4. Resolução e Papel do Orquestrador

Como Orchestrator, provoquei a crítica cruzada passando os resultados de Planner+Backend ao
Performance e, na consolidação, apliquei estas decisões:
- **Modelo aberto** adotado para B/C/D/E/F; fechado só para G (idempotência). Guarda
  `dropped_iterations<1` incorporada como anti-omission.
- **Teste F (combinado T3)** elevado a **bloqueante de release**, com comparação Δp95
  isolado vs combinado — consolidando a exigência do Backend com a priorização do Planner.
- **T4** resolvido com número **e** pendência: design crava 15/10/5 min com racional (o lag
  precisa drenar dentro da janela 23h–00h); requirements marca como ratificação humana.
- **Volume/soak fundidos** nos Testes E/F (não como corridas soltas), seguindo o enquadramento
  do Performance, por serem significativos só com estado de dados real + duração do pico.
- **Massa (T5):** unicidade por padrão + fração ~1–2% tagueada e excluída dos thresholds de
  SLA; seed de 400M como pré-condição.
- Cada tensão T1–T5 foi tocada por **≥2 papéis** (ex.: T1 Planner+Backend+Performance; T3
  Planner+Backend+Performance; T4 Planner+Backend levantaram, Performance quantificou).

### Isolamento dos subagentes — **declarado, não verificado**

Conforme a nota de isolamento do `HOW-TO-RUN.md`: a invocação de subagentes executou e
retornou respostas coerentes e distintas por papel. **Não** há prova por inspeção de que cada
subagente rodou com contexto genuinamente separado do principal. Portanto o isolamento é
tratado como **declarado, não verificado**. Observação factual: o subagente de Performance,
na primeira chamada, devolveu um **resumo** em vez do corpo integral; foi reinvocado para
confirmar o conteúdo completo — comportamento compatível com um agente separado resumindo seu
próprio trabalho, mas isso **não** prova isolamento de contexto. Nenhum sinal de que os papéis
degradaram para multi-role foi observado (cada um permaneceu na sua faixa: Planner não definiu
thresholds; Performance não alterou requisitos).

## 5. Pendências

Herdadas dos subagentes e consolidadas em `requirements.md` §7 (RQ-OPEN-1..8):
T4 (ratificar número + referência de medição), SLA do fallback, teto de taxa de fallback,
janela de medição de disponibilidade/5xx, paridade sync/async, escopo da idempotência
(cross-path?), **C_max** (bloqueante para stress e metas de lag), e disponibilidade de seed de
400M + stub injetável do bureau em QA (pré-condições bloqueantes).

## 6. Proxies de Esforço (factual)

- **Subagentes invocados:** 3 papéis distintos (QA Planner, Backend/Integration Engineer, QA
  Performance Engineer).
- **Chamadas à ferramenta de subagentes (`invoke_sub_agent`):** 4 (Planner ×1, Backend ×1,
  Performance ×2 — a 2ª para obter o corpo completo após um resumo).
- **Como foram invocados:** via `invoke_sub_agent` com o agente `general-task-execution`;
  Planner e Backend em paralelo (uma mensagem, duas chamadas), cada um recebendo só alvo+papel;
  Performance em seguida, recebendo os resultados consolidados de ambos. Isolamento: **declarado,
  não verificado** (ver §4).
- **Chamadas de ferramenta no total (aprox.):** ~15 (leituras de contexto dos arquivos
  permitidos, 4 invocações de subagente, escrita dos 3 artefatos, edição das 2 planilhas de
  rubrica, gestão da lista de tarefas).
- **Tamanho dos artefatos:** requirements ~191 linhas; design ~240 linhas; process ~125 linhas.

## 7. Custo e Tempo Reais (preenchido pelo usuário, da interface)

- **Nº de turnos deste braço:** 1  (um único turno de execução contínua até aqui)
- **Est. Credits Used (soma dos turnos):** 11.51
- **Elapsed time (soma dos turnos):** 13m 25s
