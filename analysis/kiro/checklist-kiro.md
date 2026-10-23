# Checklist de qualidade do planejamento (template em branco)

Cópia zerada para reuso em outra rodada/cenário. Avaliação factual e de alto nível, preenchida
lendo o **`design.md`** de cada braço.

Marcação **Sim / Não**. Se houver ressalva, anotar em **Obs.** de forma concisa. O critério é:
o item está presente e tratado no design? — não "quão bem", para evitar subjetividade.

## Os 8 critérios

1. **Cobriu os fluxos críticos do alvo?** Os fluxos de negócio essenciais foram contemplados.
2. **Cobriu os tipos de teste relevantes?** Os tipos aplicáveis (carga, stress, volume, etc.)
   foram escolhidos e os descartados, justificados.
3. **Critérios de aceite mensuráveis?** Metas numéricas e verificáveis, não vagas.
4. **Identificou os principais riscos/gargalos do alvo?** Os pontos de ruptura foram nomeados.
5. **Tratou a estratégia de dados de massa?** Como gerar/obter a massa e seu impacto.
6. **Apontou riscos de medição?** Onde o teste poderia medir a coisa errada / dar falso resultado.
7. **Levantou o que precisa de decisão humana?** Pendências/ambiguidades explicitadas.
8. **O plano é acionável?** Um QA conseguiria executar a partir do design.

## Planilha

> Preenchido por: Kiro (avaliação não-cega — a técnica de cada braço é inferível pelo próprio
> texto do design). Critério = presença/tratamento do item, não profundidade.

| # | Critério | A1 | A2 | A3 | B1 | B2 | B3 |
|---|----------|----|----|----|----|----|----|
| 1 | Fluxos críticos         | Sim | Sim | Sim | Sim | Sim | Sim |
| 2 | Tipos de teste          | Sim | Sim | Sim | Sim | Sim | Sim |
| 3 | Aceite mensurável       | Sim | Sim | Sim | Sim | Sim | Sim |
| 4 | Riscos/gargalos         | Sim | Sim | Sim | Sim | Sim | Sim |
| 5 | Dados de massa          | Sim | Sim | Sim | Sim | Sim | Sim |
| 6 | Riscos de medição       | Sim | Sim | Sim | Sim | Sim | Sim |
| 7 | Decisão humana          | Sim | Sim | Sim | Sim | Sim | Sim |
| 8 | Acionável               | Sim | Sim | Sim | Sim | Sim | Sim |
| **Sim (total)** |     | 8 | 8 | 8 | 8 | 8 | 8 |

Como antecipado na nota de uso, o checklist **saturou**: os seis braços atingem 8/8. Todos
cobrem os 5 fluxos, selecionam/justificam tipos de teste, trazem thresholds numéricos, nomeiam
gargalos (bureau, pool Oracle), tratam massa (unicidade vs idempotência), listam riscos de
medição e abrem pendências para decisão humana. A diferença entre braços **não** está na
presença, e sim na profundidade — capturada na rubrica, não aqui.

## Observações por braço

- **A1 (single-role, agentes próprios):** 8/8. Matriz D8 completa e autossuficiente. Item 6:
  riscos de medição presentes, porém como lista mais curta (modelo fechado, isolamento, seed
  ausente) sem alguns itens finos (pinning de Virtual Threads, HPA lag) que aparecem em B3.
- **A2 (multi-role):** 8/8. A seção "resumo do debate" dá rastro de por que cada decisão foi
  tomada; itens 4 e 6 ficam bem servidos pela voz do "Backend/Integration Engineer".
- **A3 (multi-agente isolado + OpenSpec):** 8/8. Único com **matriz de rastreabilidade**
  teste→fluxo→capability (D9) e mais Open Questions (Q1–Q6). Item 4 nota o risco de VT mascarar
  saturação.
- **B1 (Spec single-role):** 8/8. Rastreabilidade RQ-* desde o topo. Item 3 inclui mix de
  veredito na carga (80/12/8%), detalhe que os demais não explicitam. Lista de riscos de
  medição um pouco mais enxuta (6 itens).
- **B2 (Spec multi-role):** 8/8. Decisões separam premissas de ambiente (D0: RQ-ENV/OBS) dos
  thresholds (D4), o que reforça o item 8 (acionável). Item 6 com 7 riscos bem nomeados.
- **B3 (Spec multi-agente):** 8/8. Mais profundo em todos os itens. Item 4: "modelo mental do
  alvo" (D0) ordena os gargalos; item 6 tem 9 riscos, incluindo coordinated omission explícito,
  pinning de VT e HPA lag. Único a tratar idempotência concorrente como teste próprio (G).

## Nota de uso

O checklist mede **presença**, não profundidade — tende a saturar (vários braços 8/8). Serve
para garantir cobertura mínima e flagrar lacunas, não para ranquear qualidade. Para discriminar
qualidade entre braços, usar a comparação cega dos `design.md`.
