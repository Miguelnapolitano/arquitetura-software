# Observações — Experimento 3: Microkernel (faturamento por plugins)

## Condição alterada

Em `plugins/frete.py`, a constante `ISENCAO_ACIMA_DE` valia `5_000.00` (frete grátis acima de R$5 mil). Alterei para `50_000.00`. **Nenhuma linha do núcleo (`nucleo.py`) foi alterada** — mudei apenas a regra interna de um plugin já registrado.

## O que a saída revelou

O diff entre `saida-antes.txt` e `saida-depois.txt` mostra exatamente as faturas afetadas pela nova condição:

| Fatura | Valor bruto | Frete antes | Frete depois | Total antes → depois |
| --- | --- | --- | --- | --- |
| #1001 — SP | R$12.000 | R$0,00 (isenta) | **R$15,00** | 13.400,00 → 13.415,00 |
| #1002 — RJ | R$8.400 | R$0,00 (isenta) | **R$22,00** | 10.080,00 → 10.102,00 |
| #1003 — SP | R$75.000 | R$0,00 | R$0,00 (ainda isenta) | 84.000,00 (igual) |
| #1004 — MG | R$45.000 | R$0,00 (isenta) | **R$25,00** | 53.100,00 → 53.125,00 |

A mudança ficou visível na saída em três pontos: a linha `Frete:` de cada emissão, o `TOTAL` recalculado e até a mensagem de e-mail enviada pelo plugin de notificação, que passou a anunciar os novos totais. A fatura #1003 não mudou porque, mesmo com o novo piso de R$50 mil, seu valor bruto continua acima dele.

## Responsabilidade arquitetural relacionada

1. **Contrato estável do núcleo:** o núcleo só conhece a categoria "frete" e chama `processar(fatura, resultado)` na ordem definida por `ORDEM_CATEGORIAS`. Ele não sabe qual é a tabela de valores nem o limite de isenção — esses detalhes pertencem ao plugin.
2. **Extensibilidade sem tocar o núcleo:** toda a mudança de comportamento veio de editar uma extensão isolada; o registro (`Plugins ativos: {...}`) permaneceu idêntico e nenhum outro plugin foi impactado. É o princípio central do microkernel: política variável em plugins, mecanismo fixo no núcleo.
3. **Contribuição contextual das regras:** assim como cada imposto só contribui quando o estado do cliente se aplica, a regra de frete só zera o valor quando sua condição (valor bruto ≥ isenção) é verdadeira — a saída evidencia que cada plugin decide sozinho quando participa do cálculo.
