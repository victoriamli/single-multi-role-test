# Conclusões — avaliação dos 6 planos de teste (Veridia)

Documento de fechamento da análise dos braços A1–A3 e B1–B3. Baseado na leitura dos seis
`design.md`, do alvo (`context/target-app.md`) e do prompt (`context/prompt.md`), com as notas
consolidadas em `checklist-kiro.md` e `rubric-kiro.md`.

## 1. Como li cada braço

A técnica de cada braço é inferível pelo próprio texto do design (menções a "single-role",
"debate multi-role", "subagentes isolados", OpenSpec/`requirements.md`):

| Braço | Técnica | Família |
|-------|---------|---------|
| A1 | Single-role, agentes próprios | A = agentes próprios |
| A2 | Multi-role (debate encenado em contexto único) | A |
| A3 | Multi-agente (subagentes isolados) + OpenSpec | A |
| B1 | Spec single-role (OpenSpec) | B = OpenSpec/Spec |
| B2 | Spec multi-role (OpenSpec) | B |
| B3 | Spec multi-agente (subagentes isolados, OpenSpec) | B |

O eixo que muda dentro de cada família é o **número de papéis/isolamento**: 1 papel →
multi-role no mesmo contexto → multi-agente com contexto isolado. O eixo entre famílias é
**agentes próprios (A) vs OpenSpec (B)**.

## 2. Placar

Rubrica ponderada (máx. 70) e eficiência (qualidade por token, usando os tokens informados):

| Braço | Rubrica | % (de 70) | Tokens | Tempo | Qualidade/token |
|-------|---------|-----------|--------|-------|-----------------|
| B3 | 68 | 97% | 11.51 | 13m25s | 5.9 |
| A3 | 63 | 90% | 12.73 | 12m22s | 4.9 |
| B2 | 56 | 80% | 4.63 | 4m37s | **12.1** |
| A2 | 55 | 79% | 6.87 | 7m15s | 8.0 |
| A1 | 53 | 76% | 6.74 | 7m04s | 7.9 |
| B1 | 49 | 70% | 4.50 | 4m34s | 10.9 |

Duas leituras diferentes dependendo do que se otimiza:

- **Qualidade absoluta:** B3 > A3 > B2 > A2 > A1 > B1.
- **Qualidade por token:** B2 > B1 > A2 > A1 > B3 > A3.

O checklist de presença (8 critérios) saturou: **todos** os braços deram 8/8. Cobertura mínima
não discrimina aqui; quem discrimina é a profundidade, medida na rubrica.

## 3. Comparações pedidas

### 3.1 Designs A vs designs B

Emparelhando por técnica (A1↔B1, A2↔B2, A3↔B3), **a família B (OpenSpec) supera a A em dois dos
três pares** e, de quebra, é mais barata:

- Multi-role: **B2 (56) > A2 (55)** com ~1/3 menos tokens (4.63 vs 6.87).
- Multi-agente: **B3 (68) > A3 (63)** com menos tokens (11.51 vs 12.73).
- Single-role: **B1 (49) < A1 (53)** — a única inversão.

O que o OpenSpec adiciona quando há mais de um papel: rastreabilidade `RQ-*`→teste→fluxo/tensão,
e separação explícita de pré-condições de ambiente (seed de 400M, stub injetável) como itens
bloqueantes. Isso eleva C7 (realismo) e C8 (acionável) e organiza C2/C6.

A inversão no single-role é o achado mais útil: **OpenSpec sozinho, sem um segundo papel que
explore a estrutura, vira cerimônia.** B1 gastou orçamento montando o arcabouço `RQ-*` e saiu
mais raso que A1 — que, livre da estrutura, foi direto ao ponto e ainda ficou mais claro (A1
lidera C8). Ou seja: a estrutura do Spec só paga quando há colaboração/crítica para preenchê-la.

### 3.2 Designs A entre si (A1 → A2 → A3)

Progressão limpa de qualidade com custo crescente:

- **A1 → A2 (+2 pontos, custo ~igual):** o debate multi-role melhora C6 (riscos de medição) —
  a voz do "Backend/Integration Engineer" traz o acoplamento do pool como risco de primeira
  ordem. Ganho pequeno mas essencialmente de graça.
- **A2 → A3 (+8 pontos, ~1.85× tokens, ~1.7× tempo):** o isolamento em subagentes é o que puxa
  C1/C2/C3/C5 para 5. A3 é o único A com matriz de rastreabilidade (teste→capability) e seis
  Open Questions, incluindo a não-linearidade QA(80 conexões)→prod(160) e a dependência de
  `C_max`. A crítica isolada rendeu nuance que o debate encenado de A2 não produziu.

A3 é o teto da família A, mas é também o braço **menos eficiente por token de todos** (4.9):
pagou o preço mais alto e não alcançou B3.

### 3.3 Designs B entre si (B1 → B2 → B3)

A progressão mais informativa do conjunto, porque separa o efeito "número de papéis" do efeito
"OpenSpec":

- **B1 → B2 (+7 pontos, custo quase idêntico: 4.50 → 4.63 tokens):** este é o melhor negócio de
  todo o experimento. Só trocar single-role por multi-role, dentro do mesmo arcabouço Spec e
  praticamente de graça, levou o braço de 70% para 80%. O multi-role "ativa" a estrutura que o
  single-role (B1) só preencheu formalmente.
- **B2 → B3 (+12 pontos, ~2.5× tokens, ~2.9× tempo):** o isolamento multi-agente entrega o
  melhor plano do conjunto. B3 é o único com: modelo mental dos gargalos ordenado (D0),
  coordinated omission nomeado + `dropped_iterations` como gate de teste, pinning de Virtual
  Threads, HPA lag no spike, bureau-timeout distinto de bureau-5xx (dois cenários E2E), e
  idempotência concorrente como teste dedicado (G, a única exceção justificada ao modelo
  aberto). É profundidade genuína, não volume.

Observação importante: **B3 lidera em quase tudo menos clareza (C8=4)**, onde A1 (o mais barato
dos A) marca 5. Mais profundidade não comprou mais acionabilidade — ela saturou cedo.

## 4. Qualidade × gasto de tokens — vale a pena?

Três faixas de custo, com retorno decrescente claro:

1. **Barato (B1/B2, ~4.5 tokens):** B2 entrega 80% da qualidade do melhor plano por ~40% dos
   tokens de B3. É o patamar de maior eficiência (12.1 qualidade/token). B1 mostra que, no
   mesmo custo, falta de um segundo papel custa 10 pontos percentuais.
2. **Médio (A1/A2, ~6.7–6.9 tokens):** entrega 76–79%. Pior eficiência que a faixa barata (B)
   para qualidade semelhante — os agentes próprios custam mais tokens que o Spec para chegar
   ao mesmo lugar no multi-role. A2 só se justifica sobre B2 se houver razão para evitar
   OpenSpec.
3. **Caro (A3/B3, ~11.5–12.7 tokens):** 90–97%. O topo da qualidade, mas eficiência por token
   é a pior (4.9–5.9). O delta B2→B3 custa ~2.5× os tokens por +17 pontos percentuais.

**Veredito de custo-benefício:** o multi-agente isolado **vale a pena seletivamente**, não como
padrão. Os ~17 pontos extras de B3 sobre B2 concentram-se em riscos não óbvios (coordinated
omission, pinning de VT, HPA, idempotência concorrente) — exatamente o tipo de item que, se
passar batido num plano de performance, gera *falso verde* e custa caro depois. Para um alvo de
alto risco como Veridia (SLA sem folga, pool compartilhado, pico noturno), esse seguro paga.
Para planos de rotina/baixo risco, pagar 2–3× tokens por refinamento marginal não se justifica.

## 5. Recomendações para o time

1. **Padrão de trabalho: Spec multi-role (estilo B2).** Melhor relação qualidade/token do
   conjunto (80% da qualidade por ~1/3 dos tokens do topo). Deve ser o default para a maioria
   dos planos de teste/design.
2. **Reserve o multi-agente isolado (estilo B3) para alvos de alto risco ou de referência.**
   Quando a falha é cara e os gargalos são acoplados/não óbvios (sistemas com SLA apertado,
   recursos compartilhados, picos), o gasto extra compra riscos de medição que os braços baratos
   não enxergam. Use-o para o "plano canônico" que servirá de base, não para cada iteração.
3. **Evite single-role + OpenSpec (B1).** É o pior dos mundos no experimento: paga a cerimônia
   da estrutura sem a colaboração que a torna útil, e fica atrás do single-role mais simples e
   barato (A1). Se for usar um único papel, use o mais enxuto (estilo A1); se vai investir em
   OpenSpec, invista também em pelo menos dois papéis.
4. **Prefira OpenSpec a agentes próprios quando houver ≥2 papéis.** B2>A2 e B3>A3, com custo
   menor. A rastreabilidade `RQ-*` e as pré-condições de ambiente explícitas elevam a
   acionabilidade sem inflar tokens.
5. **Fluxo híbrido sugerido para itens de alto risco:** gerar o plano de referência com
   multi-agente isolado (B3) **uma vez**, e usar multi-role (B2) para as revisões/variações
   subsequentes. Assim o custo alto é pago uma vez, e o barato cobre a cadência.
6. **Passe a profundidade por uma etapa de edição de clareza.** O braço mais caro (B3) não foi
   o mais claro; A1 foi. Vale um passo final de enxugamento para que a profundidade de B3 chegue
   ao QA com a acionabilidade de A1 — ganho barato que nenhuma técnica entregou sozinha.
7. **Aposente o checklist de presença como critério de ranqueamento.** Ele saturou (6×8/8);
   serve só como porta de entrada ("cobriu o mínimo?"). Para discriminar qualidade, use a
   rubrica ponderada, com peso em C2 (tensões) e C6 (riscos de medição), que foi onde os braços
   realmente se separaram.

## 6. Ressalvas da avaliação

- **Pontuação não-cega e de um só avaliador.** As técnicas estavam identificáveis no texto e um
  único avaliador pontuou; há viés possível. O ideal (como os próprios templates pedem) seria
  avaliação cega por terceiro. Trate as notas como ordenação robusta, não como medida exata —
  as diferenças de 1–2 pontos (ex.: A1 vs A2) estão dentro da margem de ruído; as de família
  (barato vs caro) são consistentes.
- **Tokens em unidade relativa** conforme informado; o tempo foi usado só como contexto
  secundário, como solicitado. As conclusões de custo-benefício se apoiam na razão
  qualidade/token, não no tempo.
- **Nenhum plano foi executado.** A avaliação é do planejamento, não de resultados reais de
  carga; "qualidade" aqui é qualidade de design, não validação empírica dos thresholds
  propostos.
