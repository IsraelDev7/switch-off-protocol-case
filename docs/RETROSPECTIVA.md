# Retrospectiva — do repositório ao funil e à persona

*Sessão de 08 a 18/08/2026 · Israel (Smart LABS) + Claude*

Registro honesto do que foi construído, do que quebrou, e do que ficou
aprendido. Escrito para ser relido antes do próximo projeto do mesmo
tipo — não para celebrar.

---

## 1. O que foi entregue

| Frente | Estado final |
|---|---|
| **Repositório** | `repositório privado do projeto, 9,1 MB versionados (de 198 MB de pasta), CI verde |
| **Documentação** | README, SECURITY, LICENSE, 8 ADRs |
| **Páginas legais** | Terms, Privacy, Refunds, Contact — as quatro no ar |
| **Landing** | `exemplo.com`, 7 cabeçalhos de segurança, CSP restritiva |
| **Checkout** | Stripe Payment Link com consentimento registrado, order bump, redirect |
| **Entrega** | Webhook autenticado por HMAC → 8 arquivos por link assinado do R2 em **1,8 s** |
| **Sequência** | 7 e-mails, workflow desacoplado, ativo |
| **Produto** | EPUB gerado do HTML curado, pasta de entrega montada por script |
| **Persona** | Avatar, bio, 5 destaques, 9 posts com legenda, 6 frames, 3 vídeos, 3 narrações |

Ferramentas criadas: `verify.sh`, `build-pdf.sh`, `build-epub.py`,
`build-delivery.sh`, `n8n.sh`, `lib/env.sh`.

---

## 2. Padrões que funcionaram — repetir

### 2.1 Verificar a causa, nunca o efeito

O caso mais grave: os testes de rejeição do webhook retornavam HTTP 500 e
recusavam requisições forjadas. **Parecia defesa funcionando.** Ao ler a
execução, a causa era `Module 'crypto' is disallowed` — a validação de
assinatura nunca tinha rodado. O comportamento externo era
indistinguível de um sistema correto.

> Resultado certo pelo motivo errado é o modo de falha mais perigoso em
> segurança, porque não deixa sintoma.

Mesmo padrão em: `Authorization failed` que era credencial **ausente**
(o Stripe dizia isso literalmente no corpo do erro, não no rótulo), e no
DNS que "parecia" errado e era cache.

### 2.2 Medir a restrição real antes de produzir

- **Avatar:** as 4 opções pareciam ótimas a 1500 px. Simuladas em círculo
  a 32/56/110 px — o tamanho real de uso — três perderam o que as tornava
  boas.
- **Narração:** 5,96 s de voz em 6,00 s de vídeo. Só aparece com régua, e
  só apareceria depois do lip sync gerado, quando o custo já foi pago.
- **PDF regerado:** comparação byte a byte antes de aceitar que `out/`
  podia ficar fora do Git.

### 2.3 Sondar o ambiente antes de escrever código para ele

Escrevi 200 linhas de workflow n8n assumindo Node completo. O sandbox do
Code node bloqueia `require('crypto')`, não expõe Web Crypto e desliga
`$env` por padrão. Um workflow de diagnóstico descartável respondeu tudo
em dez minutos; descobrir um erro por vez teria custado a tarde.

**Corolário:** a sonda tem que perguntar a coisa certa. O primeiro
diagnóstico disse que `$env` estava vazio porque testei
`Object.keys($env)` — e `$env` é um Proxy sem `ownKeys`. O acesso direto
sempre funcionou.

### 2.4 Guardrail estrutural vence guardrail por formato

Depois do incidente das credenciais no `.env.example`, a verificação que
entrou no CI **não** procura padrões de chave. Ela diz: *"um modelo não
tem valores preenchidos"*. Isso pega chave hexadecimal genérica, que
nenhuma regex de credencial reconheceria.

### 2.5 Autonomia de execução não é acesso ao segredo

O padrão `.env.local` + `scripts/` permite operar APIs sem que o valor da
credencial apareça em conversa, log ou commit. A chave é lida no momento
da chamada e vai em header — nunca em linha de comando, que apareceria
em `ps` e no histórico do shell.

Confirmação disso: o `git push` funcionou **sem token e sem helper
configurado**, porque o Git Credential Manager já operava em nível de
sistema. A variável `GITHUB_TOKEN` era redundante — e era a parte
insegura.

### 2.6 Versiona-se a fonte, e o caminho de volta é testado

`out/` ficou fora do Git porque `build-pdf.sh` regenera. Isso só se
sustenta porque foi **verificado**: os quatro PDFs regeraram com tamanho
idêntico byte a byte, divergindo 12 e 14 bytes nos primeiros 250 — a
região de `CreationDate`. Ignorar artefato sem caminho de volta testado é
perder o artefato.

### 2.7 Documento e realidade precisam coincidir

Quatro ocorrências do mesmo erro, em disfarces diferentes:

| Documento afirmava | Realidade |
|---|---|
| checkout coleta consentimento | Payment Link não coletava |
| unsubscribe "funciona imediatamente" | marcador `{{UNSUB}}` nunca substituído |
| termos linkados no checkout | apontavam para domínio inexistente |
| três workers rodando | `docker-compose.yml` não declarava |

Nenhum aparece em teste, log ou erro. Só aparece quando alguém compara as
duas coisas de propósito.

### 2.8 A pasta de trabalho nunca é a pasta de entrega

Três vezes: `.gitignore` (198 MB → 9,1), EPUB (manuscrito com briefing →
`book.html` curado), entrega (11 PDFs → 3). Sempre o mesmo erro — tratar
"onde os arquivos estão" como "o que o produto é".

### 2.9 Em geração de imagem: fechar a lacuna, não regerar

A mesma chamada produziu uma variação inutilizável e uma perfeita.
Regerar não era solução — o problema era **ambiguidade**, e com prompt
ambíguo o resultado é sorteado dentro de uma faixa que inclui o
inaceitável.

### 2.10 Aceitar o take bom

A tentativa de corrigir dois detalhes secundários do S03 produziu um
resultado pior, e o original foi mantido. Modelo generativo não faz
correção cirúrgica: produz amostra nova inteira, arriscando o que já
funcionava.

### 2.11 Desacoplar criticidades diferentes

Entrega e sequência de marketing viraram workflows separados. Falha na
entrega é alguém que pagou e não recebeu; falha na sequência é uma pena.
Juntas, um erro de marketing poderia derrubar uma obrigação contratual.

---

## 3. Erros cometidos — e o que cada um ensinou

### 3.1 Token gravado no `.git/config`

`git push --set-upstream <url-com-token>` persistiu a credencial na
configuração local. Detectado na verificação seguinte, removido, upstream
refeito.

**Regra:** remote sempre limpo; autenticação vem de fora (GCM, `gh auth`,
SSH). Credencial em URL é transitória por acidente e permanente por
engano.

### 3.2 Verificação de vazamento mal construída

O regex `^[A-Z_]+=` não casa com `N8N_` nem `R2_` — têm dígitos. O
relatório saiu incompleto e um falso positivo (`rk_live_` que era texto
do meu próprio comentário) quase gerou alarme falso.

**Regra:** antes de reportar um achado de segurança, validar o próprio
método de detecção.

### 3.3 Hipótese errada sobre o Chrome

Atribuí a falha de renderização da capa a espaço no caminho. Testei — e o
Chrome escreve normalmente em caminho com espaço. A causa raiz nunca foi
reproduzida; o que se corrigiu foi a **invisibilidade** da falha.

**Regra:** dizer "consertei a visibilidade, não reproduzi a causa" em vez
de deixar implícito que foi resolvido.

### 3.4 Falha silenciosa engolida por `except`

O gerador de EPUB produziu arquivo de 22 KB sem capa e não reclamou.
Agora aborta com o motivo.

### 3.5 Fonte inventada

Escrevi que a persona citaria *"Why I Rest"*, de Nick Littlehales. **O livro
não existe** — o real é *Sleep*. Erro grave num documento que prega citar
fontes com precisão, e num nicho onde a autoridade vem exatamente disso.

**Regra:** toda citação verificada antes de publicar. Fonte errada é pior
que fonte nenhuma.

### 3.6 Promessa órfã no rodapé dos e-mails

`Unsubscribe here: {{UNSUB}}` — marcador nunca substituído, enquanto a
Privacy Policy prometia link imediato. Corrigidos os dois lados: opt-out
real por resposta com STOP, e a política descrevendo exatamente isso.

### 3.7 Ordem errada no workflow

O nó decidia o order bump **antes** de buscar `line_items`. Pego em
revisão, antes de entregar.

### 3.8 E-mail só em HTML

A versão em texto puro era gerada e não usada — pior entregabilidade.

### 3.9 BOM em UTF-8

`Set-Content -Encoding utf8` no PowerShell 5.1 grava BOM, que quebrou o
parse do `settings.local.json`.

### 3.10 Prompt sem vestuário

Gerou imagem inadequada — sem roupa e com a personagem duplicada. Única
causa: lacuna no prompt.

### 3.11 Recomendação desnecessária

Sugeri configurar `credential.helper manager` quando ele já estava ativo
em nível de sistema. **Verificar o estado antes de recomendar mudança.**

---

## 4. As regras do Israel — como foram aplicadas

| Regra | Aplicação |
|---|---|
| Plano antes de execução | Cumprido em todas as frentes; decisões com trade-off foram para `AskUserQuestion` |
| Evidência, nunca "pronto" | Todo entregável veio com teste rodado, screenshot, hash ou medição |
| Correções incrementais | Uma variável por iteração, principalmente em geração de imagem |
| PT-BR comigo, EN-US no produto | Mantido sem exceção |
| Segredos só em variável de ambiente | Sustentado; incidentes tratados com rotação e guardrail |
| Direção de arte, não operação de ferramenta | Raciocínio de enquadramento e referência real antes de cada geração |
| Preservar versão antes de sobrescrever | `OUT=` parametrizável no gerador de PDF; variações de capa mantidas |

**Onde falhei nas regras dele:** a citação inventada viola diretamente
"nunca invente conteúdo do produto". Foi o pior erro da sessão, e o único
que teria dano de reputação se publicado.

---

## 5. Decisões travadas (ADRs)

0001 preço $37 · 0002 Payment Link · 0003 sem depoimento fabricado ·
0004 sem mecanismo neologizado · 0005 o que fica fora do Git ·
0006 venda sem restrição de país · 0007 entrega EPUB+PDF ·

---

## 6. O que continua aberto

- `exemplo.com` sem `www` — cache de DNS, não configuração
- Opt-out automático da sequência
- E-mail do dia 8 (depende da esteira existir)
- Rotacionar a chave do n8n
- Áudios em rascunho — decisão consciente de manter
- 4 vídeos de cena a gerar