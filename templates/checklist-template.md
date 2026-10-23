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

> Indicar quem preencheu e se foi cego. (Autoavaliação do executor tem viés; preferir cego.)

| # | Critério | A1 | A2 | A3 | B1 | B2 | B3 |
|---|----------|----|----|----|----|----|----|
| 1 | Fluxos críticos         |  |  |  |  |  |  |
| 2 | Tipos de teste          |  |  |  |  |  |  |
| 3 | Aceite mensurável       |  |  |  |  |  |  |
| 4 | Riscos/gargalos         |  |  |  |  |  |  |
| 5 | Dados de massa          |  |  |  |  |  |  |
| 6 | Riscos de medição       |  |  |  |  |  |  |
| 7 | Decisão humana          |  |  |  |  |  |  |
| 8 | Acionável               |  |  |  |  |  |  |
| **Sim (total)** |     |  |  |  |  |  |  |

## Observações por braço

<!-- Anotar aqui ressalvas concisas quando um Sim/Não precisar de contexto.
Ex.: A1 — item 3: thresholds presentes mas sem p99 em um dos testes. -->

## Nota de uso

O checklist mede **presença**, não profundidade — tende a saturar (vários braços 8/8). Serve
para garantir cobertura mínima e flagrar lacunas, não para ranquear qualidade. Para discriminar
qualidade entre braços, usar a comparação cega dos `design.md`.
