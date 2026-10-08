# ADR-0007 — Entrega em EPUB e PDF, por link assinado

- **Status:** Aceito
- **Data:** 2026-08-09

## Contexto

O produto foi diagramado como PDF A5 de 40 páginas, com tipografia
trabalhada. O pedido era entregar com experiência de leitura semelhante à
de um Kindle.

**PDF não faz isso, e nenhuma plataforma conserta.** PDF tem página de
tamanho fixo: no celular, a leitora dá zoom e arrasta. A experiência de
leitor vem de conteúdo *reflowable* — EPUB — onde o texto se readapta à
tela e a leitora escolhe fonte, corpo e tema.

Para este produto a diferença é mais que conforto. É um livro sobre sono,
lido à noite, no celular, na cama. Tema escuro e corpo ajustável são a
diferença entre ajudar a dormir e entregar uma página branca A5 às 23h —
exatamente o estímulo que o protocolo manda evitar.

## Decisão

Entregar **os dois formatos**: PDF para desktop e impressão, EPUB para
leitura em tela. Sem plataforma de leitura de terceiros e sem leitor
próprio: o EPUB abre no Apple Books, no Google Play Books e no Kindle,
que a leitora já tem e já sabe usar.

Hospedagem em **Cloudflare R2 com link assinado**.

## A fonte do EPUB é o book.html, não o manuscrito

Decisão que evitou um vazamento sério. `produto/manuscrito/` contém
material que **não é do produto**:

- `the-3am-reset-fundacao-do-produto.md` é o briefing de posicionamento,
  não conteúdo do livro — e cita um order bump de $17, divergindo do $9
  que foi decidido
- `the-switch-off-protocol-part-1-v2.md` tem uma seção
  `## Notas para o Israel (não vão no produto)`

Gerar o EPUB de `manuscrito/*.md` entregaria briefing estratégico e notas
privadas a quem pagou $37. `book.html` é a versão curada e aprovada, a
mesma que gera o PDF — usá-la como fonte garante paridade entre os dois
formatos e elimina a classe inteira de erro.

`scripts/build-epub.py` valida a cada execução que nenhum termo interno
aparece no resultado, mesmo que a fonte volte a mudar um dia.

## O que se perde, e por quê está certo

A diagramação A5 não sobrevive ao EPUB — reflowable e layout fixo são
incompatíveis por natureza. Número de página e cabeçalho corrido são
descartados: no EPUB virariam texto solto no meio do capítulo.

Fontes **não** são embutidas, de propósito. Leitores permitem à leitora
escolher tipografia e tema; embutir fonte briga com essa escolha
justamente no cenário de uso que importa.

## Alternativas consideradas

| Alternativa | Por que não |
|---|---|
| **BookFunnel** (~US$20/mês) | Resolve entrega, leitor web e marca d'água antipirataria de uma vez, e guia a leitora a abrir no app dela. Descartada por adicionar serviço externo e mensalidade a um funil que ainda não vendeu — reavaliar se a fricção de abrir EPUB aparecer no suporte |
| **Leitor próprio com epub.js** | Controle total e marca própria fim a fim, custo zero de licença. São dias de trabalho e código para manter, antes da primeira venda |
| **Só PDF** | Não atende ao pedido, e é ruim exatamente no dispositivo e no horário em que o produto é consumido |
| **Vercel, pasta pública** | URL fixa e adivinhável, sem revogação possível. Para produto pago, o link vaza uma vez e vaza para sempre |

## Consequências

- Duas saídas a gerar: `bash scripts/build-pdf.sh` e
  `python scripts/build-epub.py`. Ambas reproduzíveis, ambas fora do Git
  por serem artefato (ver [ADR-0005](0005-o-que-fica-fora-do-versionamento.md)).
- O EPUB é validado no momento da geração — container, XML bem-formado,
  retenção de texto e ausência de material interno. Um EPUB malformado
  só se manifestaria no aparelho de quem comprou.
- **A geração aborta se a capa falhar.** A renderização da capa via
  Chrome já falhou de forma intermitente e o script seguiu em silêncio,
  produzindo um EPUB de 22 KB sem capa. Falhar alto é melhor que
  entregar produto incompleto.
- O R2 exige bucket e credencial configurados uma vez. A credencial vive
  em variável de ambiente, nunca no repositório (ver `SECURITY.md`).
- Alguma fricção de suporte é esperada: parte das compradoras não sabe o
  que fazer com um arquivo `.epub`. O e-mail de entrega precisa ensinar,
  e isso é trabalho da Etapa 4.
