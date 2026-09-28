<div 정렬="중앙">

# 🔵 voidbr-지원

**빠른 tmate 스타일 원격 지원(Tailscale SSH + 공유 tmux 또는 upterm)**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge)](라이센스)

</div>

---

이전 `tmate`의 정신으로 **VoidBR Linux**에 대한 빠른 원격 지원:
사용자는 **명령**(그래픽 환경 없이 tty 포함)을 입력하고 기술자는
**동일 터미널**에 들어가서 함께 보고 입력하세요.

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

- 사용자 측 계정, 로그인, 비밀번호 또는 SSH 키 없음
- 누구도 토큰을 지시할 필요가 없습니다. 기계는 기술자에게 단독으로 나타납니다.
- 서비스 후에는 아무것도 설치되거나 실행되지 않습니다.
- gettext를 사용하여 번역 가능한 메시지(pt_BR 및 en)
- tailnet이 없는 경우 **upterm**을 통한 대체 모드

---

## 색인

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

## 작동 방식

### 사용자 측

Ao rodar `voidbr-suporte`, o 스크립트:

1. 임시 폴더를 생성하고 서비스 전용 **`tailscaled`**를 업로드하세요.
(사용자 공간 모드, 메모리에만 상태, 자체 소켓 및 포트)
2. `/etc/voidbr-suporte.conf` 비밀을 사용하여 기술자의 테일넷을 입력합니다.
`tag:suporte` 태그와 `suporte-<usuario>-<hostname>-<xxxx>` 이름을 가진 **임시** 노드로 사용됩니다.
사용자는 기술자에게 알리지 않고 입력할 이름을 입력합니다.
3. **Tailscale SSH** 켜기: tailscaled 자체가 sshd 없이 SSH를 제공합니다.
4. 컬러 막대가 표시된 **tmux** 내부에서 사용자 쉘을 엽니다.
사용자, 컴퓨터 이름, IP 및 `exit = encerrar`.
5. 종료할 때(`exit` 또는 Ctrl+C): `tailscale logout`를 수행하면 머신이 **즉시 tailnet을 종료합니다**,
데몬이 종료되고 임시 폴더가 삭제됩니다.

### `tailscaled` 서비스를 원격에서 실행할 필요가 없는 이유

runit 서비스는 부팅 시 `tailscaled`를 켜는 데에만 사용됩니다. `voidbr-suporte`가 켜집니다.
서비스 중에만 자신의 `tailscaled`:

| | 시스템 서비스 | `voidbr-suporte` 데몬 |
|---|---|---|
| 소켓 | `/run/tailscale/tailscaled.sock` | `/tmp/voidbr-suporte.XXXX/tailscaled.sock` |
| 상태 | `/var/lib/tailscale/`(디스크) | 메모리에서만 |
| UDP 포트 | 41641 | 모든 무료(`--port=0`) |
| 네트워크 | `tailscale0` 인터페이스 및 경로 | 인터페이스나 경로가 없는 사용자 공간 |
| 기간 | 항상 | 지원 중에만 |

그 이유는 다음과 같습니다.

- **사용자가 도움을 원하는 동안에만 연결 상태를 유지합니다**; 기술자는 이 외부에서 기계에 접근하지 않습니다.
- **루트 필요 없음**: 일반 사용자가 지원을 요청할 수 있습니다.
- **추적을 남기지 않음**: 디스크에 아무 것도 남지 않습니다.
- **사용자의 Tailscale과 공존**: 이미 tailnet에서 Tailscale을 사용하는 경우 둘 다
섞이지 않고 나란히 달린다.
- **서비스를 활성화하지 않고도 ISO 라이브에서 작동**합니다.

### 코치 측

```bash
voidbr-suporte -l      # máquinas aguardando suporte
voidbr-suporte -c      # conecta no tmux do usuário
```

`-c`는 테일넷 자체를 통해 IP와 사용자를 검색합니다(MagicDNS 없이 작동).
항상 IP를 통해 공유 tmux에 직접 연결됩니다.

### 밖에 나가지 마세요

```
 SUPORTE  vcatafesta   0:bash         suporte-voidbr-liteon-nwhf  100.89.95.104  exit = encerrar
 vermelho azul         janelas        roxo                        verde          amarelo
```

Tokyo Night 팔레트의 색상. tty에서 tmux는 콘솔 색상에 맞춰집니다.

---

## 설치

### 작성자: pkgmake

```bash
git clone https://github.com/voidlinuxbr/voidbr-suporte.git
cd voidbr-suporte
pkgmake -s -i        # instala as dependências, gera o .xbps e instala
```

### 수동

```bash
sudo xbps-install -S tailscale tmux curl qrencode gettext
sudo install -Dm755 usr/bin/voidbr-suporte   /usr/bin/voidbr-suporte
sudo install -Dm644 etc/voidbr-suporte.conf  /etc/voidbr-suporte.conf
```

### 종속성

| 패키지 | 사용법 |
|---|---|
| `tailscale` | tailscale 모드(사용자 및 기술자) |
| `tmux` | 공유 터미널 |
| `gettext` | 번역된 메시지(그것이 없으면 모든 것이 pt_BR로 나옵니다) |
| `curl` | ntfy를 통해 기술자에게 알림(선택 사항) |
| `qrencode` | QR 코드 모드 업그레이드 없음(선택 사항) |
| `upterm` | 최신 모드에만 해당(Void 저장소에는 없음) |

| 기계 | 패키지 `tailscale` | `tailscaled` 서비스 | 테일넷 로그인 |
|---|---|---|---|
| **기술** | 예 | **예, 항상 실행 중입니다** | 예, 관리자 계정의 `tailscale up` |
| **사용자** | 예 | **필요없음** | no: 스크립트는 비밀 |

---

## 테일넷 준비(1회만)

Tailscale 관리 콘솔의 모든 것: <https://login.tailscale.com/admin>

> **권장:** 지원을 위해 **전용 테일넷**을 사용합니다(예: 조직의 지원).
> 개인 테일넷과 분리됩니다. 개인 계정을 **관리자**로 초대하세요.

> **주의:** 귀하의 계정이 하나 이상의 테일넷에 참여하는 경우 **상단에서 확인하세요.
> 정책을 저장하거나 자격 증명을 생성하기 전에 선택된 콘솔**입니다.
> 변경 내역은 **로그 → 구성**에 있습니다.

### 1. 접근정책

**액세스 제어**의 JSON 편집기에서 **먼저 현재 정책을 백업해 주세요.**

기본 정책(`"src": ["*"], "dst": ["*"]`)은 다음을 포함한 모든 사람에게 모든 것을 해제합니다.
태그가 달린 기계. 이를 통해 지원 시스템은 전체 네트워크를 볼 수 있습니다.
아래 정책은 **사람들**에게 무료 액세스를 유지하고 지원을 격리시킵니다.

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

| 규칙 | 효과 |
|---|---|
| `tagOwners` | `tag:suporte` 태그 생성(이 태그가 없으면 지원이 연결되지 않음) |
| 부여 | 구성원과 게스트는 머신, 서브넷, 종료 노드 등 모든 것에 액세스합니다.
| `ssh` 동의 | Tailscale SSH는 브라우저 확인을 요청하지 않고 관리자를 허용합니다 |
| `tag:suporte`를 떠나는 규칙이 없습니다 | 지원 기계는 아무것도 보지 못한다 |

> SSH가 작동하려면 **두 가지**가 필요합니다: 포트 22에 대한 네트워크 액세스
> (부여 적용) **및** `ssh`의 규칙.

### 2. OAuth 클라이언트(“초대”)

1. **설정 → 신뢰 자격증명 → 자격증명 → OAuth**
2. 범위: **인증 키만**, **쓰기** 포함
3. 태그: **`tag:suporte`만**(정책을 저장한 후에만 표시됨)
4. 설명: `voidbr-suporte`
5. **사용자 인증 정보 생성**
6. **클라이언트 비밀번호**(`tskey-client-...`)를 복사하세요. **다시 표시되지 않습니다**

지원을 위해 **고유** 자격 증명을 사용하세요. CI 또는 다른 CI를 재사용하지 마십시오.

### 3. `.conf`에 비밀번호를 입력하세요.

사용자 머신(또는 ISO)의 `/etc/voidbr-suporte.conf`에서:

```bash
ts_authkey="tskey-client-SEU_SECRET?ephemeral=true&preauthorized=true"
```

- `?` 및 `&`로 인해 따옴표가 필요합니다.
- 클라이언트 ID가 사용되지 않습니다.
- 기술자에게는 비밀이 **필요하지 않습니다**

### 4. 코치 측

```bash
sudo xbps-install -S tailscale tmux
sudo ln -s /etc/sv/tailscaled /var/service/
sudo tailscale up        # conta admin da tailnet do suporte
```

귀하의 계정이 두 개 이상의 tailnet에 있는 경우 서비스에서는 한 번에 하나만 활성화됩니다.

```bash
tailscale switch --list          # o * marca a ativa
sudo tailscale switch <ID>       # troca de tailnet
```

---

## 사용

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

### 사용자

```bash
voidbr-suporte
```

`auto=1`(패키지 기본값)를 사용하면 이름과 IP가 포함된 프레임이 2초 동안 나타나고
tmux가 자동으로 열립니다. 닫으려면: `exit`.

사용자는 누구든지 될 수 있습니다: 라이브 ISO의 `root`, `maria`, `anon`… 머신 이름
스크립트를 실행한 사람과 함께 종료됩니다(예: `suporte-anon-notebook-a1b2`).

### 인위적인

```bash
voidbr-suporte -l                          # quem está aguardando suporte
voidbr-suporte -c                          # se houver só uma máquina online
voidbr-suporte -c 100.95.244.100           # pelo IP
voidbr-suporte -c liteon                   # por parte do nome
voidbr-suporte -c 100.95.244.100 fulano    # forçando outro usuário
voidbr-suporte -n                          # aguarda chamados via ntfy
```

`-l`는 각 시스템의 사용자를 보여줍니다.

```
IP               USUÁRIO      NOME                                     ESTADO
100.95.244.100   vcatafesta   suporte-vcatafesta-voidbr-liteon-xe03    -
100.95.1.7       root         suporte-root-voidbr-live-ab12            active; direct
```

`-c` **시스템 이름으로 사용자를 검색합니다**. 사용자 공간 모드의 Tailscale SSH
원격에서 **스크립트를 실행한 동일한 사용자**로만 입력할 수 있으며 이것이 이름에 나와 있습니다.

| 원격으로 실행되는 사람 | 기계 이름 | `-c`는 |
|---|---|---|
| `root`(ISO 라이브) | `suporte-root-voidbr-live-x9z8` | `root` |
| `vcatafesta` | `suporte-vcatafesta-voidbr-liteon-xe03` | `vcatafesta` |
| `anon` | `suporte-anon-notebook-a1b2` | `anon` |

- 기술자는 원격지에 **계정이 필요하지 않습니다**. tailnet은 관리자에게 비밀번호 없이 이 사용자로 로그인할 수 있는 권한을 부여합니다.
- 현장 기술자의 사용자는 중요하지 않습니다.
- 세션 내에서 기술자는 사용자의 권한을 갖습니다. 루트의 경우 `sudo`(사용자가 공유 터미널에 비밀번호를 입력함).
- `a-z0-9` 이외의 문자(예: `joao.silva`)를 사용하는 사용자의 이름은 단순화됩니다(`joaosilva`).
이 경우 사용자 `voidbr-suporte -c <ip> joao.silva`를 입력합니다.

둘 이상의 머신이 온라인이고 인수가 없는 경우 `-c`는 목록을 표시하고 이름이나 IP를 묻습니다.

---

## 설정

파일: `/etc/voidbr-suporte.conf`(패키지 업데이트 전반에 걸쳐 유지됨)

| 옵션 | 표준 | 설명 |
|---|---|---|
| `modo` | `tailscale` | `tailscale` 또는 `upterm` |
| `compartilhar` | `1` | `1` = 동일한 터미널(tmux); `0` = 기술자가 자체 터미널을 엽니다 |
| `tmux_sessao` | `suporte` | tmux 세션 이름 |
| `auto` | `0`(`1` 없음) | `1` = abre o tmux direto, sem pedir Enter |
| `tmux_powerline` | `0` | `1` = 막대의 구분 기호(Nerd 글꼴 필요, tty에는 표시되지 않음) |
| `ts_authkey` | — | OAuth 클라이언트 com `?ephemeral=true&preauthorized=true` 수행 |
| `ts_tag` | `tag:suporte` | 기계에 적용된 태그 |
| `tecnicos_github` | — | (최신) 승인된 GitHub 사용자 |
| `tecnicos_chaves` | `/etc/voidbr-suporte/authorized_keys` | (upterm) chaves 엑스트라 |
| `servidor` | `ssh://uptermd.upterm.dev:22` | (상위) 하인 |
| `perguntar` | `0` | (upterm) `1` = 사용자가 각 연결을 승인합니다 |
| `ntfy_topico` | *(비어 있음)* | 기술자에게 알리는 ntfy.sh 주제; 빈 끄기 |
| `ntfy_url` | `https://ntfy.sh` | 서버 NTFY |

---

## 언어

메시지는 **gettext**(도메인 `voidbr-suporte`)를 사용합니다. 원본 텍스트는 에 있습니다.
**pt_BR**: 카탈로그가 설치되지 않거나 `gettext` 명령이 없으면 모든 것이 포르투갈어로 나옵니다.

- 카탈로그: `/usr/share/locale/<idioma>/LC_MESSAGES/voidbr-suporte.mo`
- 저장소의 번역: `po/<idioma>.po`
- 스크립트의 번역 함수는 `_`입니다(추출용 키워드: `-k_`).

영어로 보려면:

```bash
LANGUAGE=en voidbr-suporte -h
```

> 스크립트 헤더는 변수가 비어 있을 때 `LANGUAGE=pt_BR`를 설정합니다. 그렇기 때문에,
> `LANG=en_US.UTF-8`가 있는 시스템에서는 `LANGUAGE=en`가 정의된 영어만 표시됩니다.

---

## 보안

- **접근 토큰이 없습니다.** **tailnet admin**만 입장 가능합니다.
- 연결은 **WireGuard 종단 간**입니다. Tailscale 서버는 한쪽 면만 다른 쪽 면에 제공합니다.
- 지원 시스템은 **일시적**입니다. `exit`에서 tailnet을 떠나고 키, 서비스를 남기지 않습니다.
또는 이전 구성.
- `-c`는 각 서비스가 새 시스템을 생성하므로 `known_hosts`에 호스트 키를 기록하지 않습니다.
IP는 반복될 수 있습니다. 이미 tailnet을 통해 신원이 보장되어 있습니다.
- 지원 시스템은 tailnet에서 **아무것도 달성하지 못합니다**(테스트됨: 원격에서 원격으로 연결)
기술자는 합격하지 못함).

### 비밀은 실제로 공개됩니다

`ts_authkey`는 패키지/ISO 내부로 들어가며 **누구나 추출할 수 있습니다**. 그 사람이랑 누군가
tailnet에 머신을 배치할 수 있지만 다음과 같습니다.

- 항상 `tag:suporte`, ephemera 태그가 있습니다.
- **아무것도 보이지 않음**, 위의 정책 덕분입니다.

최악의 경우 `-l`에 이상한 컴퓨터가 나타납니다.

> 기본 정책(모든 것이 공개됨)으로 tailnet에서 비밀을 사용하지 마십시오.
> 지원팀은 모든 것에 액세스할 수 있습니다.

### GitHub에 비밀을 게시하지 마세요.

Tailscale은 GitHub 비밀 스캔에 참여합니다. 실제 `tskey-client-...`가 하나에 포함되어 있습니다.
공개 저장소는 **자동으로 취소**되는 경향이 있습니다. 저장소에서
예시값; 실제 비밀은 ISO 빌드에 있거나 설치 후에 배치되어야 합니다.

### 비밀이 유출되거나 남용된 경우

1. **설정 → 신임 정보 신뢰** → 신임 정보 `voidbr-suporte` 아파치
2. **기계** → `tag:suporte`로 필터링하고 인식하지 못하는 항목을 제거하세요.
3. 새 자격 증명을 생성하고 `.conf`를 업데이트하세요.

---

## Upterm 모드

[upterm](https://github.com/owenthereal/upterm) 공용 서버를 사용하는 tailnet 없는 대안:

```bash
voidbr-suporte -u
```

- `ssh ...@uptermd.upterm.dev` 명령 및 QR 코드 표시(`qrencode` 포함)
- 기술자의 키만 포함됩니다. (`tecnicos_github` / `tecnicos_chaves`)
- 사용자는 기술자에게 명령(QR 코드, ntfy 또는 받아쓰기)을 전달해야 합니다.
- `upterm` 바이너리가 설치되어 있어야 합니다(Void 저장소에는 없음).

---

## 일반적인 문제

**`backend error: key tagged with non-existent tag: tag:suporte`**
비밀 tailnet 정책에는 더 이상 `tag:suporte`(`tagOwners` 블록)가 없습니다.
일반적인 원인: 잘못된 tailnet에서 정책이 변경되었습니다. 선택한 테일넷을 확인해보세요.
콘솔을 열고 위 정책을 교체하세요.

**`-l`에는 컴퓨터가 표시되지 않지만 원격으로 연결되어 있음**
기술자의 `tailscaled` 서비스는 다른 테일넷에 있습니다. 확인해보세요
`tailscale switch --list` 및 `sudo tailscale switch <ID>`에 대한 이야기입니다.

**`ERRO: ts_authkey não definido` 비기술적**
옵션이 없으면 `voidbr-suporte`는 **사용자** 측을 시작합니다. 기술자에서는 `-l` 및 `-c`를 사용합니다.

**`can't switch user` / `não consegui entrar como '...' (código 255)`**
사용자가 원격에서 스크립트를 실행한 사람이 아니기 때문에 로그인이 거부되었습니다.
일반적인 원인: 리모컨에 `voidbr-suporte`의 **이전 버전**이 있습니다(사용자가 없는 이름,
`suporte-<host>-xxxx`) 및 `-c`는 사용자로 호스트 이름의 시작 부분을 읽습니다.
리모컨을 업데이트하거나 사용자 `voidbr-suporte -c <ip> <usuario>`를 입력하세요.

**`entrou, mas a sessão 'suporte' do tmux não abriu` / `no sessions`**
사용자의 tmux가 아직 열리지 않았습니다(`auto=0`를 사용하려면 Enter를 눌러야 함).

**`ssh: Could not resolve hostname suporte-...`**


**`_`로 나타나는 악센트**
직선 `ssh` 대신 `voidbr-suporte -c`(`tmux -u`를 사용하여 UTF-8을 강제 적용)를 사용합니다.

**Tailnet 손님은 액세스할 수 없습니다**



*


*


*



---

## 


