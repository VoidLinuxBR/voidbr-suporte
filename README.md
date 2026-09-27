<div align="center">

# 🔵 voidbr-suporte

**Suporte remoto rápido estilo tmate (Tailscale SSH + tmux compartilhado, ou upterm)**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge)](LICENSE)

</div>

---

Suporte remoto rápido para o **VoidBR Linux**, no espírito do antigo `tmate`:
o usuário digita **um comando** (inclusive no tty, sem ambiente gráfico) e o técnico
entra no **mesmo terminal**, vendo e digitando junto.

```
 [usuário, no tty]                 [Tailscale]                 [técnico]
 voidbr-suporte                    coordenação                 voidbr-suporte entrar
      │                                 │                            │
      ├─ sobe tailscaled temporário     │                            │
      ├─ entra na tailnet ─────────────►│  suporte-<user>-<host>-xxxx│
      │                                 │  aparece para o técnico ──►│
      ├─ abre tmux compartilhado        │                            │
      │◄═══════════ conexão direta, criptografada (WireGuard) ══════►│
      └─ exit → logout → máquina some da rede
```

- Sem conta, login, senha ou chave SSH do lado do usuário
- Ninguém precisa ditar token: a máquina aparece sozinha para o técnico
- Nada fica instalado ou rodando depois do atendimento
- Modo alternativo via **upterm**, para quando não houver tailnet

---

## Índice

- [Como funciona](#como-funciona)
- [Instalação](#instalação)
- [Preparação da tailnet (uma vez só)](#preparação-da-tailnet-uma-vez-só)
- [Uso](#uso)
- [Configuração](#configuração)
- [Segurança](#segurança)
- [Modo upterm](#modo-upterm)
- [Problemas comuns](#problemas-comuns)

---

## Como funciona

### Lado do usuário

Ao rodar `voidbr-suporte`, o script:

1. Cria uma pasta temporária e sobe um **`tailscaled` só para o atendimento**
   (modo userspace, estado só em memória). Não mexe na rede da máquina nem num
   Tailscale que já esteja instalado.
2. Entra na tailnet do técnico usando o segredo do `/etc/voidbr-suporte.conf`,
   como nó **efêmero** com a tag `tag:suporte` e o nome `suporte-<usuario>-<hostname>-<xxxx>`.
   O usuário vai no nome para o técnico entrar sem precisar informá-lo.
3. Liga o **Tailscale SSH**: o próprio tailscaled atende o SSH, sem sshd.
4. Abre o shell do usuário dentro de um **tmux** com uma barra colorida mostrando
   usuário, nome da máquina, IP e `exit = encerrar`.
5. Ao sair (`exit` ou Ctrl+C): faz `tailscale logout`, a máquina **sai da tailnet na hora**,
   o daemon é encerrado e a pasta temporária apagada.

### Lado do técnico

```bash
voidbr-suporte lista      # máquinas de suporte online
voidbr-suporte entrar     # entra no tmux do usuário
```

O `entrar` descobre o IP e o usuário pela própria tailnet (funciona sem MagicDNS)
e conecta direto no tmux compartilhado, sempre pelo IP.

### Barra do tmux

```
 SUPORTE  vcatafesta   0:bash         suporte-voidbr-liteon-nwhf  100.89.95.104  exit = encerrar
 vermelho azul         janelas        roxo                        verde          amarelo
```

Cores na paleta Tokyo Night. No tty, o tmux adapta para as cores do console.

---

## Instalação

### Pelo pkgmake

```bash
git clone https://github.com/voidlinuxbr/voidbr-suporte.git
cd voidbr-suporte
pkgmake -s -i        # instala as dependências, gera o .xbps e instala
```

### Manual

```bash
sudo xbps-install -S tailscale tmux curl qrencode
sudo install -Dm755 usr/bin/voidbr-suporte   /usr/bin/voidbr-suporte
sudo install -Dm644 etc/voidbr-suporte.conf  /etc/voidbr-suporte.conf
```

### Dependências

| Pacote | Uso |
|---|---|
| `tailscale` | modo tailscale (usuário e técnico) |
| `tmux` | terminal compartilhado |
| `curl` | aviso ao técnico via ntfy (opcional) |
| `qrencode` | QR code no modo upterm (opcional) |
| `upterm` | só para o modo upterm (não está no repositório do Void) |

> Na máquina do usuário **não** habilite o serviço `tailscaled`: o script sobe um próprio.

---

## Preparação da tailnet (uma vez só)

Tudo no admin console do Tailscale: <https://login.tailscale.com/admin>

### 1. Política de acesso

Em **Access controls**, no editor JSON. **Faça backup da política atual antes.**

A política padrão libera tudo para todos. Ela precisa sair, senão uma máquina de
suporte enxergaria sua rede inteira:

```jsonc
{
	"tagOwners": {
		"tag:suporte": ["autogroup:admin"],
	},

	"grants": [
		// REMOVER a regra "tudo liberado":
		// {"src": ["*"], "dst": ["*"], "ip": ["*"]},

		// suas máquinas continuam se enxergando
		{"src": ["autogroup:member"], "dst": ["autogroup:self"], "ip": ["*"]},

		// só admin alcança máquinas de suporte, só na porta 22
		{"src": ["autogroup:admin"], "dst": ["tag:suporte"], "ip": ["tcp:22"]},
	],

	"ssh": [
		{
			"action": "check",
			"src":    ["autogroup:member"],
			"dst":    ["autogroup:self"],
			"users":  ["autogroup:nonroot", "root"],
		},
		{
			"action": "accept",
			"src":    ["autogroup:admin"],
			"dst":    ["tag:suporte"],
			"users":  ["root", "autogroup:nonroot"],
		},
	],
}
```

| Regra | Efeito |
|---|---|
| `tagOwners` | cria a tag `tag:suporte` |
| grant 1 | suas máquinas continuam conversando entre si |
| grant 2 | admin → suporte, só porta 22 |
| `ssh` accept | Tailscale SSH aceita o admin sem pedir confirmação no navegador |
| nenhuma regra saindo de `tag:suporte` | máquina de suporte não enxerga nada |

> Para SSH funcionar são necessárias **as duas coisas**: o grant de rede (porta 22)
> **e** a regra em `ssh`.

### 2. OAuth client (o "convite")

1. **Settings → Trust credentials → Credential → OAuth**
2. Escopo: **somente Auth Keys**, com **Write**
3. Tag: **somente `tag:suporte`**
4. Descrição: `voidbr-suporte`
5. **Generate credential**
6. Copie o **Client secret** (`tskey-client-...`): ele **não aparece de novo**

Use uma credencial **exclusiva** para o suporte; não reaproveite a de CI ou outra.

### 3. Colocar o segredo no `.conf`

Na máquina do usuário (ou na ISO), em `/etc/voidbr-suporte.conf`:

```bash
ts_authkey="tskey-client-SEU_SECRET?ephemeral=true&preauthorized=true"
```

- As aspas são obrigatórias por causa do `?` e do `&`
- O Client ID não é usado

### 4. Lado do técnico

```bash
sudo xbps-install -S tailscale tmux
sudo ln -s /etc/sv/tailscaled /var/service/
sudo tailscale up        # conta admin da tailnet
```

O técnico **não** precisa do segredo no `.conf`.

---

## Uso

### Usuário

```bash
voidbr-suporte
```

Com `auto=1` (padrão do pacote), o quadro com o nome e o IP aparece por 2 segundos e o
tmux abre sozinho. Para encerrar: `exit`.

O usuário pode ser qualquer um: `root` na ISO live, `maria`, `anon`… O nome da máquina
sai com quem rodou o script, por exemplo `suporte-anon-notebook-a1b2`.

### Técnico

```bash
voidbr-suporte lista                              # quem está pedindo suporte
voidbr-suporte entrar                             # se houver só uma máquina online
voidbr-suporte entrar 100.95.244.100              # pelo IP
voidbr-suporte entrar liteon                      # por parte do nome
voidbr-suporte entrar 100.95.244.100 fulano       # forçando outro usuário
voidbr-suporte ouvir                              # aguarda chamados via ntfy
```

O `lista` mostra o usuário de cada máquina:

```
IP               USUÁRIO      NOME                                     ESTADO
100.95.244.100   vcatafesta   suporte-vcatafesta-voidbr-liteon-xe03    -
100.95.1.7       root         suporte-root-voidbr-live-ab12            active; direct
```

O `entrar` **descobre o usuário pelo nome da máquina**. O Tailscale SSH em modo userspace
só permite entrar como **o mesmo usuário que rodou o script** no remoto, e é esse que vai no nome:

| Quem roda no remoto | Nome da máquina | O `entrar` usa |
|---|---|---|
| `root` (ISO live) | `suporte-root-voidbr-live-x9z8` | `root` |
| `vcatafesta` | `suporte-vcatafesta-voidbr-liteon-xe03` | `vcatafesta` |
| `anon` | `suporte-anon-notebook-a1b2` | `anon` |

- O técnico **não precisa ter conta** no remoto: a tailnet autoriza o admin a entrar como esse usuário, sem senha.
- O usuário do técnico no local não importa.
- Dentro da sessão, o técnico tem as permissões do usuário; para root, `sudo` (o usuário digita a senha no terminal compartilhado).
- Usuários com caracteres fora de `a-z0-9` (ex.: `joao.silva`) vão simplificados no nome (`joaosilva`);
  nesse caso informe o usuário: `voidbr-suporte entrar <ip> joao.silva`.

Com mais de uma máquina online e sem argumento, o `entrar` mostra a lista e pede o nome ou o IP.

### Todos os comandos

| Comando | Lado | Descrição |
|---|---|---|
| *(nada)* | usuário | inicia no modo padrão (`modo` do `.conf`) |
| `tailscale` | usuário | inicia pelo Tailscale |
| `upterm` | usuário | inicia pelo upterm |
| `lista` / `ls` | técnico | máquinas `suporte-*` online |
| `entrar [nome\|IP] [usuario]` | técnico | entra no tmux compartilhado (usuário vem do nome) |
| `ouvir` | técnico | recebe chamados via ntfy |
| `ajuda` | ambos | ajuda |

---

## Configuração

Arquivo: `/etc/voidbr-suporte.conf` (preservado nas atualizações do pacote).

| Opção | Padrão | Descrição |
|---|---|---|
| `modo` | `tailscale` | `tailscale` ou `upterm` |
| `compartilhar` | `1` | `1` = mesmo terminal (tmux); `0` = técnico abre terminal próprio |
| `tmux_sessao` | `suporte` | nome da sessão tmux |
| `auto` | `0` (`1` no pacote) | `1` = abre o tmux direto, sem pedir Enter |
| `tmux_powerline` | `0` | `1` = separadores  na barra (precisa de Nerd Font; não aparece no tty) |
| `ts_authkey` | — | segredo do OAuth client com `?ephemeral=true&preauthorized=true` |
| `ts_tag` | `tag:suporte` | tag aplicada à máquina |
| `tecnicos_github` | — | (upterm) usuários GitHub autorizados |
| `tecnicos_chaves` | `/etc/voidbr-suporte/authorized_keys` | (upterm) chaves extras |
| `servidor` | `ssh://uptermd.upterm.dev:22` | (upterm) servidor |
| `perguntar` | `0` | (upterm) `1` = usuário aprova cada conexão |
| `ntfy_topico` | *(vazio)* | tópico ntfy.sh para avisar o técnico; vazio desliga |
| `ntfy_url` | `https://ntfy.sh` | servidor ntfy |

---

## Segurança

- **Não existe token de acesso.** Só entra quem é **admin da tailnet**.
- A conexão é **WireGuard ponta a ponta**; o servidor do Tailscale só apresenta um lado ao outro.
- As máquinas de suporte são **efêmeras**: saem da tailnet no `exit` e não deixam chave, serviço
  ou configuração para trás.
- O `entrar` não grava a host key em `known_hosts`, porque cada atendimento gera uma máquina nova
  e o IP pode se repetir; a identidade já é garantida pela tailnet.
- Máquinas de suporte **não alcançam nada** na tailnet (testado: conexão do remoto para o
  técnico não passa). Só o admin chega nelas, e só na porta 22.

### O segredo é público na prática

O `ts_authkey` vai dentro do pacote/ISO e **qualquer um consegue extraí-lo**. Com ele, alguém
consegue colocar máquinas na sua tailnet, mas:

- sempre com a tag `tag:suporte`, efêmeras;
- **sem enxergar nada**, graças à política acima.

Na pior hipótese aparecem máquinas estranhas no `lista`.

### Nunca publique o segredo no GitHub

O Tailscale participa da varredura de segredos do GitHub: um `tskey-client-...` real num
repositório público tende a ser **revogado automaticamente**. No repositório, mantenha o
valor de exemplo; o segredo real deve ser colocado no build da ISO ou após a instalação.

### Se o segredo vazar ou for abusado

1. **Settings → Trust credentials** → apague a credencial `voidbr-suporte`
2. **Machines** → filtre por `tag:suporte` e remova o que não reconhecer
3. Gere uma credencial nova e atualize o `.conf`

---

## Modo upterm

Alternativa sem tailnet, usando o servidor público do [upterm](https://github.com/owenthereal/upterm):

```bash
voidbr-suporte upterm
```

- Mostra o comando `ssh ...@uptermd.upterm.dev` e um QR code (com `qrencode`)
- Só entram as chaves dos técnicos (`tecnicos_github` / `tecnicos_chaves`)
- O usuário precisa passar o comando ao técnico (QR code, ntfy ou ditando)
- Requer o binário `upterm` instalado (não está no repositório do Void)

---

## Problemas comuns

**`ssh: Could not resolve hostname suporte-...`**
O MagicDNS não está ativo no seu Void. Use `voidbr-suporte entrar`, que resolve o IP
pela própria tailnet e conecta sempre pelo IP.

**`ERRO: ts_authkey não definido` no técnico**
Sem argumento, `voidbr-suporte` inicia o lado do **usuário**. No técnico use
`voidbr-suporte lista` e `voidbr-suporte entrar`.

**`can't switch user` / `não consegui entrar como '...' (código 255)`**
O login foi recusado porque o usuário não é o que rodou o script no remoto.
Causa comum: o remoto está com **versão antiga** do `voidbr-suporte` (nome sem o usuário,
`suporte-<host>-xxxx`), e o `entrar` leu o começo do hostname como usuário.
Atualize o remoto ou informe o usuário: `voidbr-suporte entrar <ip> <usuario>`.

**`entrou, mas a sessão 'suporte' do tmux não abriu` / `no sessions`**
O tmux do usuário ainda não abriu (com `auto=0`, ele precisa apertar Enter).

**Acentos aparecendo como `_`**
Use `voidbr-suporte entrar` (ele já força UTF-8 com `tmux -u`), em vez de `ssh` direto.

**Barra do tmux cortada**
Ela ocupa ~110 colunas. Em tty/VM com 80 colunas, o tmux corta o bloco da direita.

**`tailscaled não subiu`**
Veja o log em `/tmp/voidbr-suporte.*/tailscaled.log`.

**`não consegui conectar; confira a internet e a ts_authkey`**
Confira se o segredo está completo (maiúsculas/minúsculas), entre aspas, com
`?ephemeral=true&preauthorized=true`, e se a credencial não foi revogada.

---

## Autor

Vilmar Catafesta — [VoidBR Linux](https://voidbr.org)
