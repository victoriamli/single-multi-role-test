# Cenário do experimento

**Este enunciado é idêntico para os seis braços (A1, A2, A3, B1, B2, B3).**

## Tarefa

Analisar a aplicação alvo descrita em [`target-app.md`](./target-app.md) e produzir um
**plano de testes** usando **K6**, cobrindo:

- **E2E (end-to-end):** validar os fluxos de negócio críticos de ponta a ponta.
- **Carga (load):** comportamento sob a carga esperada em produção.
- **Volume:** comportamento com grande quantidade de dados/mensagens acumuladas.
- **Stress:** comportamento acima da capacidade esperada, até o ponto de ruptura.

O objetivo é **garantir o funcionamento da aplicação** sob condições reais e extremas,
identificando riscos, gargalos e critérios de aceite mensuráveis.

## O que entregar

Um plano de testes seguindo **exatamente** o formato de
[`../output-format.md`](../output-format.md). O plano deve ser **específico para o alvo**
descrito — referenciar seus fluxos, endpoints, SLAs e pontos de tensão reais, não
recomendações genéricas.

## O que NÃO fazer

- Não implementar os scripts K6 completos. O foco é o **planejamento**: estratégia, tipos
  de teste, modelagem de carga, thresholds, dados de massa, cenários e critérios de aceite.
  Trechos ilustrativos curtos de configuração K6 (ex.: `options`/`thresholds`) são
  permitidos para explicitar decisões, mas não o script inteiro.
- Não propor mudanças na arquitetura do alvo. O plano testa o alvo como ele é.
- Não investigar código real de outros projetos. Todo o conhecimento sobre o alvo está em
  `target-app.md`.

## Restrição de organização (pano de fundo, não é o foco)

A aplicação de testes mantém os scripts K6 em uma **pasta interna** do próprio projeto
(diferente de buscá-los em um repositório externo). Isso é uma decisão de organização já
tomada e **não** é objeto do planejamento — mencione apenas se afetar a estratégia de teste.
