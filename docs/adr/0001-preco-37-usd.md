# ADR-0001 — Preço de $37 USD, não a régua brasileira 37–97

- **Status:** Aceito
- **Data:** 2026-08-07

## Contexto

O produto é vendido em dólar, para EUA e Reino Unido, por uma persona que
se comunica em inglês. A referência de precificação que estava na mesa era
a régua "37–97" difundida no mercado brasileiro de infoprodutos.

Essa régua foi calibrada em **reais**. R$ 37 e US$ 37 não são o mesmo
produto na cabeça de quem compra: são faixas de mercado diferentes, com
expectativa de entrega diferente e concorrência diferente.

## Decisão

Preço de **$37 USD**, pagamento único.

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| Traduzir a régua 37–97 direto para dólar | Erro de categoria: a régua é uma convenção de um mercado específico, em outra moeda. Aplicá-la fora dele é copiar o número sem o raciocínio que o gerou |
| Preço menor (~$9–$17) | Abaixo do piso de credibilidade de low ticket em US/UK, e comprime demais a margem para tráfego pago |
| Preço maior (~$67+) | Sai da compra por impulso e passa a exigir uma página de vendas longa e prova social — que o ADR-0003 deliberadamente não teremos |

## Consequências

- $37 é o preço canônico de low ticket nesses mercados: alto o bastante
  para sustentar tráfego pago, baixo o bastante para decisão por impulso.
- O funil é **low ticket porta**, não centro de lucro. A métrica que
  importa é comprador qualificado entrando na esteira, não a receita
  bruta do $37 — o P.S. do livro aponta para o próximo produto.
- Se a operação abrir para a União Europeia, revisar junto com o
  ADR-0002: o IVA incide desde o primeiro euro e muda a margem real.
