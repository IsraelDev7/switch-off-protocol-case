# ADR-0006 — Venda sem restrição geográfica, com exposição de VAT assumida

- **Status:** Aceito
- **Data:** 2026-08-09

## Contexto

O Payment Link do Stripe aceita restringir os países que podem comprar.
A escolha desse parâmetro tem consequência fiscal direta, porque a Smart
LABS vende **com o Stripe como processador, não como revendedor** — o
Stripe move o dinheiro, mas quem tem a obrigação tributária é a Smart
LABS (ver [ADR-0002](0002-stripe-payment-link.md)).

O ponto que importa:

- **Reino Unido** — serviços digitais vendidos a consumidor final por
  fornecedor estrangeiro exigem registro de VAT **desde a primeira
  venda**. Não existe limite de isenção como o que se aplica a empresas
  locais.
- **União Europeia** — mesma lógica, via regime OSS.
- **Estados Unidos** — a tributação de produto digital varia por estado
  e depende de limites de *economic nexus* que são altos. A exposição
  prática de um vendedor estrangeiro em volume inicial é próxima de zero.

O ADR-0002 já tinha registrado a União Europeia como gatilho para
reavaliar um *merchant of record*. O que não estava registrado com
clareza: **o gatilho não é volume, é o primeiro cliente britânico.**

## Decisão

O Payment Link é criado **sem restrição de país**. Qualquer pessoa, em
qualquer lugar, pode comprar.

Decisão tomada pelo dono do produto em 09/08/2026, depois de a exposição
fiscal ter sido apresentada e discutida.

## Alternativas consideradas

| Alternativa | Por que não foi escolhida |
|---|---|
| **Restringir aos EUA no lançamento** | Era a recomendação técnica: elimina a obrigação de VAT no Reino Unido e na UE enquanto o funil ainda está sendo validado, com exposição mínima nos EUA. Descartada por reduzir o mercado de um produto concebido para US **e** UK |
| **Merchant of record** (Paddle, Lemon Squeezy) | Assume a responsabilidade fiscal e recolhe o imposto no lugar da Smart LABS. Custa ~5% + $0,50 contra ~2,9% + $0,30. Continua sendo a saída natural se o volume no Reino Unido justificar |
| **Stripe Tax** | Calcula e cobra o imposto correto no checkout, mas **não** elimina a obrigação de registro e recolhimento — apenas produz o número certo. Vale ativar quando houver registro fiscal a declarar |

## Consequências

- Alcance máximo desde o primeiro dia, como o produto foi concebido.
- **A obrigação de registro de VAT no Reino Unido e na UE passa a existir
  na primeira venda para esses mercados**, e é da Smart LABS. Isso é uma
  exposição assumida conscientemente, não um descuido — está registrada
  aqui para que a decisão possa ser revista com informação.
- **Confirmar com contador** antes que o volume internacional cresça.
  Regularizar depois custa mais do que registrar antes.
- Enquanto não houver registro fiscal, `automatic_tax` fica desativado no
  Payment Link: cobrar imposto sem ter onde recolher seria pior que não
  cobrar.
- Revisitar quando surgir a primeira venda para o Reino Unido ou para a
  União Europeia — não quando o volume total crescer.
