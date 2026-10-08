# Segurança

## Política de credenciais

Chave, token e segredo vivem **exclusivamente em variável de ambiente**.
Nunca em código, nunca em arquivo de configuração versionado, nunca em
documentação.

O que protege essa regra na prática:

- `.gitignore` exclui `.env*`, `*.pem`, `*.key` e
  `.claude/settings.local.json` desde o commit inicial — antes de existir
  histórico, porque o `.gitignore` não tem efeito retroativo.
- `scripts/verify.sh` varre todo arquivo rastreado em busca de padrões de
  credencial (Stripe, GitHub, AWS, Google, Slack, chave privada PEM) e
  **falha o build** se encontrar. Roda no CI a cada push e localmente com
  `bash scripts/verify.sh`.

## Um cuidado específico com ferramentas de agente

`.claude/settings.local.json` guarda a lista de comandos aprovados do
Claude Code, gravados **literalmente**. Se um comando aprovado tiver um
token embutido — por exemplo um `curl` com header de autorização — o token
fica em texto plano nesse arquivo.

Foi o que aconteceu neste projeto antes do repositório existir. O arquivo
está no `.gitignore` e o token envolvido foi rotacionado. A recomendação
que ficou: aprovar comandos que leiam a credencial de variável de ambiente
(`-H "Authorization: token $GITHUB_TOKEN"`), nunca com o valor literal.

## Como um agente opera as APIs sem ver as credenciais

O objetivo é autonomia de **execução**, não acesso ao **segredo**. As duas
coisas costumam ser confundidas, e separá-las é o que torna a automação
segura.

O padrão do projeto:

1. As credenciais ficam em `.env.local`, que está no `.gitignore` desde o
   commit inicial. O modelo versionado é `.env.example`, sem valores.
2. Os scripts em `scripts/` leem a variável **no momento da chamada** e a
   passam em header HTTP — nunca em linha de comando, que apareceria na
   lista de processos e no histórico do shell.
3. Nada em `scripts/lib/env.sh` imprime o conteúdo de uma credencial. O
   diagnóstico mostra presença, tamanho e um prefixo de 4 caracteres —
   suficiente para conferir se a chave certa foi carregada, insuficiente
   para reconstruí-la.
4. Operações que mudam estado pedem confirmação explícita (`SIM=1`).

O efeito prático: o valor da credencial não entra na conversa, não entra
no transcrito da sessão em disco, e não pode ser colado num commit por
descuido. É o mesmo princípio do Git Credential Manager, que já opera o
GitHub deste repositório sem que o token passe por ninguém.

**A regra que não muda:** nunca cole o valor de uma credencial em chat.
Se ela apareceu numa conversa, considere-a comprometida e rotacione — o
transcrito fica gravado em disco.

## Escopo de token

Tokens do GitHub usados aqui devem ser **fine-grained**, restritos a este
repositório, com a permissão mínima da tarefa — `Contents: Read and write`
é o suficiente para push. Token clássico com escopo administrativo dá a
quem o obtiver controle sobre a conta inteira, não sobre um projeto.

## Como reportar

Encontrou credencial exposta ou falha neste repositório: abra uma issue
**sem incluir o valor da credencial** e sinalize a urgência no título.
Se a exposição for ativa, rotacione primeiro e reporte depois — o
histórico do Git preserva o segredo mesmo após o arquivo ser removido,
então rotação é a única mitigação real.
