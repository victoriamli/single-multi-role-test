# Registro de processo — Braço B1 (Spec do Kiro + single-role)

> Registra **como** o plano foi produzido, não o plano (que vive em `design.md`). Braço
> single-role: sem papéis distintos nem debate encenado — as seções 2–4 são naturalmente
> curtas, como prevê `output-format.md`.

## 1. Entendimento do Problema

Veridia é um gateway síncrono (`POST /v1/assessments`) que orquestra 3 motores externos e
consolida um veredito, com um caminho assíncrono por Kafka e persistência de auditoria em
Oracle. O objetivo do plano é garantir funcionamento sob carga real e extrema (E2E, load,
stress, spike, soak, volume) com critérios mensuráveis. A dificuldade central está nas tensões
deliberadas: o SLA (p95<1.5s/p99<2.5s) tem pouca folga sobre o `engine-bureau` lento/instável
(T1); o pico Kafka noturno (2M msgs/40min) é atípico frente aos 50 msg/s normais (T2); API e
consumer disputam o mesmo pool Oracle (20/pod) no pico (T3); a meta de lag no pico é vaga (T4);
e a massa mal gerada mascara a carga de escrita ao ativar idempotência (T5). Interpretei o
alvo como "planejar testes que exponham essas tensões, não que as escondam".

## 2. Propostas por Papel

Single-role — consolidado num parágrafo (sem papéis separados). Como executor único cobri as
três perspectivas de uma vez: **planejamento** (mapear os 5 fluxos E2E e priorizar os caminhos
críticos F1/F3/F5 e riscos T1–T5); **performance** (escolher open model/arrival-rate para não
mascarar o SLA sob bureau lento, dimensionar VUs por Little's Law, definir estágios e
thresholds numéricos ancorados no SLA, e transformar T4 em alvo testável de 5 min); e
**backend/integração** (identificar o pool Oracle e o bureau como gargalos dominantes, e exigir
teste concorrente API+Kafka para não esconder o acoplamento T3, além de seed de 400M como
pré-condição de volume).

## 3. Pontos de Discussão

Sem papéis para divergir; os trade-offs foram ponderados internamente:
- **Open vs. closed model:** closed (VUs) é mais simples, mas mascara o SLA quando o bureau
  trava VUs. Resolvido por open model (D2) apesar do custo de dimensionar `preAllocatedVUs`.
- **Teste isolado vs. concorrente (T3):** isolado é mais limpo de interpretar, mas esconde a
  disputa pelo pool. Mantive ambos (baseline isolado + CT-1 concorrente com Δp95).
- **T4 vago:** escolher um número arbitrário vs. deixar em aberto. Resolvido propondo 5 min
  como alvo e 15 min como teto, marcando como pendência de confirmação humana.
- **Soak:** soak longo "puro" tem baixo retorno; redirecionei para drenagem do pico (foco em
  T2/T4) e deixei o soak de estabilidade como secundário.

## 4. Resolução e Papel do Orquestrador

Sem orquestração entre agentes. As decisões foram tomadas por priorização direta ancorada nas
tensões: toda escolha que aumentava a chance de **falso verde** (closed model, teste isolado,
massa repetida, volume ausente) foi rejeitada em favor da opção que **expõe o risco real**,
mesmo quando mais cara de montar. Thresholds foram derivados diretamente dos SLAs declarados;
onde o SLA era vago (T4), propus número com racional explícito e registrei como pendência.

## 5. Pendências

Consolidadas em D7 do design: (1) confirmar alvo de lag de pico (5 min proposto, teto 15 min);
(2) disponibilidade de ambiente QA com volume e stub de bureau fiéis; (3) observabilidade
exportável do pool Oracle para medir T3 em vez de inferir; (4) capacidade-alvo máxima do
stress; (5) retenção/expurgo da auditoria (fora de escopo, mas afeta validade futura).

## 6. Proxies de Esforço (factual)

- **Subagentes invocados:** 0
- **Chamadas de ferramenta (aprox.):** ~13 (5 leituras de arquivo do cenário/formato/rubrica;
  2 listagens de diretório; 3 escritas de artefato; 2 edições de rubrica; atualizações de lista
  de tarefas).
- **Tamanho dos artefatos:** requirements.md ~150 linhas, design.md ~170 linhas,
  process.md ~95 linhas.

## 7. Custo e Tempo Reais (preenchido pelo usuário, da interface)

- **Nº de turnos deste braço:** 1
- **Est. Credits Used (soma dos turnos):** 4.5
- **Elapsed time (soma dos turnos):** 4m 34s
