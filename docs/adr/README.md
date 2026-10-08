# Registro de decisões de arquitetura (ADR)

Cada arquivo aqui registra **uma decisão que foi difícil de tomar** e que
seria cara de reverter — junto com o que foi descartado e por quê.

O motivo de existirem: daqui a seis meses, a pergunta que aparece não é
"como isso funciona" (o código responde), é **"por que não fizemos do
outro jeito?"**. Sem registro, essa pergunta é redecidida do zero toda
vez, geralmente com menos informação do que se tinha no dia.

Um ADR é imutável. Quando uma decisão muda, escreve-se um ADR novo que
substitui o anterior, e o antigo passa a `Substituído por ADR-XXXX` —
o histórico do raciocínio é o ativo, não o arquivo mais recente.

| # | Decisão | Status | Data |
|---|---|---|---|
| [0001](0001-preco-37-usd.md) | Preço de $37 USD, não a régua brasileira 37–97 | Aceito | 2026-08-07 |
| [0002](0002-stripe-payment-link.md) | Stripe Payment Link, não Checkout Session nem revendedor | Aceito | 2026-08-07 |
| [0003](0003-sem-depoimento-fabricado.md) | Nenhum depoimento fabricado na landing | Aceito | 2026-08-07 |
| [0004](0004-sem-mecanismo-neologizado.md) | Sem mecanismo científico com nome inventado | Aceito | 2026-08-07 |
| [0005](0005-o-que-fica-fora-do-versionamento.md) | Produto pago e acervo bruto fora do Git | Aceito | 2026-08-08 |
| [0006](0006-venda-sem-restricao-geografica.md) | Venda sem restrição de país, com exposição de VAT assumida | Aceito | 2026-08-09 |
| [0007](0007-entrega-em-epub-e-pdf.md) | Entrega em EPUB e PDF, gerados do book.html, por link assinado | Aceito | 2026-08-09 |