<div align="center">

# 🔵 voidbr-support

**Fast tmate-style remote support (Tailscale SSH + shared tmux, or upterm)**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge)](LICENSE)

</div>

---

Fast remote support for **VoidBR Linux**, in the spirit of the old `tmate`:
the user types **a command** (including in tty, without a graphical environment) and the technician
enter the **same terminal**, seeing and typing together.

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

- No user-side account, login, password or SSH key
- No one needs to dictate tokens: the machine appears alone to the technician
- Nothing stays installed or running after service
- Translatable messages with gettext (pt_BR and en)
- Alternative mode via **upterm**, for when there is no tailnet

---

## Index

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

## How it works

### User side

Ao rodar `voidbr-suporte`, o script:

1. Create a temporary folder and upload a **`tailscaled` just for the service**
(userspace mode, state only in memory, own socket and port).
2. Enter the technician's tailnet using the `/etc/voidbr-suporte.conf` secret,
as an **ephemeral** node with the tag `tag:suporte` and the name `suporte-<usuario>-<hostname>-<xxxx>`.
The user enters the name for the technician to enter without having to inform him.
3. Turn on **Tailscale SSH**: tailscaled itself serves SSH, without sshd.
4. Opens the user's shell inside a **tmux** with a colored bar showing
user, machine name, IP and `exit = encerrar`.
5. When exiting (`exit` or Ctrl+C): do `tailscale logout`, the machine **exits the tailnet immediately**,
the daemon is terminated and the temp folder is deleted.

### Why the `tailscaled` service does not need to be running on the remote

The runit service is only used to turn on `tailscaled` at boot. `voidbr-suporte` turns on
your own `tailscaled`, only during the service:

| | System Service | The `voidbr-suporte` daemon |
|---|---|---|
| Socket | `/run/tailscale/tailscaled.sock` | `/tmp/voidbr-suporte.XXXX/tailscaled.sock` |
| Status | `/var/lib/tailscale/` (disk) | only in memory |
| UDP Port | 41641 | any free (`--port=0`) |
| Network | `tailscale0` interface and routes | userspace, without interface or routes |
| Duration | always | only during support |

That's why:

- **Only stays connected while the user wants help**; the technician does not access the machine outside of this.
- **No need for root**: a regular user can ask for support.
- **Leaves no trace**: nothing goes to disk.
- **Coexists with the user's Tailscale**: if he already uses Tailscale on his tailnet, both
they run side by side without mixing.
- **Works on ISO live** without enabling any service.

### Coach side

```bash
voidbr-suporte -l      # máquinas aguardando suporte
voidbr-suporte -c      # conecta no tmux do usuário
```

`-c` discovers the IP and user via the tailnet itself (works without MagicDNS)
and connects directly to the shared tmux, always via IP.

### Don't go outside

```
 SUPORTE  vcatafesta   0:bash         suporte-voidbr-liteon-nwhf  100.89.95.104  exit = encerrar
 vermelho azul         janelas        roxo                        verde          amarelo
```

Colors in the Tokyo Night palette. In tty, tmux adapts to the console colors.

---

## Installation

### By pkgmake

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

### Dependencies

| Package | Usage |
|---|---|
| `tailscale` | tailscale mode (user and technician) |
| `tmux` | shared terminal |
| `gettext` | translated messages (without it, everything comes out in pt_BR) |
| `curl` | notice to technician via ntfy (optional) |
| `qrencode` | QR code no modo upterm (opcional) |
| `upterm` | only for upterm mode (not in the Void repository) |

| Machine | Package `tailscale` | `tailscaled` Service | Tailnet login |
|---|---|---|---|
| **Technical** | yes | **yes, always running** | yes, `tailscale up` with admin account |
| **User** | yes | **no need** | no: the script enters alone with the secret |

---

## Tailnet preparation (once only)

Everything in the Tailscale admin console: <https://login.tailscale.com/admin>

> **Recommended:** use a **tailnet dedicated** to support (e.g. the organization's),
> separate from your personal tailnet. Invite your personal account as **Admin** in it.

> **Attention:** if your account participates in more than one tailnet, **check at the top of the
> console which one is selected** before saving the policy or creating credentials.
> The change history is in **Logs → Configuration**.

### 1. Access policy

In **Access controls**, in the JSON editor. **Please back up your current policy first.**

The default policy (`"src": ["*"], "dst": ["*"]`) releases everything to everyone, including
tagged machines. With it, a support machine would see the entire network.
The policy below keeps access free for **people** and leaves support isolated:

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

| Rule | Effect |
|---|---|
| `tagOwners` | creates the `tag:suporte` tag (without it, the support will not connect) |
| grant | members and guests access everything: machines, subnets, exit nodes |
| `ssh` accept | Tailscale SSH accepts admin without asking for browser confirmation |
| no rules leaving `tag:suporte` | support machine sees nothing |

> For SSH to work, you need **both things**: network access to port 22
> (covered by grant) **and** the rule in `ssh`.

### 2. OAuth client (the “invitation”)

1. **Settings → Trust credentials → Credential → OAuth**
2. Scope: **Auth Keys only**, with **Write**
3. Tag: **only `tag:suporte`** (only appears after saving the policy)
4. Description: `voidbr-suporte`
5. **Generate credential**
6. Copy the **Client secret** (`tskey-client-...`): it **does not appear again**

Use a **unique** credential for support; do not reuse the CI or other one.

### 3. Put the secret in `.conf`

On the user machine (or in the ISO), in `/etc/voidbr-suporte.conf`:

```bash
ts_authkey="tskey-client-SEU_SECRET?ephemeral=true&preauthorized=true"
```

- Quotation marks are required because of `?` and `&`
- Client ID is not used
- The technician **doesn't** need the secret

### 4. Coach side

```bash
sudo xbps-install -S tailscale tmux
sudo ln -s /etc/sv/tailscaled /var/service/
sudo tailscale up        # conta admin da tailnet do suporte
```

If your account is on more than one tailnet, only one is active on the service at a time:

```bash
tailscale switch --list          # o * marca a ativa
sudo tailscale switch <ID>       # troca de tailnet
```

---

## Use

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

### User

```bash
voidbr-suporte
```

With `auto=1` (package default), the frame with the name and IP appears for 2 seconds and the
tmux opens by itself. To close: `exit`.

The user can be anyone: `root` in the live ISO, `maria`, `anon`… The machine name
exits with whoever ran the script, for example `suporte-anon-notebook-a1b2`.

### Technical

```bash
voidbr-suporte -l                          # quem está aguardando suporte
voidbr-suporte -c                          # se houver só uma máquina online
voidbr-suporte -c 100.95.244.100           # pelo IP
voidbr-suporte -c liteon                   # por parte do nome
voidbr-suporte -c 100.95.244.100 fulano    # forçando outro usuário
voidbr-suporte -n                          # aguarda chamados via ntfy
```

`-l` shows the user of each machine:

```
IP               USUÁRIO      NOME                                     ESTADO
100.95.244.100   vcatafesta   suporte-vcatafesta-voidbr-liteon-xe03    -
100.95.1.7       root         suporte-root-voidbr-live-ab12            active; direct
```

`-c` **discovers the user by machine name**. Tailscale SSH in userspace mode
It only allows you to enter as **the same user who ran the script** on the remote, and that's what's in the name:

| Who runs on remote | Machine name | `-c` uses |
|---|---|---|
| `root` (ISO live) | `suporte-root-voidbr-live-x9z8` | `root` |
| `vcatafesta` | `suporte-vcatafesta-voidbr-liteon-xe03` | `vcatafesta` |
| `anon` | `suporte-anon-notebook-a1b2` | `anon` |

- The technician **does not need to have an account** on the remote: tailnet authorizes the admin to log in as this user, without a password.
- The user of the on-site technician does not matter.
- Within the session, the technician has the user's permissions; for root, `sudo` (user enters password on shared terminal).
- Users with characters outside of `a-z0-9` (e.g.: `joao.silva`) will have their name simplified (`joaosilva`);
In this case, enter the user: `voidbr-suporte -c <ip> joao.silva`.

With more than one machine online and without an argument, `-c` shows the list and asks for the name or IP.

---

## Settings

File: `/etc/voidbr-suporte.conf` (preserved across package updates).

| Option | Standard | Description |
|---|---|---|
| `modo` | `tailscale` | `tailscale` or `upterm` |
| `compartilhar` | `1` | `1` = same terminal (tmux); `0` = technician opens own terminal |
| `tmux_sessao` | `suporte` | tmux session name |
| `auto` | `0` (`1` no pacote) | `1` = abre o tmux direto, sem pedir Enter |
| `tmux_powerline` | `0` | `1` = separators in the bar (needs Nerd Font; does not appear in tty) |
| `ts_authkey` | — | segredo do OAuth client com `?ephemeral=true&preauthorized=true` |
| `ts_tag` | `tag:suporte` | tag applied to the machine |
| `tecnicos_github` | — | (upterm) authorized GitHub users |
| `tecnicos_chaves` | `/etc/voidbr-suporte/authorized_keys` | (upterm) chaves extras |
| `servidor` | `ssh://uptermd.upterm.dev:22` | (upterm) servidor |
| `perguntar` | `0` | (upterm) `1` = user approves each connection |
| `ntfy_topico` | *(empty)* | ntfy.sh topic to notify the technician; empty turn off |
| `ntfy_url` | `https://ntfy.sh` | server ntfy |

---

## Languages

Messages use **gettext** (domain `voidbr-suporte`). The original texts are in
**pt_BR**: without the catalog installed, or without the `gettext` command, everything comes out in Portuguese.

- Catalogs: `/usr/share/locale/<idioma>/LC_MESSAGES/voidbr-suporte.mo`
- Translations in the repository: `po/<idioma>.po`
- The translation function in the script is `_` (keyword for extraction: `-k_`)

To see in English:

```bash
LANGUAGE=en voidbr-suporte -h
```

> The script header sets `LANGUAGE=pt_BR` when the variable is empty. That's why,
> on a system with `LANG=en_US.UTF-8`, English only appears with `LANGUAGE=en` defined.

---

## Security

- **There is no access token.** Only those who are **tailnet admin** can enter.
- The connection is **WireGuard end-to-end**; the Tailscale server only presents one side to the other.
- Support machines are **ephemeral**: they leave the tailnet at `exit` and leave no key, service
or backward configuration.
- `-c` does not record the host key in `known_hosts`, because each service generates a new machine
and the IP can be repeated; identity is already guaranteed by tailnet.
- Support machines **do not achieve anything** on the tailnet (tested: connection from remote to
technician does not pass).

### The secret is public in practice

`ts_authkey` goes inside the package/ISO and **anyone can extract it**. With him, someone
You can place machines on your tailnet, but:

- always with the tag `tag:suporte`, ephemera;
- **without seeing anything**, thanks to the policy above.

In the worst case scenario, strange machines appear in `-l`.

> Never use the secret on a tailnet with the default policy (everything released): there, the machine
> Support would have access to everything.

### Never publish the secret on GitHub

Tailscale participates in GitHub secrets scan: a real `tskey-client-...` in one
public repository tends to be **automatically revoked**. In the repository, keep the
example value; the real secret must be placed in the ISO build or after installation.

### If the secret is leaked or abused

1. **Settings → Trust credentials** → apague a credencial `voidbr-suporte`
2. **Machines** → filter by `tag:suporte` and remove what you don't recognize
3. Generate a new credential and update `.conf`

---

## Upterm mode

Alternative without tailnet, using the [upterm](https://github.com/owenthereal/upterm) public server:

```bash
voidbr-suporte -u
```

- Shows the `ssh ...@uptermd.upterm.dev` command and a QR code (with `qrencode`)
- Only technicians' keys are included (`tecnicos_github` / `tecnicos_chaves`)
- The user needs to pass the command to the technician (QR code, ntfy or dictating)
- Requires the `upterm` binary installed (not in the Void repository)

---

## Common Problems

**`backend error: key tagged with non-existent tag: tag:suporte`**
The secret tailnet policy no longer has the `tag:suporte` (`tagOwners` block).
Common cause: Policy was changed on the wrong tailnet. Check out the selected tailnet on
console and replace the above policy.

**`-l` does not show any machine, but the remote connected**
The technician's `tailscaled` service is on another tailnet. Check with
`tailscale switch --list` e troque com `sudo tailscale switch <ID>`.

**`ERRO: ts_authkey não definido` non-technical**
Without options, `voidbr-suporte` starts the **user** side. In the technician use `-l` and `-c`.

**`can't switch user` / `não consegui entrar como '...' (código 255)`**
Login was refused because the user is not the one who ran the script on the remote.
Common cause: the remote has an **old version** of `voidbr-suporte` (name without user,
`suporte-<host>-xxxx`), and `-c` read the beginning of the hostname as user.
Update the remote or enter the user: `voidbr-suporte -c <ip> <usuario>`.

**`entrou, mas a sessão 'suporte' do tmux não abriu` / `no sessions`**
The user's tmux has not yet opened (with `auto=0`, he needs to press Enter).

**`ssh: Could not resolve hostname suporte-...`**
MagicDNS is not active. Use `voidbr-suporte -c`, which resolves the IP to the tailnet itself.

**Accents appearing as `_`**
Use `voidbr-suporte -c` (it forces UTF-8 with `tmux -u`), instead of straight `ssh`.

**Tailnet guests lost access**
The policy must have `autogroup:shared` in the source in addition to `autogroup:member`
(whoever received a shared machine is not a member).

**tmux bar cut**
It takes up ~110 columns. In tty/VM with 80 columns, tmux cuts the right block.

**`tailscaled não subiu`**
See the log at `/tmp/voidbr-suporte.*/tailscaled.log`.

**`não consegui conectar; confira a internet e a ts_authkey`**
Check if the secret is complete (upper/lower case), in quotation marks, with
`?ephemeral=true&preauthorized=true`, and that the credential has not been revoked.

---

## Author

Vilmar Catafesta <vcatafesta@gmail.com> — [VoidBR Linux](https://voidbr.org) / [ChiliLinux](https://chililinux.com)
