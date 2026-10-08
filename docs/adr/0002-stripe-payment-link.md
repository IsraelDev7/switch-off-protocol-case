# ADR-0002 — Stripe Payment Link, não Checkout Session nem revendedor

- **Status:** Aceito
- **Data:** 2026-08-07

## Contexto

A landing page é **estática**: HTML, CSS e um arquivo JS, servidos pelo
Vercel. Não existe backend, e introduzir um só para cobrar $37 adicionaria
uma superfície inteira de manutenção, deploy e falha.

O produto precisa cobrar cartão, Apple Pay e Google Pay, emitir recibo e
tratar imposto, com um item opcional de order bump de $9.

## Decisão

**Stripe Payment Link** — uma URL fixa, colada no `href` dos CTAs.

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Checkout Session** via API | Exige um servidor para criar a sessão a cada clique. Trocar uma página estática por uma aplicação com backend, para um produto único de preço fixo, é custo sem contrapartida |
| **Lemon Squeezy / Paddle** (revendedor legal) | Assumem o papel de *merchant of record* e resolvem imposto internacional — vantagem real. Custam ~5% + $0,50 contra ~2,9% + $0,30 do Stripe. No volume atual, a diferença não paga a perda de controle sobre o checkout |
| Gateway nacional | Público-alvo é US/UK; a experiência de checkout e a moeda estariam erradas |

## Consequências

- Nenhum servidor a manter. O deploy do Vercel continua sendo cópia de
  arquivos estáticos.
- O preço fica **fixo na URL**: promoção ou teste A/B de preço exige
  criar outro Payment Link, não um parâmetro.
- A entrega do arquivo **não** é resolvida pelo Stripe. Ele cobra e
  dispara `checkout.session.completed`; quem entrega é a automação do
  N8n. Essa dependência é da Etapa 4 do funil.
- **Revisitar se:** a operação começar a vender para a União Europeia
  (o IVA incide desde o primeiro euro e um *merchant of record* passa a
  valer os 2 pontos percentuais), ou se passar de ~300 vendas/mês.
