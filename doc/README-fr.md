<div align="centre">

# 🔵 voidbr-support

**Support à distance rapide de style tmate (Tailscale SSH + tmux partagé ou upterm)**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge)](LICENCE)

</div>

---

Support à distance rapide pour **VoidBR Linux**, dans l'esprit de l'ancien `tmate` :
l'utilisateur tape **une commande** (y compris en tty, sans environnement graphique) et le technicien
entrez dans le **même terminal**, voyez et tapez ensemble.

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

- Pas de compte côté utilisateur, de login, de mot de passe ou de clé SSH
- Personne n'a besoin de dicter des jetons : la machine apparaît seule au technicien
- Rien ne reste installé ou en cours d'exécution après le service
- Messages traduisibles avec gettext (pt_BR et en)
- Mode alternatif via **upterm**, lorsqu'il n'y a pas de tailnet

---

## Indice

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

## Comment ça marche

### Côté utilisateur

À propos de `voidbr-suporte`, le script :

1. Créez un dossier temporaire et téléchargez un **`tailscaled` juste pour le service**
(mode espace utilisateur, état uniquement en mémoire, propre socket et port).
2. Entrez le tailnet du technicien à l'aide du secret `/etc/voidbr-suporte.conf`,
en tant que nœud **éphémère** avec la balise `tag:suporte` et le nom `suporte-<usuario>-<hostname>-<xxxx>`.
L'utilisateur saisit le nom du technicien sans avoir à l'en informer.
3. Activez **Tailscale SSH** : tailscaled lui-même sert SSH, sans sshd.
4. Ouvre le shell de l'utilisateur dans un **tmux** avec une barre colorée affichant
utilisateur, nom de la machine, IP et `exit = encerrar`.
5. En sortant (`exit` ou Ctrl+C) : faites `tailscale logout`, la machine **quitte immédiatement le tailnet**,
le démon est terminé et le dossier temporaire est supprimé.

### Pourquoi le service `tailscaled` n'a pas besoin d'être exécuté sur le serveur distant

Le service runit est uniquement utilisé pour activer `tailscaled` au démarrage. `voidbr-suporte` s'allume
votre propre `tailscaled`, uniquement pendant le service :

| | Service système | Le démon `voidbr-suporte` |
|---|---|---|
| Prise | `/run/tailscale/tailscaled.sock` | `/tmp/voidbr-suporte.XXXX/tailscaled.sock` |
| Statut | `/var/lib/tailscale/` (disque) | seulement en mémoire |
| Port UDP | 41641 | tout gratuit (`--port=0`) |
| Réseau | `tailscale0` interface et routes | espace utilisateur, sans interface ni routes |
| Durée | toujours | uniquement pendant le support |

C'est pourquoi :

- **Reste connecté uniquement lorsque l'utilisateur souhaite de l'aide** ; le technicien n'accède pas à la machine en dehors de cela.
- **Pas besoin de root** : un utilisateur régulier peut demander de l'aide.
- **Ne laisse aucune trace** : rien ne va sur le disque.
- **Coexiste avec le Tailscale de l'utilisateur** : s'il utilise déjà Tailscale sur son tailnet, les deux
ils courent côte à côte sans se mélanger.
- **Fonctionne sur ISO live** sans activer aucun service.

### Côté entraîneur

```bash
voidbr-suporte -l      # máquinas aguardando suporte
voidbr-suporte -c      # conecta no tmux do usuário
```

`-c` découvre l'adresse IP et l'utilisateur via le tailnet lui-même (fonctionne sans MagicDNS)
et se connecte directement au tmux partagé, toujours via IP.

### Ne sors pas

```
 SUPORTE  vcatafesta   0:bash         suporte-voidbr-liteon-nwhf  100.89.95.104  exit = encerrar
 vermelho azul         janelas        roxo                        verde          amarelo
```

Couleurs de la palette Tokyo Night. En tty, tmux s'adapte aux couleurs de la console.

---

## Installation

### Par pkgmake

```bash
git clone https://github.com/voidlinuxbr/voidbr-suporte.git
cd voidbr-suporte
pkgmake -s -i        # instala as dependências, gera o .xbps e instala
```

### Manuel

```bash
sudo xbps-install -S tailscale tmux curl qrencode gettext
sudo install -Dm755 usr/bin/voidbr-suporte   /usr/bin/voidbr-suporte
sudo install -Dm644 etc/voidbr-suporte.conf  /etc/voidbr-suporte.conf
```

### Dépendances

| Forfait | Utilisation |
|---|---|
| `tailscale` | mode tailscale (utilisateur et technicien) |
| `tmux` | terminal partagé |
| `gettext` | messages traduits (sans cela, tout sort dans pt_BR) |
| `curl` | avis au technicien via ntfy (facultatif) |
| `qrencode` | Code QR sans mode de mise à jour (facultatif) |
| `upterm` | uniquement pour le mode upterm (pas dans le référentiel Void) |

| Machines | Forfait `tailscale` | `tailscaled` Service | Connexion au réseau Tailnet |
|---|---|---|---|
| **Technique** | oui | **oui, toujours en cours d'exécution** | oui, `tailscale up` avec compte administrateur |
| **Utilisateur** | oui | **pas besoin** | non : le script entre seul avec le secret |

---

## Préparation du filet de queue (une seule fois)

Tout dans la console d'administration Tailscale : <https://login.tailscale.com/admin>

> **Recommandé :** utiliser un **tailnet dédié** au support (par exemple celui de l'organisation),
> séparé de votre filet personnel. Invitez-y votre compte personnel en tant qu'**Administrateur**.

> **Attention :** si votre compte participe à plus d'un tailnet, **cochez en haut de la page
> consolez laquelle est sélectionnée** avant d'enregistrer la stratégie ou de créer les informations d'identification.
> L'historique des modifications se trouve dans **Journaux → Configuration**.

### 1. Politique d'accès

Dans **Contrôles d'accès**, dans l'éditeur JSON. **Veuillez d'abord sauvegarder votre politique actuelle.**

La stratégie par défaut (`"src": ["*"], "dst": ["*"]`) diffuse tout à tout le monde, y compris
machines étiquetées. Avec lui, une machine de support verrait l’ensemble du réseau.
La politique ci-dessous maintient l'accès gratuit pour les **personnes** et laisse l'assistance isolée :

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

| Règle | Effet |
|---|---|
| `tagOwners` | crée le tag `tag:suporte` (sans lui, le support ne se connectera pas) |
| subvention | les membres et les invités accèdent à tout : machines, sous-réseaux, nœuds de sortie |
| `ssh` accepter | Tailscale SSH accepte l'administrateur sans demander la confirmation du navigateur |
| aucune règle ne quitte `tag:suporte` | la machine de support ne voit rien |

> Pour que SSH fonctionne, vous avez besoin des **deux choses** : un accès réseau au port 22
> (couvert par une subvention) **et** la règle dans `ssh`.

### 2. Client OAuth (l'« invitation »)

1. **Paramètres → Informations d'identification de confiance → Informations d'identification → OAuth**
2. Portée : **Clés d'authentification uniquement**, avec **Écriture**
3. Balise : **uniquement `tag:suporte`** (apparaît uniquement après l'enregistrement de la stratégie)
4. Description : `voidbr-suporte`
5. **Générer un identifiant**
6. Copiez le **Secret client** (`tskey-client-...`) : il **n'apparaît plus**

Utilisez un identifiant **unique** pour l'assistance ; ne réutilisez pas le CI ou autre.

### 3. Mettez le secret dans `.conf`

Sur la machine utilisateur (ou dans l'ISO), dans `/etc/voidbr-suporte.conf` :

```bash
ts_authkey="tskey-client-SEU_SECRET?ephemeral=true&preauthorized=true"
```

- Les guillemets sont obligatoires à cause de `?` et `&`
- L'ID client n'est pas utilisé
- Le technicien **n'a** pas besoin du secret

### 4. Côté coach

```bash
sudo xbps-install -S tailscale tmux
sudo ln -s /etc/sv/tailscaled /var/service/
sudo tailscale up        # conta admin da tailnet do suporte
```

Si votre compte est sur plusieurs tailnets, un seul est actif sur le service à la fois :

```bash
tailscale switch --list          # o * marca a ativa
sudo tailscale switch <ID>       # troca de tailnet
```

---

## Utiliser

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

### Utilisateur

```bash
voidbr-suporte
```

Avec `auto=1` (par défaut du package), la trame avec le nom et l'IP apparaît pendant 2 secondes et le
tmux s'ouvre tout seul. Fermer : `exit`.

L'utilisateur peut être n'importe qui : `root` dans l'ISO live, `maria`, `anon`… Le nom de la machine
se termine avec celui qui a exécuté le script, par exemple `suporte-anon-notebook-a1b2`.

### Technique

```bash
voidbr-suporte -l                          # quem está aguardando suporte
voidbr-suporte -c                          # se houver só uma máquina online
voidbr-suporte -c 100.95.244.100           # pelo IP
voidbr-suporte -c liteon                   # por parte do nome
voidbr-suporte -c 100.95.244.100 fulano    # forçando outro usuário
voidbr-suporte -n                          # aguarda chamados via ntfy
```

`-l` affiche l'utilisateur de chaque machine :

```
IP               USUÁRIO      NOME                                     ESTADO
100.95.244.100   vcatafesta   suporte-vcatafesta-voidbr-liteon-xe03    -
100.95.1.7       root         suporte-root-voidbr-live-ab12            active; direct
```

`-c` **découvre l'utilisateur par nom de machine**. Tailscale SSH en mode espace utilisateur
Il vous permet uniquement d'entrer comme **le même utilisateur qui a exécuté le script** sur la télécommande, et c'est ce qu'il y a dans le nom :

| Qui fonctionne à distance | Nom de l'appareil | `-c` utilise |
|---|---|---|
| `root` (ISO en direct) | `suporte-root-voidbr-live-x9z8` | `root` |
| `vcatafesta` | `suporte-vcatafesta-voidbr-liteon-xe03` | `vcatafesta` |
| `anon` | `suporte-anon-notebook-a1b2` | `anon` |

- Le technicien **n'a pas besoin d'avoir un compte** sur la télécommande : tailnet autorise l'administrateur à se connecter sous cet utilisateur, sans mot de passe.
- L'utilisateur du technicien sur place n'a pas d'importance.
- Au sein de la session, le technicien dispose des autorisations de l'utilisateur ; pour root, `sudo` (l'utilisateur saisit le mot de passe sur le terminal partagé).
- Les utilisateurs avec des caractères en dehors de `a-z0-9` (ex. : `joao.silva`) verront leur nom simplifié (`joaosilva`) ;
Dans ce cas, saisissez l'utilisateur : `voidbr-suporte -c <ip> joao.silva`.

Avec plus d'une machine en ligne et sans argument, `-c` affiche la liste et demande le nom ou l'IP.

---

## Paramètres

Fichier : `/etc/voidbr-suporte.conf` (conservé dans les mises à jour du package).

| Options | Norme | Descriptif |
|---|---|---|
| `modo` | `tailscale` | `tailscale` ou `upterm` |
| `compartilhar` | `1` | `1` = même terminal (tmux) ; `0` = le technicien ouvre son propre terminal |
| `tmux_sessao` | `suporte` | nom de session tmux |
| `auto` | `0` (`1` sans pacote) | `1` = ouvrir le tmux directement, sans entrer |
| `tmux_powerline` | `0` | `1` = séparateurs dans la barre (nécessite Nerd Font ; n'apparaît pas dans le tty) |
| `ts_authkey` | — | séparé du client OAuth avec `?ephemeral=true&preauthorized=true` |
| `ts_tag` | `tag:suporte` | étiquette appliquée à la machine |
| `tecnicos_github` | — | (à terme) utilisateurs GitHub autorisés |
| `tecnicos_chaves` | `/etc/voidbr-suporte/authorized_keys` | (à terme) chaves extras |
| `servidor` | `ssh://uptermd.upterm.dev:22` | (à terme) serveur |
| `perguntar` | `0` | (upterm) `1` = l'utilisateur approuve chaque connexion |
| `ntfy_topico` | *(vide)* | Sujet ntfy.sh pour avertir le technicien ; vide éteindre |
| `ntfy_url` | `https://ntfy.sh` | serveur ntfy |

---

## Langues

Les messages utilisent **gettext** (domaine `voidbr-suporte`). Les textes originaux sont en
**pt_BR** : sans le catalogue installé, ou sans la commande `gettext`, tout sort en portugais.

- Catalogues : `/usr/share/locale/<idioma>/LC_MESSAGES/voidbr-suporte.mo`
- Traductions dans le référentiel : `po/<idioma>.po`
- La fonction de traduction dans le script est `_` (mot clé pour l'extraction : `-k_`)

A voir en anglais :

```bash
LANGUAGE=en voidbr-suporte -h
```

> L'en-tête du script définit `LANGUAGE=pt_BR` lorsque la variable est vide. C'est pourquoi,
> sur un système avec `LANG=en_US.UTF-8`, l'anglais n'apparaît qu'avec `LANGUAGE=en` défini.

---

## Sécurité

- **Il n'y a pas de jeton d'accès.** Seuls ceux qui sont **administrateurs tailnet** peuvent participer.
- La connexion est **WireGuard de bout en bout** ; le serveur Tailscale ne présente qu'un côté à l'autre.
- Les machines de support sont **éphémères** : elles quittent le tailnet à `exit` et ne laissent aucune clé, aucun service
ou une configuration rétrospective.
- `-c` n'enregistre pas la clé de l'hôte dans `known_hosts`, car chaque service génère une nouvelle machine
et l'IP peut être répétée ; l’identité est déjà garantie par tailnet.
- Les machines de support **ne réalisent rien** sur le tailnet (testé : connexion du distant au
le technicien ne passe pas).

### Le secret est public en pratique

`ts_authkey` va à l'intérieur du package/ISO et **n'importe qui peut l'extraire**. Avec lui, quelqu'un
Vous pouvez placer des machines sur votre filet arrière, mais :

- toujours avec le tag `tag:suporte`, éphémère ;
- **sans rien voir**, grâce à la politique ci-dessus.

Dans le pire des cas, des machines étranges apparaissent dans `-l`.

> Ne jamais utiliser le secret sur un tailnet avec la politique par défaut (tout publié) : là, la machine
> Le support aurait accès à tout.

### Ne publiez jamais le secret sur GitHub

Tailscale participe au scan des secrets de GitHub : un vrai `tskey-client-...` en un
le référentiel public a tendance à être **automatiquement révoqué**. Dans le référentiel, conservez le
exemple de valeur ; le vrai secret doit être placé dans la version ISO ou après l'installation.

### Si le secret est divulgué ou abusé

1. **Paramètres → Informations d'identification de confiance** → apaguez une information d'identification `voidbr-suporte`
2. **Machines** → filtrer par `tag:suporte` et supprimer ce que vous ne reconnaissez pas
3. Générez un nouvel identifiant et mettez à jour `.conf`

---

## Mode à terme

Alternative sans tailnet, utilisant le serveur public [upterm](https://github.com/owenthereal/upterm) :

```bash
voidbr-suporte -u
```

- Affiche la commande `ssh ...@uptermd.upterm.dev` et un code QR (avec `qrencode`)
- Seules les clés des techniciens sont incluses (`tecnicos_github` / `tecnicos_chaves`)
- L'utilisateur doit transmettre la commande au technicien (code QR, ntfy ou dictée)
- Nécessite l'installation du binaire `upterm` (pas dans le référentiel Void)

---

## Problèmes courants

**`backend error: key tagged with non-existent tag: tag:suporte`**
La stratégie secrète tailnet n’a plus le `tag:suporte` (bloc `tagOwners`).
Cause fréquente : la stratégie a été modifiée sur le mauvais tailnet. Découvrez le tailnet sélectionné sur
console et remplacez la stratégie ci-dessus.

**`-l` n'affiche aucune machine, mais la télécommande connectée**
Le service `tailscaled` du technicien se trouve sur un autre tailnet. Vérifiez auprès de
`tailscale switch --list` et troque avec `sudo tailscale switch <ID>`.

*


*
La connexion a été refusée car l'utilisateur n'est pas celui qui a exécuté le script sur la télécommande.




*


*


*


*



*


*


*



---

## 


