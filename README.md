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
 voidbr-suporte                    coordenação                 voidbr-suporte -c
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
- Mensagens traduzíveis com gettext (pt_BR e en)
- Modo alternativo via **upterm**, para quando não houver tailnet

---

## Índice

- [Como funciona](#como-funciona)
- [Instalação](#instalação)
- [Preparação da tailnet (uma vez só)](#preparação-da-tailnet-uma-vez-só)
- [Uso](#uso)
- [Configuração](#configuração)
- [Idiomas](#idiomas)
- [Segurança](#segurança)
- [Modo upterm](#modo-upterm)
- [Problemas comuns](#problemas-comuns)

---

## Como funciona

### Lado do usuário

Ao rodar `voidbr-suporte`, o script:

1. Cria uma pasta temporária e sobe um **`tailscaled` só para o atendimento**
   (modo userspace, estado só em memória, socket e porta próprios).
2. Entra na tailnet do técnico usando o segredo do `/etc/voidbr-suporte.conf`,
   como nó **efêmero** com a tag `tag:suporte` e o nome `suporte-<usuario>-<hostname>-<xxxx>`.
   O usuário vai no nome para o técnico entrar sem precisar informá-lo.
3. Liga o **Tailscale SSH**: o próprio tailscaled atende o SSH, sem sshd.
4. Abre o shell do usuário dentro de um **tmux** com uma barra colorida mostrando
   usuário, nome da máquina, IP e `exit = encerrar`.
5. Ao sair (`exit` ou Ctrl+C): faz `tailscale logout`, a máquina **sai da tailnet na hora**,
   o daemon é encerrado e a pasta temporária apagada.

### Por que o serviço `tailscaled` não precisa estar rodando no remoto

O serviço do runit só serve para ligar o `tailscaled` no boot. O `voidbr-suporte` liga
o seu próprio `tailscaled`, só durante o atendimento:

| | Serviço do sistema | Daemon do `voidbr-suporte` |
|---|---|---|
| Socket | `/run/tailscale/tailscaled.sock` | `/tmp/voidbr-suporte.XXXX/tailscaled.sock` |
| Estado | `/var/lib/tailscale/` (disco) | só em memória |
| Porta UDP | 41641 | qualquer livre (`--port=0`) |
| Rede | interface `tailscale0` e rotas | userspace, sem interface nem rotas |
| Duração | sempre | só durante o suporte |

Por isso:

- **Só fica conectado enquanto o usuário quer ajuda**; o técnico não acessa a máquina fora disso.
- **Não precisa de root**: um usuário comum consegue pedir suporte.
- **Não deixa rastro**: nada vai para o disco.
- **Convive com o Tailscale do usuário**: se ele já usa Tailscale na tailnet dele, os dois
  rodam lado a lado sem se misturar.
- **Funciona na ISO live** sem habilitar serviço nenhum.

### Lado do técnico

```bash
voidbr-suporte -l      # máquinas aguardando suporte
voidbr-suporte -c      # conecta no tmux do usuário
```

O `-c` descobre o IP e o usuário pela própria tailnet (funciona sem MagicDNS)
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
sudo xbps-install -S tailscale tmux curl qrencode gettext
sudo install -Dm755 usr/bin/voidbr-suporte   /usr/bin/voidbr-suporte
sudo install -Dm644 etc/voidbr-suporte.conf  /etc/voidbr-suporte.conf
```

### Dependências

| Pacote | Uso |
|---|---|
| `tailscale` | modo tailscale (usuário e técnico) |
| `tmux` | terminal compartilhado |
| `gettext` | mensagens traduzidas (sem ele, tudo sai em pt_BR) |
| `curl` | aviso ao técnico via ntfy (opcional) |
| `qrencode` | QR code no modo upterm (opcional) |
| `upterm` | só para o modo upterm (não está no repositório do Void) |

| Máquina | Pacote `tailscale` | Serviço `tailscaled` | Login na tailnet |
|---|---|---|---|
| **Técnico** | sim | **sim, sempre rodando** | sim, `tailscale up` com a conta admin |
| **Usuário** | sim | **não precisa** | não: o script entra sozinho com o segredo |

---

## Preparação da tailnet (uma vez só)

Tudo no admin console do Tailscale: <https://login.tailscale.com/admin>

> **Recomendado:** use uma **tailnet dedicada** ao suporte (ex.: a da organização),
> separada da sua tailnet pessoal. Convide sua conta pessoal como **Admin** nela.

> **Atenção:** se a sua conta participa de mais de uma tailnet, **confira no topo do
> console qual está selecionada** antes de salvar a política ou criar credenciais.
> O histórico de alterações fica em **Logs → Configuration**.

### 1. Política de acesso

Em **Access controls**, no editor JSON. **Faça backup da política atual antes.**

A política padrão (`"src": ["*"], "dst": ["*"]`) libera tudo para todos, inclusive para
máquinas com tag. Com ela, uma máquina de suporte enxergaria a rede inteira.
A política abaixo mantém o acesso livre para **pessoas** e deixa o suporte isolado:

```jsonc
{
	"tagOwners": {
		// máquinas que entram pelo voidbr-suporte
		"tag:suporte": ["autogroup:admin"],
	},

	"grants": [
		// tudo liberado, como na política padrão, mas só a partir de pessoas:
		//   autogroup:member = usuários da tailnet (inclusive convidados)
		//   autogroup:shared = quem aceitou uma máquina compartilhada
		// máquinas de suporte são "tagged", então ficam de fora
		{
			"src": ["autogroup:member", "autogroup:shared"],
			"dst": ["*"],
			"ip":  ["*"],
		},

		// outras máquinas com tag que precisem INICIAR conexões:
		// {"src": ["tag:server"], "dst": ["*"], "ip": ["*"]},

		// NUNCA coloque "tag:suporte" em src, nem use src "*".
	],

	"ssh": [
		// SSH nas próprias máquinas (padrão do Tailscale)
		{
			"action": "check",
			"src":    ["autogroup:member"],
			"dst":    ["autogroup:self"],
			"users":  ["autogroup:nonroot", "root"],
		},

		// admin entra nas máquinas de suporte como root ou qualquer usuário
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
| `tagOwners` | cria a tag `tag:suporte` (sem ela, o suporte não conecta) |
| grant | membros e convidados acessam tudo: máquinas, sub-redes, exit nodes |
| `ssh` accept | Tailscale SSH aceita o admin sem pedir confirmação no navegador |
| nenhuma regra saindo de `tag:suporte` | máquina de suporte não enxerga nada |

> Para SSH funcionar são necessárias **as duas coisas**: acesso de rede à porta 22
> (coberto pelo grant) **e** a regra em `ssh`.

### 2. OAuth client (o "convite")

1. **Settings → Trust credentials → Credential → OAuth**
2. Escopo: **somente Auth Keys**, com **Write**
3. Tag: **somente `tag:suporte`** (só aparece depois de salvar a política)
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
- O técnico **não** precisa do segredo

### 4. Lado do técnico

```bash
sudo xbps-install -S tailscale tmux
sudo ln -s /etc/sv/tailscaled /var/service/
sudo tailscale up        # conta admin da tailnet do suporte
```

Se a sua conta estiver em mais de uma tailnet, só uma fica ativa por vez no serviço:

```bash
tailscale switch --list          # o * marca a ativa
sudo tailscale switch <ID>       # troca de tailnet
```

---

## Uso

```
uso: voidbr-suporte [opções]

Usuário (pedir suporte):
  (sem opções)                    inicia o suporte (modo do .conf: tailscale)
  -t, --tailscale                 força o modo Tailscale
  -u, --upterm                    força o modo upterm

Técnico (dar suporte):
  -l, --list                      máquinas aguardando suporte
  -c, --connect [alvo] [usuario]  conecta na máquina (nome, parte do nome ou IP)
  -n, --listen                    aguarda chamados via ntfy

Gerais:
  -h, --help                      mostra esta ajuda
  -V, --version                   mostra a versão
```

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
voidbr-suporte -l                          # quem está aguardando suporte
voidbr-suporte -c                          # se houver só uma máquina online
voidbr-suporte -c 100.95.244.100           # pelo IP
voidbr-suporte -c liteon                   # por parte do nome
voidbr-suporte -c 100.95.244.100 fulano    # forçando outro usuário
voidbr-suporte -n                          # aguarda chamados via ntfy
```

O `-l` mostra o usuário de cada máquina:

```
IP               USUÁRIO      NOME                                     ESTADO
100.95.244.100   vcatafesta   suporte-vcatafesta-voidbr-liteon-xe03    -
100.95.1.7       root         suporte-root-voidbr-live-ab12            active; direct
```

O `-c` **descobre o usuário pelo nome da máquina**. O Tailscale SSH em modo userspace
só permite entrar como **o mesmo usuário que rodou o script** no remoto, e é esse que vai no nome:

| Quem roda no remoto | Nome da máquina | O `-c` usa |
|---|---|---|
| `root` (ISO live) | `suporte-root-voidbr-live-x9z8` | `root` |
| `vcatafesta` | `suporte-vcatafesta-voidbr-liteon-xe03` | `vcatafesta` |
| `anon` | `suporte-anon-notebook-a1b2` | `anon` |

- O técnico **não precisa ter conta** no remoto: a tailnet autoriza o admin a entrar como esse usuário, sem senha.
- O usuário do técnico no local não importa.
- Dentro da sessão, o técnico tem as permissões do usuário; para root, `sudo` (o usuário digita a senha no terminal compartilhado).
- Usuários com caracteres fora de `a-z0-9` (ex.: `joao.silva`) vão simplificados no nome (`joaosilva`);
  nesse caso informe o usuário: `voidbr-suporte -c <ip> joao.silva`.

Com mais de uma máquina online e sem argumento, o `-c` mostra a lista e pede o nome ou o IP.

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

## Idiomas

As mensagens usam **gettext** (domínio `voidbr-suporte`). Os textos originais estão em
**pt_BR**: sem catálogo instalado, ou sem o comando `gettext`, tudo sai em português.

- Catálogos: `/usr/share/locale/<idioma>/LC_MESSAGES/voidbr-suporte.mo`
- Traduções no repositório: `po/<idioma>.po`
- A função de tradução no script é `_` (palavra-chave para extração: `-k_`)

Para ver em inglês:

```bash
LANGUAGE=en voidbr-suporte -h
```

> O cabeçalho do script define `LANGUAGE=pt_BR` quando a variável está vazia. Por isso,
> num sistema com `LANG=en_US.UTF-8`, o inglês só aparece com `LANGUAGE=en` definido.

---

## Segurança

- **Não existe token de acesso.** Só entra quem é **admin da tailnet**.
- A conexão é **WireGuard ponta a ponta**; o servidor do Tailscale só apresenta um lado ao outro.
- As máquinas de suporte são **efêmeras**: saem da tailnet no `exit` e não deixam chave, serviço
  ou configuração para trás.
- O `-c` não grava a host key em `known_hosts`, porque cada atendimento gera uma máquina nova
  e o IP pode se repetir; a identidade já é garantida pela tailnet.
- Máquinas de suporte **não alcançam nada** na tailnet (testado: conexão do remoto para o
  técnico não passa).

### O segredo é público na prática

O `ts_authkey` vai dentro do pacote/ISO e **qualquer um consegue extraí-lo**. Com ele, alguém
consegue colocar máquinas na sua tailnet, mas:

- sempre com a tag `tag:suporte`, efêmeras;
- **sem enxergar nada**, graças à política acima.

Na pior hipótese aparecem máquinas estranhas no `-l`.

> Nunca use o segredo numa tailnet com a política padrão (tudo liberado): lá, a máquina
> de suporte teria acesso a tudo.

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
voidbr-suporte -u
```

- Mostra o comando `ssh ...@uptermd.upterm.dev` e um QR code (com `qrencode`)
- Só entram as chaves dos técnicos (`tecnicos_github` / `tecnicos_chaves`)
- O usuário precisa passar o comando ao técnico (QR code, ntfy ou ditando)
- Requer o binário `upterm` instalado (não está no repositório do Void)

---

## Problemas comuns

**`backend error: key tagged with non-existent tag: tag:suporte`**
A política da tailnet do segredo não tem mais a `tag:suporte` (bloco `tagOwners`).
Causa comum: a política foi trocada na tailnet errada. Confira a tailnet selecionada no
console e recoloque a política acima.

**`-l` não mostra nenhuma máquina, mas o remoto conectou**
O serviço `tailscaled` do técnico está em outra tailnet. Confira com
`tailscale switch --list` e troque com `sudo tailscale switch <ID>`.

**`ERRO: ts_authkey não definido` no técnico**
Sem opções, `voidbr-suporte` inicia o lado do **usuário**. No técnico use `-l` e `-c`.

**`can't switch user` / `não consegui entrar como '...' (código 255)`**
O login foi recusado porque o usuário não é o que rodou o script no remoto.
Causa comum: o remoto está com **versão antiga** do `voidbr-suporte` (nome sem o usuário,
`suporte-<host>-xxxx`), e o `-c` leu o começo do hostname como usuário.
Atualize o remoto ou informe o usuário: `voidbr-suporte -c <ip> <usuario>`.

**`entrou, mas a sessão 'suporte' do tmux não abriu` / `no sessions`**
O tmux do usuário ainda não abriu (com `auto=0`, ele precisa apertar Enter).

**`ssh: Could not resolve hostname suporte-...`**
O MagicDNS não está ativo. Use `voidbr-suporte -c`, que resolve o IP pela própria tailnet.

**Acentos aparecendo como `_`**
Use `voidbr-suporte -c` (ele força UTF-8 com `tmux -u`), em vez de `ssh` direto.

**Convidados da tailnet perderam acesso**
A política precisa ter `autogroup:shared` na origem, além de `autogroup:member`
(quem recebeu máquina compartilhada não é membro).

**Barra do tmux cortada**
Ela ocupa ~110 colunas. Em tty/VM com 80 colunas, o tmux corta o bloco da direita.

**`tailscaled não subiu`**
Veja o log em `/tmp/voidbr-suporte.*/tailscaled.log`.

**`não consegui conectar; confira a internet e a ts_authkey`**
Confira se o segredo está completo (maiúsculas/minúsculas), entre aspas, com
`?ephemeral=true&preauthorized=true`, e se a credencial não foi revogada.

---

## Autor

Vilmar Catafesta <vcatafesta@gmail.com> — [VoidBR Linux](https://voidbr.org) / [ChiliLinux](https://chililinux.com)
