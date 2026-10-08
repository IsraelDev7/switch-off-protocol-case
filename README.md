# The Switch-Off Protocol — estudo de caso

Sistema autônomo de venda e entrega de produto digital: landing estática,
checkout com consentimento registrado, webhook validado por assinatura, entrega
por link assinado e sequência de e-mails — sem servidor de aplicação para manter.

**Este repositório é a documentação de arquitetura.** O código de produção e os
ativos do produto são privados; o que está aqui são as decisões, os erros e as
medições.

---

## O sistema em uma passada

```
  Landing estática (CDN)
         │
         ▼
  Payment Link ──────────▶ consentimento registrado no checkout
         │
         ▼  webhook assinado (HMAC)
  Orquestrador (n8n, modo fila, 3 workers)
         │
         ├──▶ Entrega: 8 arquivos por link assinado  ──▶ 1,8 s até o e-mail
         │
         └──▶ Sequência de 7 e-mails (workflow separado)
```

Dois workflows, deliberadamente desacoplados. Falha na entrega é alguém que
pagou e não recebeu — obrigação contratual. Falha na sequência é uma pena.
Juntas, um erro de marketing poderia derrubar a obrigação.

**Medido ponta a ponta:** checkout → webhook validado → 8 arquivos entregues por
link assinado em **1,8 segundo** → sequência disparada em seguida.

---

## As decisões que definem a arquitetura

### Nenhum servidor de aplicação

A landing é HTML, CSS e um arquivo JS. Introduzir um backend só para cobrar um
preço fixo adicionaria superfície inteira de deploy, manutenção e falha.

A cobrança é um **Payment Link**: uma URL no `href` do CTA. O custo dessa escolha
está documentado — preço fica fixo na URL, teste A/B exige outro link — e o
gatilho para revisitar está escrito: venda para a UE (IVA desde o primeiro euro)
ou volume acima de ~300 vendas/mês.

[ADR-0002](docs/adr/0002-stripe-payment-link.md)

### A entrega não confia no navegador

Confirmar pagamento no retorno do navegador é confiar no cliente. A confirmação
válida é a que chega do provedor para o servidor, **com assinatura verificada**.

O webhook valida HMAC antes de qualquer efeito. Só então os links assinados são
gerados, com expiração.

### O que fica fora do versionamento, e o caminho de volta testado

A pasta de trabalho tinha 198 MB; o repositório versiona 9,1 MB. Artefato gerado
(`out/`) fica fora porque existe script que regenera.

Isso só se sustenta porque foi **verificado**: os PDFs regeraram com tamanho
idêntico byte a byte, divergindo apenas 12 e 14 bytes na região de `CreationDate`.
Ignorar artefato sem testar o caminho de volta é perder o artefato.

[ADR-0005](docs/adr/0005-o-que-fica-fora-do-versionamento.md) ·
[ADR-0007](docs/adr/0007-entrega-em-epub-e-pdf.md)

---

## Os erros que ensinaram mais que os acertos

### Resultado certo pelo motivo errado

Os testes de rejeição do webhook devolviam HTTP 500 e recusavam requisições
forjadas. **Parecia defesa funcionando.**

Ao ler a execução, a causa real era `Module 'crypto' is disallowed`. A validação
de assinatura **nunca tinha rodado**. O comportamento externo era indistinguível
de um sistema correto.

> Resultado certo pelo motivo errado é o modo de falha mais perigoso em
> segurança, porque não deixa sintoma.

O mesmo padrão apareceu em outros dois pontos: um `Authorization failed` que era
credencial **ausente** (dito literalmente no corpo do erro, não no rótulo), e um
DNS que "parecia" mal configurado e era cache.

### Documento e realidade divergindo em silêncio

Quatro ocorrências do mesmo erro, em disfarces diferentes:

| O documento afirmava | A realidade |
|---|---|
| checkout coleta consentimento | o Payment Link não coletava |
| unsubscribe funciona imediatamente | o marcador nunca era substituído |
| termos linkados no checkout | apontavam para domínio inexistente |
| três workers rodando | o Compose não os declarava |

Nenhum aparece em teste, log ou erro. Só aparece quando alguém compara as duas
coisas de propósito.

### Sondar o ambiente antes de escrever código para ele

Escrevi 200 linhas de workflow assumindo Node completo. O sandbox do Code node
bloqueia `require('crypto')`, não expõe Web Crypto e desliga o acesso a variáveis
de ambiente por padrão.

Um workflow de diagnóstico descartável respondeu tudo em dez minutos. Descobrir
um erro por vez teria custado a tarde.

**Corolário:** a sonda precisa perguntar a coisa certa. O primeiro diagnóstico
concluiu que o objeto de ambiente estava vazio porque testei `Object.keys()` — e
ele é um Proxy sem `ownKeys`. O acesso direto sempre funcionou.

### Guardrail estrutural vence guardrail por formato

Depois de um incidente com valores preenchidos no arquivo de modelo, a
verificação que entrou no CI **não** procura padrões de chave. Ela afirma algo
mais forte: *"um modelo não tem valores preenchidos"*.

Isso pega chave hexadecimal genérica, que nenhuma regex de credencial
reconheceria.

### Medir a restrição real antes de produzir

- Quatro opções de avatar pareciam ótimas a 1500 px. Simuladas em círculo a
  32, 56 e 110 px — o tamanho real de uso — **três perderam o que as tornava boas**
- Narração de 5,96 s em vídeo de 6,00 s. Só aparece com régua, e só apareceria
  depois do lip sync gerado, quando o custo já foi pago

### A pasta de trabalho nunca é a pasta de entrega

Três vezes o mesmo erro: 198 MB → 9,1 MB no versionamento; manuscrito com
anotações internas → fonte curada para o EPUB; 11 PDFs → 3 na entrega.

Sempre tratar "onde os arquivos estão" como "o que o produto é".

---

## Credenciais: autonomia de execução sem acesso ao segredo

O objetivo é que a automação **opere** as APIs, não que ela **veja** as chaves.
São coisas diferentes, e separá-las é o que torna o acesso seguro.

1. Credenciais vivem só em variável de ambiente, em arquivo que está no
   `.gitignore` **desde o commit inicial** — `.gitignore` não tem efeito retroativo
2. Os scripts leem a variável no momento da chamada e passam em **header HTTP**,
   nunca em linha de comando (apareceria em `ps` e no histórico do shell)
3. O diagnóstico mostra presença, tamanho e prefixo de 4 caracteres: suficiente
   para conferir se a chave certa carregou, insuficiente para reconstruí-la
4. Operação que muda estado exige confirmação explícita
5. A verificação roda no CI a cada push e **falha o build** se achar credencial

**A regra que não muda:** credencial nunca é colada em conversa. Se apareceu em
uma, considere comprometida e rotacione — o transcrito fica gravado em disco.

Um cuidado específico com ferramentas de agente está documentado em
[SECURITY.md](SECURITY.md): arquivos de comandos aprovados gravam o comando
**literalmente**, então um `curl` aprovado com token embutido deixa o token em
texto plano.

---

## Conformidade como decisão de engenharia

Duas decisões que normalmente ficam fora de repositório técnico, e não deveriam:

**Nenhum depoimento fabricado.** FTC (EUA) e ASA (Reino Unido) multam review
inventada, e sono é categoria fiscalizada. A seção que faz o mesmo trabalho
emocional sem inventar ninguém está documentada em
[ADR-0003](docs/adr/0003-sem-depoimento-fabricado.md).

**Nenhum mecanismo científico neologizado.** Inventar um nome de síndrome exigiria
o leitor aprender um conceito novo — contra o "zero curva de aprendizagem" que era
o diferencial do produto. [ADR-0004](docs/adr/0004-sem-mecanismo-neologizado.md)

---

## Documentação

| | |
|---|---|
| [`docs/adr/`](docs/adr/) | 7 Architecture Decision Records |
| [`docs/RETROSPECTIVA.md`](docs/RETROSPECTIVA.md) | Padrões que funcionaram, erros cometidos, o que cada um ensinou |
| [`SECURITY.md`](SECURITY.md) | Política de credenciais e operação por agente |

**Stack:** n8n · Stripe · Cloudflare R2 · Vercel · Node.js · Python · Bash ·
GitHub Actions

---

## Sobre este repositório

Documentação extraída de um projeto privado em produção. Identificadores de
conta, endpoints, dados cadastrais e chaves foram removidos ou substituídos por
placeholders — o que permanece são as decisões e as medições.

**Israel Passos** — AI Engineer · Smart LABS
[github.com/IsraelDev7](https://github.com/IsraelDev7) ·
[portifolio-smartlabs.vercel.app](https://portifolio-smartlabs.vercel.app)
