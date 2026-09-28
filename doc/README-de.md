<div align="center">

# 🔵 voidbr-Unterstützung

**Schnelle Remote-Unterstützung im tmate-Stil (Tailscale SSH + gemeinsam genutztes tmux oder upterm)**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge)](LIZENZ)

</div>

---

Schneller Remote-Support für **VoidBR Linux**, ganz im Sinne des alten `tmate`:
Der Benutzer gibt **einen Befehl** ein (auch in TTY, ohne grafische Umgebung) und der Techniker
Betreten Sie das **gleiche Terminal** und sehen und tippen Sie gleichzeitig.

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

- Kein benutzerseitiges Konto, Login, Passwort oder SSH-Schlüssel
- Niemand muss Token diktieren: Die Maschine erscheint dem Techniker allein
- Nach der Wartung bleibt nichts installiert oder läuft
- Übersetzbare Nachrichten mit gettext (pt_BR und en)
- Alternativer Modus über **upterm**, wenn kein Tailnet vorhanden ist

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

## Wie es funktioniert

### Benutzerseite

Dann schreibe `voidbr-suporte`, oder Skript:

1. Erstellen Sie einen temporären Ordner und laden Sie ein **`tailscaled` nur für den Dienst** hoch.
(Userspace-Modus, Status nur im Speicher, eigener Socket und Port).
2. Betreten Sie das Tailnet des Technikers mit dem `/etc/voidbr-suporte.conf`-Geheimnis.
als **ephemerer** Knoten mit dem Tag `tag:suporte` und dem Namen `suporte-<usuario>-<hostname>-<xxxx>`.
Der Benutzer gibt den Namen für den Techniker ein, ohne ihn darüber informieren zu müssen.
3. Aktivieren Sie **Tailscale SSH**: Tailscaled selbst bedient SSH, ohne sshd.
4. Öffnet die Shell des Benutzers in einem **tmux** mit einem farbigen Balken
Benutzer, Maschinenname, IP und `exit = encerrar`.
5. Beim Verlassen (`exit` oder Strg+C): `tailscale logout` ausführen, die Maschine **verlässt das Hecknetz sofort**,
Der Daemon wird beendet und der temporäre Ordner gelöscht.

### Warum der `tailscaled`-Dienst nicht auf der Fernbedienung ausgeführt werden muss

Der Runit-Dienst wird nur verwendet, um `tailscaled` beim Booten zu aktivieren. `voidbr-suporte` schaltet sich ein
Ihr eigenes `tailscaled`, nur während des Gottesdienstes:

| | Systemdienst | Der `voidbr-suporte`-Daemon |
|---|---|---|
| Sockel | `/run/tailscale/tailscaled.sock` | `/tmp/voidbr-suporte.XXXX/tailscaled.sock` |
| Status | `/var/lib/tailscale/` (Datenträger) | nur in der Erinnerung |
| UDP-Port | 41641 | beliebig frei (`--port=0`) |
| Netzwerk | `tailscale0`-Schnittstelle und Routen | Userspace, ohne Schnittstelle oder Routen |
| Dauer | immer | nur während des Supports |

Deshalb:

- **Bleibt nur verbunden, solange der Benutzer Hilfe benötigt**; Außerhalb dieser Frist greift der Techniker nicht auf die Maschine zu.
- **Kein Root erforderlich**: Ein normaler Benutzer kann um Support bitten.
- **Hinterlässt keine Spuren**: Nichts wird auf der Festplatte gespeichert.
- **Koexistiert mit der Tailscale des Benutzers**: Wenn er Tailscale bereits auf seinem Tailnet verwendet, beides
sie laufen nebeneinander, ohne sich zu vermischen.
- **Funktioniert auf ISO Live**, ohne dass ein Dienst aktiviert werden muss.

### Trainerseite

```bash
voidbr-suporte -l      # máquinas aguardando suporte
voidbr-suporte -c      # conecta no tmux do usuário
```

`-c` erkennt die IP und den Benutzer über das Tailnet selbst (funktioniert ohne MagicDNS)
und verbindet sich direkt mit dem gemeinsam genutzten tmux, immer über IP.

### Gehen Sie nicht nach draußen

```
 SUPORTE  vcatafesta   0:bash         suporte-voidbr-liteon-nwhf  100.89.95.104  exit = encerrar
 vermelho azul         janelas        roxo                        verde          amarelo
```

Farben in der Tokyo Night-Palette. In tty passt sich tmux den Konsolenfarben an.

---

## Installation

### Von pkgmake

```bash
git clone https://github.com/voidlinuxbr/voidbr-suporte.git
cd voidbr-suporte
pkgmake -s -i        # instala as dependências, gera o .xbps e instala
```

### Handbuch

```bash
sudo xbps-install -S tailscale tmux curl qrencode gettext
sudo install -Dm755 usr/bin/voidbr-suporte   /usr/bin/voidbr-suporte
sudo install -Dm644 etc/voidbr-suporte.conf  /etc/voidbr-suporte.conf
```

### Abhängigkeiten

| Paket | Verwendung |
|---|---|
| `tailscale` | Tailscale-Modus (Benutzer und Techniker) |
| `tmux` | gemeinsam genutztes Terminal |
| `gettext` | übersetzte Nachrichten (ohne sie kommt alles in pt_BR heraus) |
| `curl` | Mitteilung an den Techniker über ntfy (optional) |
| `qrencode` | QR-Code ohne Modo Upterm (optional) |
| `upterm` | nur für den Upterm-Modus (nicht im Void-Repository) |

| Maschine | Paket `tailscale` | `tailscaled`-Dienst | Tailnet-Login |
|---|---|---|---|
| **Technisch** | ja | **Ja, läuft immer** | ja, `tailscale up` mit Admin-Konto |
| **Benutzer** | ja | **keine Notwendigkeit** | nein: Das Skript tritt alleine mit dem Geheimnis | ein

---

## Hecknetzvorbereitung (einmalig)

Alles in der Tailscale-Administratorkonsole: <https://login.tailscale.com/admin>

> **Empfohlen:** Verwenden Sie ein **dediziertes** Tailnet zur Unterstützung (z. B. der Organisation).
> getrennt von Ihrem persönlichen Tailnet. Laden Sie darin Ihr persönliches Konto als **Administrator** ein.

> **Achtung:** Wenn Ihr Konto an mehr als einem Tailnet teilnimmt, **überprüfen Sie dies oben im
> Konsole, welche ausgewählt ist**, bevor Sie die Richtlinie speichern oder Anmeldeinformationen erstellen.
> Der Änderungsverlauf befindet sich unter **Protokolle → Konfiguration**.

### 1. Zugriffsrichtlinie

In **Zugriffskontrollen**, im JSON-Editor. **Bitte sichern Sie zuerst Ihre aktuelle Richtlinie.**

Die Standardrichtlinie (`"src": ["*"], "dst": ["*"]`) gibt alles für alle frei, einschließlich
getaggte Maschinen. Damit würde eine Support-Maschine das gesamte Netzwerk sehen.
Die folgende Richtlinie hält den Zugang für **Personen** frei und lässt den Support isoliert:

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

| Regel | Wirkung |
|---|---|
| `tagOwners` | erstellt das Tag `tag:suporte` (ohne dieses wird keine Verbindung zum Support hergestellt) |
| gewähren | Mitglieder und Gäste haben Zugriff auf alles: Maschinen, Subnetze, Ausgangsknoten |
| `ssh` akzeptieren | Tailscale SSH akzeptiert Administratoren, ohne nach einer Browserbestätigung zu fragen |
| Keine Regeln verlassen `tag:suporte` | Support-Maschine sieht nichts |

> Damit SSH funktioniert, benötigen Sie **beides**: Netzwerkzugriff auf Port 22
> (durch Zuschuss abgedeckt) **und** die Regel in `ssh`.

### 2. OAuth-Client (die „Einladung“)

1. **Einstellungen → Anmeldeinformationen vertrauen → Anmeldeinformationen → OAuth**
2. Geltungsbereich: **Nur Authentifizierungsschlüssel**, mit **Schreiben**
3. Tag: **nur `tag:suporte`** (erscheint erst nach dem Speichern der Richtlinie)
4. Beschreibung: `voidbr-suporte`
5. **Anmeldeinformationen generieren**
6. Kopieren Sie das **Client-Geheimnis** (`tskey-client-...`): Es wird **nicht mehr angezeigt**

Verwenden Sie für den Support einen **eindeutigen** Berechtigungsnachweis. Verwenden Sie das CI oder ein anderes nicht wieder.

### 3. Geben Sie das Geheimnis in `.conf` ein

Auf dem Benutzercomputer (oder im ISO) in `/etc/voidbr-suporte.conf`:

```bash
ts_authkey="tskey-client-SEU_SECRET?ephemeral=true&preauthorized=true"
```

- Wegen `?` und `&` sind Anführungszeichen erforderlich
- Die Client-ID wird nicht verwendet
- Der Techniker braucht das Geheimnis nicht

### 4. Trainerseite

```bash
sudo xbps-install -S tailscale tmux
sudo ln -s /etc/sv/tailscaled /var/service/
sudo tailscale up        # conta admin da tailnet do suporte
```

Wenn sich Ihr Konto in mehr als einem Tailnet befindet, ist jeweils nur eines im Dienst aktiv:

```bash
tailscale switch --list          # o * marca a ativa
sudo tailscale switch <ID>       # troca de tailnet
```

---

## Verwenden

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

### Benutzer

```bash
voidbr-suporte
```

Bei `auto=1` (Paketstandard) erscheint der Frame mit dem Namen und der IP für 2 Sekunden und der
tmux öffnet sich von selbst. Zum Schließen: `exit`.

Der Benutzer kann jeder sein: `root` im Live-ISO, `maria`, `anon`… Der Maschinenname
wird mit demjenigen beendet, der das Skript ausgeführt hat, zum Beispiel `suporte-anon-notebook-a1b2`.

### Technisch

```bash
voidbr-suporte -l                          # quem está aguardando suporte
voidbr-suporte -c                          # se houver só uma máquina online
voidbr-suporte -c 100.95.244.100           # pelo IP
voidbr-suporte -c liteon                   # por parte do nome
voidbr-suporte -c 100.95.244.100 fulano    # forçando outro usuário
voidbr-suporte -n                          # aguarda chamados via ntfy
```

`-l` zeigt den Benutzer jeder Maschine:

```
IP               USUÁRIO      NOME                                     ESTADO
100.95.244.100   vcatafesta   suporte-vcatafesta-voidbr-liteon-xe03    -
100.95.1.7       root         suporte-root-voidbr-live-ab12            active; direct
```

`-c` **erkennt den Benutzer anhand des Maschinennamens**. Tailscale SSH im Userspace-Modus
Sie können sich nur als **derselbe Benutzer anmelden, der das Skript auf der Fernbedienung ausgeführt hat**, und das ist es, was im Namen steht:

| Wer läuft auf der Fernbedienung | Maschinenname | `-c` verwendet |
|---|---|---|
| `root` (ISO live) | `suporte-root-voidbr-live-x9z8` | `root` |
| `vcatafesta` | `suporte-vcatafesta-voidbr-liteon-xe03` | `vcatafesta` |
| `anon` | `suporte-anon-notebook-a1b2` | `anon` |

- Der Techniker muss auf der Fernbedienung **kein Konto haben**: tailnet autorisiert den Administrator, sich als dieser Benutzer ohne Passwort anzumelden.
- Der Benutzer des Vor-Ort-Technikers spielt keine Rolle.
- Innerhalb der Sitzung verfügt der Techniker über die Berechtigungen des Benutzers; für Root `sudo` (Benutzer gibt Passwort auf gemeinsam genutztem Terminal ein).
- Bei Benutzern mit Zeichen außerhalb von `a-z0-9` (z. B.: `joao.silva`) wird der Name vereinfacht (`joaosilva`);
Geben Sie in diesem Fall den Benutzer ein: `voidbr-suporte -c <ip> joao.silva`.

Wenn mehr als eine Maschine online ist und kein Argument vorhanden ist, zeigt `-c` die Liste an und fragt nach dem Namen oder der IP.

---

## Einstellungen

Datei: `/etc/voidbr-suporte.conf` (über Paketaktualisierungen hinweg beibehalten).

| Option | Standard | Beschreibung |
|---|---|---|
| `modo` | `tailscale` | `tailscale` oder `upterm` |
| `compartilhar` | `1` | `1` = gleiches Terminal (tmux); `0` = Techniker öffnet eigenes Terminal |
| `tmux_sessao` | `suporte` | tmux-Sitzungsname |
| `auto` | `0` (`1` ohne Paket) | `1` = Öffnen Sie den tmux direkt, ohne Eingabe |
| `tmux_powerline` | `0` | `1` = Trennzeichen in der Leiste (benötigt Nerd-Schriftart; erscheint nicht in tty) |
| `ts_authkey` | — | Trennen Sie den OAuth-Client von `?ephemeral=true&preauthorized=true` |
| `ts_tag` | `tag:suporte` | auf die Maschine angewendetes Tag |
| `tecnicos_github` | — | (upterm) autorisierte GitHub-Benutzer |
| `tecnicos_chaves` | `/etc/voidbr-suporte/authorized_keys` | (upterm) chaves extras |
| `servidor` | `ssh://uptermd.upterm.dev:22` | (Upterm) Server |
| `perguntar` | `0` | (upterm) `1` = Benutzer genehmigt jede Verbindung |
| `ntfy_topico` | *(leer)* | ntfy.sh-Thema, um den Techniker zu benachrichtigen; leer ausschalten |
| `ntfy_url` | `https://ntfy.sh` | server ntfy |

---

## Sprachen

Nachrichten verwenden **gettext** (Domäne `voidbr-suporte`). Die Originaltexte liegen vor
**pt_BR**: Ohne installierten Katalog oder ohne den Befehl `gettext` wird alles auf Portugiesisch ausgegeben.

- Kataloge: `/usr/share/locale/<idioma>/LC_MESSAGES/voidbr-suporte.mo`
- Übersetzungen im Repository: `po/<idioma>.po`
- Die Übersetzungsfunktion im Skript ist `_` (Schlüsselwort zur Extraktion: `-k_`)

Auf Englisch zu sehen:

```bash
LANGUAGE=en voidbr-suporte -h
```

> Der Skript-Header setzt `LANGUAGE=pt_BR`, wenn die Variable leer ist. Deshalb,
> Auf einem System mit `LANG=en_US.UTF-8` erscheint Englisch nur, wenn `LANGUAGE=en` definiert ist.

---

## Sicherheit

- **Es gibt kein Zugriffstoken.** Nur diejenigen, die **Tailnet-Administrator** sind, können teilnehmen.
- Die Verbindung ist **WireGuard End-to-End**; Der Tailscale-Server präsentiert der anderen nur eine Seite.
- Support-Maschinen sind **flüchtig**: Sie verlassen das Hecknetz bei `exit` und hinterlassen keinen Schlüssel, keinen Service
oder Rückwärtskonfiguration.
- `-c` zeichnet den Hostschlüssel nicht in `known_hosts` auf, da jeder Dienst eine neue Maschine generiert
und die IP kann wiederholt werden; Die Identität ist durch Tailnet bereits gewährleistet.
- Support-Maschinen erreichen im Tailnet **nichts** (getestet: Verbindung von Remote zu
Techniker besteht nicht).

### Das Geheimnis ist in der Praxis öffentlich

`ts_authkey` geht in das Paket/ISO und **jeder kann es extrahieren**. Mit ihm, jemand
Sie können Maschinen auf Ihrem Hecknetz platzieren, aber:

- immer mit dem Tag `tag:suporte`, ephemera;
- **ohne etwas zu sehen**, dank der oben genannten Richtlinie.

Im schlimmsten Fall tauchen in `-l` seltsame Maschinen auf.

> Verwenden Sie das Geheimnis niemals in einem Tailnet mit der Standardrichtlinie (alles freigegeben): dort, der Maschine
> Der Support hätte Zugriff auf alles.

### Veröffentlichen Sie das Geheimnis niemals auf GitHub

Tailscale nimmt am Secrets-Scan von GitHub teil: ein echter `tskey-client-...` in einem
Das öffentliche Repository wird in der Regel **automatisch widerrufen**. Bewahren Sie im Repository die auf
Beispielwert; Das eigentliche Geheimnis muss im ISO-Build oder nach der Installation platziert werden.

### Wenn das Geheimnis preisgegeben oder missbraucht wird

1. **Einstellungen → Anmeldeinformationen vertrauen** → Anmeldeinformationen `voidbr-suporte` festlegen
2. **Maschinen** → Filtern Sie nach `tag:suporte` und entfernen Sie, was Sie nicht erkennen
3. Generieren Sie einen neuen Berechtigungsnachweis und aktualisieren Sie `.conf`

---

## Upterm-Modus

Alternative ohne Tailnet, mit dem öffentlichen Server [upterm](https://github.com/owenthereal/upterm):

```bash
voidbr-suporte -u
```

- Zeigt den `ssh ...@uptermd.upterm.dev`-Befehl und einen QR-Code (mit `qrencode`)
- Es sind nur Technikerschlüssel enthalten (`tecnicos_github` / `tecnicos_chaves`)
- Der Benutzer muss den Befehl an den Techniker weitergeben (QR-Code, NTFY oder Diktieren).
- Erfordert die Installation der `upterm`-Binärdatei (nicht im Void-Repository)

---

## Häufige Probleme

**`backend error: key tagged with non-existent tag: tag:suporte`**
Die geheime Tailnet-Richtlinie verfügt nicht mehr über den Block `tag:suporte` (`tagOwners`).
Häufige Ursache: Die Richtlinie wurde im falschen Tailnet geändert. Schauen Sie sich das ausgewählte Tailnet an
Konsole und ersetzen Sie die obige Richtlinie.

**`-l` zeigt keine Maschine an, sondern die angeschlossene Fernbedienung**
Der `tailscaled`-Dienst des Technikers befindet sich in einem anderen Tailnet. Erkundigen Sie sich bei
`tailscale switch --list` und enthält `sudo tailscale switch <ID>`.

**`ERRO: ts_authkey não definido` nicht technisch**
Ohne Optionen startet `voidbr-suporte` die **Benutzerseite**. Verwenden Sie im Techniker `-l` und `-c`.

**`can't switch user` / `não consegui entrar como '...' (código 255)`**
Die Anmeldung wurde abgelehnt, da der Benutzer nicht derjenige ist, der das Skript auf der Fernbedienung ausgeführt hat.
Häufige Ursache: Die Fernbedienung hat eine **alte Version** von `voidbr-suporte` (Name ohne Benutzer,
`suporte-<host>-xxxx`) und `-c` lesen den Anfang des Hostnamens als Benutzer.
Aktualisieren Sie die Fernbedienung oder geben Sie den Benutzer ein: `voidbr-suporte -c <ip> <usuario>`.

**`entrou, mas a sessão 'suporte' do tmux não abriu` / `no sessions`**
Der tmux des Benutzers wurde noch nicht geöffnet (bei `auto=0` muss er die Eingabetaste drücken).

**`ssh: Could not resolve hostname suporte-...`**
MagicDNS ist nicht aktiv. Verwenden Sie `voidbr-suporte -c`, das die IP in das Tailnet selbst auflöst.

**Akzente erscheinen als `_`**
Verwenden Sie `voidbr-suporte -c` (es erzwingt UTF-8 mit `tmux -u`) anstelle von reinem `ssh`.

**Tailnet-Gäste haben den Zugang verloren**
Die Richtlinie muss zusätzlich zu `autogroup:member` `autogroup:shared` in der Quelle enthalten
(Wer eine geteilte Maschine erhalten hat, ist kein Mitglied).

**tmux-Barschnitt**
Es nimmt etwa 110 Spalten ein. In tty/VM mit 80 Spalten schneidet tmux den rechten Block ab.

**`tailscaled não subiu`**


**`não consegui conectar; confira a internet e a ts_authkey`**
Überprüfen Sie, ob das Geheimnis vollständig ist (Groß-/Kleinschreibung), in Anführungszeichen gesetzt, mit


---

## 


