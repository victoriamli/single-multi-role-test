# Processo — Braço A2 (OpenSpec real + multi-role)

Técnica: OpenSpec de verdade (CLI 1.4.1, change `a2-veridia-test-plan`) com os 4 papéis de
`agents/*.md` encenados num único contexto, debate conduzido pelo Orchestrator. O plano de
testes mora no `design.md`.

## 1. Entendimento do Problema

Veridia é um gateway síncrono de decisão de crédito (Kotlin/Spring Boot, Virtual Threads, K8s
4 pods QA / 8 prod) com caminho assíncrono por Kafka. O objetivo do plano é validar os 5
fluxos E2E e expor os pontos de ruptura reais sob K6, ancorando tudo nos números do alvo: SLA
da API p95<1.5s/p99<2.5s, 5xx<0.5%, ~300 req/s sustentado (~700 pico); 3 motores downstream com
o `engine-bureau` lento e instável (p95 ~1.2s, timeout 2s sem retry); pico Kafka noturno ~2M
msgs/40 min (~830 msg/s) vs 50 msg/s; Oracle com 400M linhas e pool compartilhado de 20
conexões/pod. O plano precisa endereçar as tensões T1–T5, não dar recomendações genéricas.

## 2. Propostas por Papel

- **QA Planner:** mapear os 5 fluxos em cenários E2E; marcar fallback do bureau (fluxo 3) e
  idempotência (fluxo 5) como os de maior risco; priorizar T1 e T3 (podem derrubar produção) e
  tratar T5 como risco de *medição*. Aceite de negócio por fluxo (check de corpo), não só HTTP.
- **QA Performance Engineer:** load/stress da API com executores de **arrival-rate** (não VUs),
  thresholds derivados direto do SLA; pico Kafka via `ramping-arrival-rate` sobre xk6-kafka;
  stress em degraus até o knee point; exigir número para o lag no pico (T4).
- **Backend/Integration Engineer:** o que satura primeiro é o pool Oracle (20/pod) ou o
  downstream, raramente a CPU; teste isolado por caminho dá número falso — só a concorrência
  API+Kafka expõe o pool (T3); mocks dos motores precisam injetar latência/5xx senão T1 não é
  testável; Virtual Threads não resolvem bloqueio em pool finito.

## 3. Pontos de Discussão

- **Soak sim/não (D1):** Planner levantou soak pela tabela que só cresce; Backend rebateu que o
  pico real dura ~40 min e o risco é estrutural, não de soak longo.
- **Executor da API (D2):** `ramping-vus` é mais simples, mas Backend mostrou que VUs presos no
  bureau lento mascaram a violação de SLA.
- **Onde fica o knee point (D3):** stress da API acha capacidade, mas o knee pode ser o bureau
  mock saudável, não a Veridia — faltava degradar o bureau.
- **Isolado vs concorrente (D5):** maior divergência — QA Perf queria isolar por limpeza de
  medição; Backend e Planner insistiram que o pool só satura concorrente.
- **Lag vago (D6):** QA Perf exigiu número; Planner lembrou que precisa de aval do dono do SLA.

## 4. Resolução e Papel do Orquestrador

O Orchestrator forçou que cada tensão fosse discutida por ≥2 papéis e consolidou (ver
`design.md` §Decisions D1–D8):
- D1: incluir E2E/load/stress/spike/volume; soak só como extensão opcional (SK-1), justificado.
- D2: API em arrival-rate (`constant-arrival-rate`/`ramping-arrival-rate`) com VUs folgados.
- D3: separar `sync-api-load-stress` (motores saudáveis → capacidade) de
  `bureau-instability-spike` (degradar bureau → medir fallback e não-propagação de 5xx).
- D5: manter isolados como baseline **e** tornar `shared-pool-contention` (CT-1) bloqueante,
  com Δp95 concorrente−isolado + observabilidade do pool.
- D6: adotar lag ≤ 5 min no pico e drenagem ≤ 15 min, mais rígido que o "15 min" vago, marcado
  como pendente de homologação.
- D7: massa única por padrão, repetição só no cenário de idempotência, métricas por caminho.
- D8: matriz final de 12 testes (E2E-1..5, LD-1/2, ST-1, SP-1, VL-1, CT-1, SK-1 opcional).

## 5. Pendências

- T4: homologar o threshold de lag (5 min / 15 min) com o dono do SLA — meta original não
  contratual.
- Extrapolação QA (4 pods) → produção (8 pods): como projetar capacidade e Δp95 do pool.
- 12 partições Kafka suportam ~830 msg/s com o consumer atual? Medir em VL-1.
- Disponibilidade da seed Oracle de ~400M linhas e do stub de bureau com injeção em QA
  (bloqueiam VL-1/CT-1/E2E-5 se ausentes).
- Observabilidade do pool por pod disponível no ambiente de teste (sem ela, T3 fica cego).

## 6. Proxies de Esforço (factual)

- **Subagentes invocados:** 0 (multi-role de contexto único — os 4 papéis encenados no mesmo
  contexto, sem `invoke_sub_agent`).
- **Chamadas de ferramenta (aprox.):** ~33 (2 todo_list, 9 read_file, 1 list_directory, 8
  execute_bash para CLI OpenSpec/validação/cópia, ~11 fs_write para proposal/7 specs/design/
  tasks/process, + edição dos 2 arquivos de rubrica). [Interface reportou 37; mantida a
  estimativa interna por consistência com os demais braços.]
- **Tamanho dos artefatos:** proposal 61 linhas; design 205 linhas; specs 7 arquivos / 197
  linhas no total; tasks 39 linhas; process ~90 linhas.

## 7. Custo e Tempo Reais (preenchido pelo usuário, da interface)

- **Nº de turnos deste braço:** 1 (um único turno conduziu toda a execução do braço A2)
- **Est. Credits Used (soma dos turnos):** 6.87
- **Elapsed time (soma dos turnos):** 7m 15s
