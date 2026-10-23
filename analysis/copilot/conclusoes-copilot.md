# Conclusões (revisadas com custo e tempo)

Atualizado em 2026-10-23, incorporando qualidade (rubrica), custo em tokens e tempo de execução.

## 1) Base de comparação

| Design | Qualidade (0-70) | Tokens | Tempo | Qualidade/tokens | Qualidade/min |
|---|---:|---:|---:|---:|---:|
| A1 | 56 | 6.74 | 7m04 | 8.31 | 7.93 |
| A2 | 56 | 6.87 | 7m15 | 8.15 | 7.72 |
| A3 | 63 | 12.73 | 12m22 | 4.95 | 5.09 |
| B1 | 55 | 4.50 | 4m34 | 12.22 | 12.04 |
| B2 | 56 | 4.63 | 4m37 | 12.10 | 12.13 |
| B3 | 66 | 11.51 | 13m25 | 5.73 | 4.92 |

Leitura direta:
- **Eficiência máxima (custo/tempo):** `B1` e `B2`.
- **Qualidade máxima absoluta:** `B3`.
- **Pior relação custo-benefício relativa:** `A3` (qualidade boa, mas mais caro/lento que alternativas próximas ou melhores).

## 2) Ganho incremental por estratégia

### No braço A
- **A1 -> A2:** praticamente zero ganho de qualidade (56 -> 56) com mais custo e tempo.
- **A2 -> A3:** ganho real de qualidade (+7), mas com salto grande de custo (+5.86 tokens) e tempo (+5m07).

### No braço B
- **B1 -> B2:** ganho pequeno (+1), com custo/tempo quase iguais.
- **B2 -> B3:** maior salto de qualidade (+10), com custo (+6.88) e tempo (+8m48) significativamente maiores.

## 3) Conclusão A versus B (com eficiência)

- **Braço B domina em eficiência:** para qualidade equivalente (56), `B2` usa ~33% menos tokens e ~36% menos tempo que `A1/A2`.
- **Braço B também vence no topo:** `B3` entrega melhor qualidade total (66) com menos tokens que `A3` (11.51 vs 12.73), embora demore um pouco mais.
- **Conclusão geral:** com custo e tempo na conta, **B é superior a A** tanto no modo econômico quanto no modo premium.

## 4) Melhor estratégia, dependendo do objetivo

1. **Se o objetivo principal for eficiência (menor custo/tempo com boa qualidade):** usar **`B2`**.
2. **Se o objetivo principal for qualidade máxima do plano:** usar **`B3`**.
3. **Se quiser estratégia em duas etapas (melhor custo-benefício prático):**
   - gerar baseline com **`B2`**;
   - elevar para **`B3`** apenas em iniciativas críticas (go/no-go, release sensível, auditoria).

## 5) Recomendação final

A estratégia com melhor equilíbrio global é **`B2` como padrão operacional + `B3` sob demanda**.

Motivo:
- `B2` mantém qualidade alta (56/70) com o menor custo/tempo entre os planos dessa faixa;
- `B3` concentra os ganhos de rigor que realmente diferenciam (observabilidade mandatória, guarda anti-omission, contenção bloqueante), mas com custo premium.

