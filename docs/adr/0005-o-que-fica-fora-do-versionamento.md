# ADR-0005 — Produto pago e acervo bruto fora do Git

- **Status:** Aceito
- **Data:** 2026-08-08

## Contexto

A pasta do projeto tinha **198 MB** quando o repositório foi criado.
Composição medida:

| Conteúdo | Peso | Natureza |
|---|---|---|
| Acervo bruto da persona persona | 153 MB | imagens e vídeo de origem |
| `diagramacao/out/` | 28 MB | PDFs, PNGs de página e áudios — saídas |
| Chunks de áudio de origem | 8,6 MB | material de montagem |
| Código, manuscrito, design system | 9,1 MB | fonte |

Dois riscos distintos se somam. **Peso:** o Git guarda cada versão de um
binário por inteiro, nunca o diff — um PDF de 1 MB revisado cinco vezes
custa 5 MB permanentes, e o histórico só cresce. **Exposição:**
`book.pdf`, `card.pdf`, `checklist.pdf` e os áudios são exatamente aquilo
que o cliente paga $37 para receber.

O agravante concreto: no dia em que o repositório foi criado, um token do
GitHub foi encontrado em texto plano dentro de `.claude/settings.local.json`
(gravado ali pela allowlist de permissões). Um token vazado com o produto
versionado entrega o produto inteiro.

## Decisão

Versiona-se a **fonte**. O artefato fica fora, sempre que houver caminho
de volta.

Fora do Git:

- `switch-off/diagramacao/out/` — artefatos de impressão e áudios
- `switch-off/a persona da marca - Persona/` — acervo bruto
- `switch-off/Firt Low Ticket - Audio/` — chunks de origem
- `.claude/settings.local.json` — pode conter credencial

Resultado: **9,1 MB versionados** em lugar de 198 MB.

## O caminho de volta é testado, não presumido

Ignorar artefato só se sustenta se ele puder ser regerado. `scripts/build-pdf.sh`
reconstrói os quatro PDFs a partir do HTML versionado. Verificação feita
no dia da decisão, comparando com os arquivos aprovados:

| Arquivo | Tamanho aprovado | Regerado | Bytes diferentes |
|---|---|---|---|
| `book.pdf` | 966.966 | 966.966 | 12 |
| `card.pdf` | 135.308 | 135.308 | — |
| `checklist.pdf` | 124.846 | 124.846 | — |
| `capa-final.pdf` | 460.673 | 460.673 | 14 |

As diferenças caem todas nos primeiros ~250 bytes de cada arquivo — a
região onde o Chrome grava `CreationDate`, `ModDate` e o ID do documento.
O conteúdo é idêntico; a divergência é timestamp, não conteúdo.

## Alternativas consideradas

| Alternativa | Por que foi descartada |
|---|---|
| Versionar tudo (198 MB) | Histórico inchado de forma irreversível, e o produto pago dentro do repositório |
| Git LFS | Resolve o peso, não a exposição — e adiciona dependência de infraestrutura e cota para um projeto de um desenvolvedor |
| Versionar só os áudios (~6,7 MB) | Foi considerado a sério, porque **áudio não é regerável** pelo repositório. Descartado por colocar produto pago no repositório em troca de um backup que uma pasta em nuvem resolve melhor |

## Consequências

- Repositório de 9,1 MB: clone rápido, histórico legível, diffs que fazem
  sentido.
- Um token vazado **não** entrega o produto.
- **Os áudios ficam sem backup versionado.** Não são reproduzíveis a
  partir do repositório — vieram de síntese de voz e montagem manual. O
  backup deles é responsabilidade externa e está registrado no README em
  *Ativos fora do versionamento*. Este é o custo consciente da decisão.
- `scripts/verify.sh` falha o build se qualquer arquivo acima de 5 MB
  entrar no rastreamento. A regra deixa de depender de disciplina.
