# Conclusões — avaliação dos 6 planos de teste (Veridia)

Documento de fechamento da análise dos braços A1–A3 e B1–B3. Baseado na leitura dos seis
`design.md`, do alvo (`context/target-app.md`), do prompt (`context/prompt.md`) e nas notas
consolidadas em `checklist-kiro.md` e `rubric-kiro.md`.

## 1. Os braços e as duas famílias

Há dois eixos no experimento:

- **Família (como o plano é gerado):** **A = OpenSpec** (fluxo de spec estruturado) · **B =
  agentes próprios** (sem o arcabouço de spec).
- **Grau de colaboração (dentro de cada família):** single-role → multi-role (vários papéis no
  mesmo contexto) → multi-agente (papéis em subagentes isolados, com um Orchestrator arbitrando).

| Braço | Família | Colaboração |
|-------|---------|-------------|
| A1 | OpenSpec | single-role |
| A2 | OpenSpec | multi-role |
| A3 | OpenSpec | multi-agente (isolado) |
| B1 | Agentes próprios | single-role |
| B2 | Agentes próprios | multi-role |
| B3 | Agentes próprios | multi-agente (isolado) |

Com isso dá para separar o que cada escolha custa e entrega: subir a colaboração (1→2→3 papéis)
e trocar a família (OpenSpec vs agentes próprios).

## 2. Placar

Rubrica ponderada (máx. 70) e eficiência (qualidade por token, com os tokens informados):

| Braço | Família / colaboração | Rubrica | % (de 70) | Tokens | Tempo | Qualidade/token |
|-------|-----------------------|---------|-----------|--------|-------|-----------------|
| B3 | agentes · multi-agente | 68 | 97% | 11.51 | 13m25s | 5.9 |
| A3 | OpenSpec · multi-agente | 63 | 90% | 12.73 | 12m22s | 4.9 |
| B2 | agentes · multi-role | 56 | 80% | 4.63 | 4m37s | **12.1** |
| A2 | OpenSpec · multi-role | 55 | 79% | 6.87 | 7m15s | 8.0 |
| A1 | OpenSpec · single-role | 53 | 76% | 6.74 | 7m04s | 7.9 |
| B1 | agentes · single-role | 49 | 70% | 4.50 | 4m34s | 10.9 |

- **Qualidade absoluta:** B3 > A3 > B2 > A2 > A1 > B1.
- **Qualidade por token:** B2 > B1 > A2 > A1 > B3 > A3.

O checklist de presença (8 critérios) saturou — **todos os seis deram 8/8**. Cobertura mínima
não discrimina; quem discrimina é a profundidade, medida na rubrica, sobretudo em **C2
(tratamento das tensões)** e **C6 (riscos de medição)**.

## 3. Comparações

### 3.1 OpenSpec (A) vs agentes próprios (B)

Emparelhando por grau de colaboração (A1↔B1, A2↔B2, A3↔B3):

| Colaboração | OpenSpec (A) | Agentes próprios (B) | Vencedor | Custo em tokens |
|-------------|--------------|----------------------|----------|-----------------|
| single-role | A1 = 53 | B1 = 49 | **A (OpenSpec) +4** | A mais caro (6.74 vs 4.50) |
| multi-role  | A2 = 55 | B2 = 56 | **B (agentes) +1** | B mais barato (4.63 vs 6.87) |
| multi-agente| A3 = 63 | B3 = 68 | **B (agentes) +5** | B mais barato (11.51 vs 12.73) |

Duas leituras se combinam aqui:

1. **OpenSpec só compensa quando há um único papel.** Em A1 vs B1, a estrutura do spec deu a um
   agente solo a disciplina que ele não teria sozinho — rastreabilidade de requisitos, seções
   fixas, pré-condições explícitas — e isso valeu +4 pontos mesmo custando mais tokens. É o
   único par onde o OpenSpec paga o próprio custo.
2. **A partir de dois papéis, a colaboração substitui o arcabouço — e os agentes próprios ficam
   à frente, mais baratos.** B2≈A2 (empate técnico, B mais barato) e B3>A3 por 5 pontos com
   menos tokens. Ou seja: quando já existe crítica entre papéis, a estrutura extra do OpenSpec
   deixa de agregar e passa a cobrar tokens sem retorno proporcional. O multi-agente de agentes
   próprios (B3) é o melhor plano do conjunto **e** mais barato que seu par OpenSpec (A3).

Em resumo: **OpenSpec é um andaime para o agente solo; com colaboração, vira peso morto.**

### 3.2 Designs A entre si (A1 → A2 → A3) — família OpenSpec

- **A1 → A2 (+2 pontos, custo ~igual):** o multi-role melhora C6 (riscos de medição); a voz do
  "Backend/Integration Engineer" eleva o acoplamento do pool a risco de primeira ordem. Ganho
  pequeno e quase de graça.
- **A2 → A3 (+8 pontos, ~1.85× tokens, ~1.7× tempo):** o isolamento em subagentes puxa
  C1/C2/C3/C5 para 5. A3 traz matriz de rastreabilidade (teste→capability) e seis Open Questions
  — a não-linearidade QA(80 conexões)→prod(160) e a dependência de `C_max`. A crítica isolada
  rendeu nuance que o debate encenado de A2 não produziu.

A3 é o teto da família OpenSpec, mas é o **braço menos eficiente por token de todos** (4.9):
pagou o preço mais alto do conjunto e mesmo assim ficou abaixo de B3.

### 3.3 Designs B entre si (B1 → B2 → B3) — família agentes próprios

- **B1 → B2 (+7 pontos, custo quase idêntico: 4.50 → 4.63 tokens):** o melhor negócio de todo o
  experimento. Só trocar single-role por multi-role — sem spec, praticamente de graça — levou o
  braço de 70% para 80%. A colaboração entre papéis entrega sozinha a profundidade (C2/C6) que
  faltava a B1.
- **B2 → B3 (+12 pontos, ~2.5× tokens, ~2.9× tempo):** o isolamento multi-agente entrega o
  melhor plano do conjunto. B3 é o único com: modelo mental dos gargalos ordenado (D0),
  coordinated omission nomeado + `dropped_iterations` como gate, pinning de Virtual Threads, HPA
  lag no spike, bureau-timeout distinto de bureau-5xx (dois cenários E2E) e idempotência
  concorrente como teste dedicado (G, a única exceção justificada ao modelo aberto). Profundidade
  genuína, não volume — e sem precisar do OpenSpec para chegar lá.

Observação: **B3 lidera em quase tudo, menos clareza (C8=4)**, onde A1 (o braço OpenSpec
single-role, mais barato dos A) marca 5. Mais profundidade não comprou mais acionabilidade —
ela saturou cedo.

## 4. Qualidade × gasto de tokens — vale a pena?

Três faixas de custo, com retorno claramente decrescente:

1. **Barato (B1/B2, ~4.5 tokens — agentes próprios):** B2 entrega 80% da qualidade do topo por
   ~40% dos tokens de B3. É o patamar de maior eficiência (12.1 qualidade/token). B1 mostra o
   outro lado: no mesmo custo, faltar um segundo papel custa ~10 pontos percentuais.
2. **Médio (A1/A2, ~6.7–6.9 tokens — OpenSpec):** 76–79%. Pior eficiência que a faixa barata
   para qualidade semelhante — o OpenSpec cobra tokens a mais para chegar ao mesmo lugar que os
   agentes próprios no multi-role. Só se justifica no single-role, onde A1 de fato supera B1.
3. **Caro (A3/B3, ~11.5–12.7 tokens — multi-agente):** 90–97%, o topo da qualidade, mas a pior
   eficiência por token (4.9–5.9). O delta B2→B3 custa ~2.5× os tokens por +17 pontos percentuais.

**Veredito:** o multi-agente isolado **vale seletivamente, não como padrão**. Os ~17 pontos de
B3 sobre B2 concentram-se em riscos não óbvios (coordinated omission, pinning de VT, HPA,
idempotência concorrente) — justamente o tipo de item que, se escapar num plano de performance,
gera *falso verde* e custa caro na produção. Para um alvo de alto risco como Veridia (SLA sem
folga, pool compartilhado, pico Kafka noturno), esse seguro paga. Para planos de rotina/baixo
risco, pagar 2–3× tokens por refinamento marginal não compensa.

E, cruzando com o eixo família: **gastar os tokens extras do OpenSpec não ajudou na ponta cara**
— A3 custou mais que B3 e entregou menos. Quando o orçamento for alto, ele rende mais em um
segundo/terceiro papel (colaboração) do que no arcabouço do spec.

## 5. Recomendações para o time

1. **Padrão de trabalho: multi-role com agentes próprios (estilo B2).** Melhor relação
   qualidade/token do conjunto (80% da qualidade do topo por ~1/3 dos tokens de B3). Deve ser o
   default para a maioria dos planos de teste/design.
2. **Reserve o multi-agente isolado (estilo B3) para alvos de alto risco ou para o plano de
   referência.** Quando a falha é cara e os gargalos são acoplados/não óbvios (SLA apertado,
   recursos compartilhados, picos), o custo extra compra riscos de medição que os braços baratos
   não enxergam. Use-o para o "plano canônico" de base, não a cada iteração.
3. **Use OpenSpec apenas quando for rodar com um único papel.** Foi o único regime em que ele
   pagou o próprio custo (A1 > B1). Com dois ou mais papéis, os agentes próprios alcançam
   qualidade igual ou superior por menos tokens — o arcabouço do spec vira cerimônia.
4. **Não combine OpenSpec com multi-agente como rotina.** A3 foi o braço mais caro e menos
   eficiente de todos, e ainda ficou abaixo de B3. Se o orçamento comportar o regime caro,
   invista em colaboração/isolamento (mais papéis), não na estrutura de spec.
5. **Fluxo híbrido para itens de alto risco:** gerar o plano de referência com multi-agente de
   agentes próprios (B3) **uma vez** e usar multi-role (B2) nas revisões/variações seguintes. O
   custo alto é pago uma vez; o barato cobre a cadência.
6. **Acrescente um passo final de edição de clareza.** O braço mais caro (B3) não foi o mais
   claro; A1 foi. Vale enxugar o plano de B3 para que sua profundidade chegue ao QA com a
   acionabilidade de A1 — ganho barato que nenhuma técnica entregou sozinha.
7. **Aposente o checklist de presença como critério de ranqueamento.** Saturou (6×8/8); serve só
   como porta de entrada ("cobriu o mínimo?"). Para discriminar qualidade, use a rubrica
   ponderada com peso em C2 (tensões) e C6 (riscos de medição) — onde os braços realmente se
   separaram.

## 6. Ressalvas da avaliação

- **Atribuição de família.** A = OpenSpec e B = agentes próprios, conforme o desenho do
  experimento. Os títulos internos dos arquivos B mencionam "Spec", mas isso é rótulo do método,
  não uso de OpenSpec; a análise seguiu a atribuição correta de família.
- **Pontuação não-cega e de um só avaliador.** Há viés possível; o ideal (como os templates
  pedem) seria avaliação cega por terceiro. Trate as notas como ordenação robusta, não medida
  exata — diferenças de 1–2 pontos (ex.: A2 vs B2) estão dentro da margem de ruído; as de faixa
  (barato vs caro) são consistentes.
- **Tokens em unidade relativa** conforme informado; o tempo foi usado só como contexto
  secundário. O custo-benefício se apoia na razão qualidade/token, não no tempo.
- **Nenhum plano foi executado.** A avaliação é do planejamento, não de resultados reais de
  carga; "qualidade" aqui é qualidade de design, não validação empírica dos thresholds.
