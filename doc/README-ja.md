<div align="center">

# 🔵 voidbr-サポート

**高速 tmate スタイルのリモート サポート (Tailscale SSH + 共有 tmux、または upterm)**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge)](ライセンス)

</div>

---

古い `tmate` の精神に基づいた **VoidBR Linux** の高速リモート サポート:
ユーザーが **コマンド** (グラフィカル環境を使用しない tty を含む) を入力し、技術者が
**同じ端末** に入り、一緒に表示して入力します。

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

- ユーザー側のアカウント、ログイン、パスワード、または SSH キーは不要
- 誰もトークンを指示する必要はありません。技術者の目にはマシンが単独で表示されます。
- サービス後にインストールまたは実行されたままになるものはありません
- gettext を使用した翻訳可能なメッセージ (pt_BR および en)
- テールネットがない場合の **upterm** による代替モード

---

## 索引

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

## 仕組み

### ユーザー側

青ロダー `voidbr-suporte`、スクリプト:

1. 一時フォルダーを作成し、**サービス専用の **`tailscaled`** をアップロードします
(ユーザー空間モード、メモリ内のみの状態、独自のソケットとポート)。
2. `/etc/voidbr-suporte.conf` シークレットを使用して技術者のテールネットに入ります。
タグ `tag:suporte` と名前 `suporte-<usuario>-<hostname>-<xxxx>` を持つ **一時** ノードとして。
ユーザーは、技術者に通知することなく入力できる名前を入力します。
3. **Tailscale SSH** をオンにします。tailscaled 自体は sshd なしで SSH を提供します。
4. **tmux** 内のユーザーのシェルを開き、色付きのバーが表示されます。
ユーザー、マシン名、IP、および `exit = encerrar`。
5. 終了する場合 (`exit` または Ctrl+C): `tailscale logout` を実行すると、マシンは **直ちにテールネットを終了します**。
デーモンが終了し、一時フォルダーが削除されます。

### `tailscaled` サービスをリモートで実行する必要がない理由

runit サービスは、起動時に `tailscaled` をオンにするためにのみ使用されます。 `voidbr-suporte` がオンになります
サービス中のみ、自分の `tailscaled`:

| |システムサービス | `voidbr-suporte` デーモン |
|---|---|---|
|ソケット |チリ_REF_0_チリ |チリ_REF_1_チリ |
|ステータス | `/var/lib/tailscale/` (ディスク) |記憶の中だけ |
| UDP ポート | 41641 |無料 (`--port=0`) |
|ネットワーク | `tailscale0` インターフェイスとルート |ユーザー空間、インターフェイスまたはルートなし |
|期間 |いつも |サポート期間中のみ |

それが理由です：

- **ユーザーが助けを求めている間のみ接続を維持します**;技術者は、これ以外ではマシンにアクセスできません。
- **root は必要ありません**: 通常のユーザーはサポートを求めることができます。
- **痕跡を残さない**: ディスクには何も保存されません。
- **ユーザーの Tailscale と共存**: ユーザーがテールネットですでに Tailscale を使用している場合、両方
彼らは混ざり合うことなく並んで走ります。
- **サービスを有効にすることなく、ISO ライブで動作します**。

### コーチ側

```bash
voidbr-suporte -l      # máquinas aguardando suporte
voidbr-suporte -c      # conecta no tmux do usuário
```

`-c` はテールネット自体を介して IP とユーザーを検出します (MagicDNS なしで動作します)
そして常に IP 経由で共有 tmux に直接接続します。

### 外に出ないでください

```
 SUPORTE  vcatafesta   0:bash         suporte-voidbr-liteon-nwhf  100.89.95.104  exit = encerrar
 vermelho azul         janelas        roxo                        verde          amarelo
```

東京の夜のパレットの色。 tty では、tmux はコンソールの色に適応します。

---

## インストール

### 投稿者:pkgmake

```bash
git clone https://github.com/voidlinuxbr/voidbr-suporte.git
cd voidbr-suporte
pkgmake -s -i        # instala as dependências, gera o .xbps e instala
```

### マニュアル

```bash
sudo xbps-install -S tailscale tmux curl qrencode gettext
sudo install -Dm755 usr/bin/voidbr-suporte   /usr/bin/voidbr-suporte
sudo install -Dm644 etc/voidbr-suporte.conf  /etc/voidbr-suporte.conf
```

### 依存関係

|パッケージ |使い方 |
|---|---|
|チリ_REF_0_チリ |テールスケール モード (ユーザーおよび技術者) |
|チリ_REF_0_チリ |共有端末 |
|チリ_REF_0_チリ |翻訳されたメッセージ (これがないと、すべてが pt_BR に表示されます) |
|チリ_REF_0_チリ | ntfy 経由で技術者に通知 (オプション) |
|チリ_REF_0_チリ | QR コード no modo upterm (オプション) |
|チリ_REF_0_チリ | upterm モードのみ (Void リポジトリには含まれません) |

|機械 |パッケージ `tailscale` | `tailscaled` サービス |テールネットログイン |
|---|---|---|---|
| **技術** |はい | **はい、常に実行中です** |はい、管理者アカウントを持つ `tailscale up` |
| **ユーザー** |はい | **必要ありません** |いいえ: スクリプトはシークレットとともに単独で入力されます。

---

## テールネットの準備 (1 回のみ)

Tailscale 管理コンソール内のすべて: <https://login.tailscale.com/admin>

> **推奨:** サポート (組織など) 専用の **テールネット** を使用します。
> 個人のテールネットから切り離してください。個人アカウントを **管理者** として招待します。

> **注意:** アカウントが複数のテールネットに参加している場合は、**
> コンソールで、ポリシーを保存するか資格情報を作成する前に、どれが選択されているかを確認してください**。
> 変更履歴は**ログ→構成**にあります。

### 1. アクセスポリシー

**アクセス コントロール**、JSON エディター。 **まず現在のポリシーをバックアップしてください。**

デフォルトのポリシー (`"src": ["*"], "dst": ["*"]`) は、以下を含むすべてを全員にリリースします。
タグ付けされたマシン。これを使用すると、サポート マシンがネットワーク全体を監視できるようになります。
以下のポリシーにより、**人**は無料でアクセスできるようになり、サポートは分離されたままになります。

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

|ルール |効果 |
|---|---|
|チリ_REF_0_チリ | `tag:suporte` タグを作成します (これがないと、サポートは接続されません)。
|助成金 |メンバーとゲストは、マシン、サブネット、出口ノードなどすべてにアクセスします。
| `ssh` 受け入れる | Tailscale SSH はブラウザの確認を求めずに管理者を受け入れます |
| `tag:suporte` を離れるルールはありません |サポート マシンには何も表示されません。

> SSH が機能するには、**両方**が必要です: ポート 22 へのネットワーク アクセス
> (助成金の対象) **および** `ssh` のルール。

### 2. OAuth クライアント (「招待」)

1. **設定 → 信頼資格情報 → 資格情報 → OAuth**
2. 範囲: **認証キーのみ**、**書き込み**あり
3. タグ: **`tag:suporte` のみ** (ポリシーの保存後にのみ表示されます)
4. 説明: `voidbr-suporte`
5. **資格情報を生成**
6. **クライアント シークレット** (`tskey-client-...`) をコピーします。**二度と表示されません**

サポートには**一意**の資格情報を使用してください。 CI または他のものを再利用しないでください。

### 3. シークレットを `.conf` に入れます

ユーザー マシン (または ISO) の `/etc/voidbr-suporte.conf`:

```bash
ts_authkey="tskey-client-SEU_SECRET?ephemeral=true&preauthorized=true"
```

- `?` と `&` のため引用符が必要です
- クライアントIDは使用されません
- 技術者は**秘密を必要としません**

### 4. コーチ側

```bash
sudo xbps-install -S tailscale tmux
sudo ln -s /etc/sv/tailscaled /var/service/
sudo tailscale up        # conta admin da tailnet do suporte
```

アカウントが複数のテールネット上にある場合、サービス上で一度にアクティブになるのは 1 つだけです。

```bash
tailscale switch --list          # o * marca a ativa
sudo tailscale switch <ID>       # troca de tailnet
```

---

## 使用

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

### ユーザー

```bash
voidbr-suporte
```

`auto=1` (パッケージのデフォルト) を使用すると、名前と IP を含むフレームが 2 秒間表示され、
tmux が自動的に開きます。閉じるには: `exit`。

ユーザーは誰でも構いません: ライブ ISO の `root`、`maria`、`anon`... マシン名
スクリプトを実行した人 (`suporte-anon-notebook-a1b2` など) とともに終了します。

### テクニカル

```bash
voidbr-suporte -l                          # quem está aguardando suporte
voidbr-suporte -c                          # se houver só uma máquina online
voidbr-suporte -c 100.95.244.100           # pelo IP
voidbr-suporte -c liteon                   # por parte do nome
voidbr-suporte -c 100.95.244.100 fulano    # forçando outro usuário
voidbr-suporte -n                          # aguarda chamados via ntfy
```

`-l` は、各マシンのユーザーを示します。

```
IP               USUÁRIO      NOME                                     ESTADO
100.95.244.100   vcatafesta   suporte-vcatafesta-voidbr-liteon-xe03    -
100.95.1.7       root         suporte-root-voidbr-live-ab12            active; direct
```

`-c` **マシン名によってユーザーを検出します**。ユーザー空間モードの Tailscale SSH
**リモートでスクリプトを実行したのと同じユーザー**としてのみ入力できます。それが名前に含まれています。

|リモートで実行する人 |マシン名 | `-c` は | を使用します。
|---|---|---|
| `root` (ISO ライブ) |チリ_REF_1_チリ |チリ_REF_2_チリ |
|チリ_REF_0_チリ |チリ_REF_1_チリ |チリ_REF_2_チリ |
|チリ_REF_0_チリ |チリ_REF_1_チリ |チリ_REF_2_チリ |

- 技術者はリモートに **アカウントを持つ必要はありません**。テールネットは、管理者にパスワードなしでこのユーザーとしてログインすることを許可します。
- オンサイト技術者のユーザーは関係ありません。
- セッション内では、技術者はユーザーの権限を持っています。 root の場合、`sudo` (ユーザーは共有端末でパスワードを入力します)。
- `a-z0-9` 以外の文字を使用するユーザー (例: `joao.silva`) の名前は簡略化されます (`joaosilva`)。
この場合、ユーザー「`voidbr-suporte -c <ip> joao.silva`」を入力します。

複数のマシンがオンラインで引数なしの場合、`-c` はリストを表示し、名前または IP を尋ねます。

---

## 設定

ファイル: `/etc/voidbr-suporte.conf` (パッケージの更新後も保持されます)。

|オプション |標準 |説明 |
|---|---|---|
|チリ_REF_0_チリ |チリ_REF_1_チリ | `tailscale` または `upterm` |
|チリ_REF_0_チリ |チリ_REF_1_チリ | `1` = 同じ端末 (tmux); `0` = 技術者が自分の端末を開きます |
|チリ_REF_0_チリ |チリ_REF_1_チリ | tmux セッション名 |
|チリ_REF_0_チリ | `0` (`1` ノーパコート) | `1` = 多重化ディレクトリを使用して、必要な情報を入力します。
|チリ_REF_0_チリ |チリ_REF_1_チリ | `1` = バー内の区切り記号 (Nerd Font が必要; tty には表示されません) |
|チリ_REF_0_チリ | — | segredo do OAuth クライアント com `?ephemeral=true&preauthorized=true` |
|チリ_REF_0_チリ |チリ_REF_1_チリ |マシンに適用されるタグ |
|チリ_REF_0_チリ | — | (期限内) 承認された GitHub ユーザー |
|チリ_REF_0_チリ |チリ_REF_1_チリ | (アップターム) チャベス エクストラ |
|チリ_REF_0_チリ |チリ_REF_1_チリ | (任期中) 奉仕者 |
|チリ_REF_0_チリ |チリ_REF_1_チリ | (upterm) `1` = ユーザーは各接続を承認します |
|チリ_REF_0_チリ | *(空)* | ntfy.sh トピックを使用して技術者に通知します。空の電源をオフにする |
|チリ_REF_0_チリ |チリ_REF_1_チリ |サーバーntfy |

---

## 言語

メッセージは **gettext** (ドメイン `voidbr-suporte`) を使用します。原文は以下のとおりです
**pt_BR**: カタログがインストールされていない場合、または `gettext` コマンドがない場合は、すべてポルトガル語で表示されます。

- カタログ: `/usr/share/locale/<idioma>/LC_MESSAGES/voidbr-suporte.mo`
- リポジトリ内の翻訳: `po/<idioma>.po`
- スクリプト内の変換関数は`_`（抽出用キーワード：`-k_`）です。

英語で見るには:

```bash
LANGUAGE=en voidbr-suporte -h
```

> 変数が空の場合、スクリプト ヘッダーは `LANGUAGE=pt_BR` を設定します。それが理由です、
> `LANG=en_US.UTF-8` を含むシステムでは、`LANGUAGE=en` が定義されている場合にのみ英語が表示されます。

---

## 安全

- **アクセス トークンはありません。** **テールネット管理者**のみがアクセスできます。
- 接続は **WireGuard エンドツーエンド** です。 Tailscale サーバーは一方の側を他方に提示するだけです。
- サポート マシンは **一時的**です。サポート マシンは `exit` でテールネットを離れ、キーやサービスを残しません。

- 各サービスが新しいマシンを生成するため、`-c` は `known_hosts` にホスト キーを記録しません。
IP は繰り返すことができます。 ID はテールネットによってすでに保証されています。
- サポート マシンはテールネット上で **何も達成しません** (テスト済み: リモートから
技術者は合格しません）。

### 秘密は実際に公開されます

`ts_authkey` はパッケージ/ISO 内にあり、**誰でも抽出できます**。彼と一緒に、誰かが
テールネット上にマシンを配置できますが、次のことが可能です。

- 常にタグ `tag:suporte`、エフェメラが付きます。
- 上記のポリシーのおかげで、**何も表示されずに**。

最悪の場合、`-l` に奇妙なマシンが出現します。

> デフォルト ポリシー (すべてがリリースされる) を使用してテールネットでシークレットを使用しないでください。そこにはマシンが存在します。
> サポートはすべてにアクセスできるようになります。

### GitHub でシークレットを公開しないでください

Tailscale が GitHub シークレット スキャンに参加: 1 つの本物の `tskey-client-...`
パブリック リポジトリは **自動的に取り消される**傾向があります。リポジトリ内に、
値の例。本当のシークレットは ISO ビルド内、またはインストール後に配置する必要があります。

### 秘密が漏洩または悪用された場合

1. **設定 → 認証情報を信頼する** → 認証情報を非表示 `voidbr-suporte`
2. **マシン** → `tag:suporte` でフィルタリングし、認識できないものを削除します
3. 新しい認証情報を生成し、`.conf` を更新します

---

## アップタームモード

テールネットを使用せず、[upterm](https://github.com/owenthereal/upterm) パブリック サーバーを使用する代替方法:

```bash
voidbr-suporte -u
```

- `ssh ...@uptermd.upterm.dev` コマンドと QR コード (`qrencode` 付き) を表示します。
- 技術者のキーのみが含まれます (`tecnicos_github` / `tecnicos_chaves`)
- ユーザーは技術者にコマンドを渡す必要があります (QR コード、ntfy、または口述入力)。
- `upterm` バイナリがインストールされている必要があります (Void リポジトリにはありません)。

---

## 

**チリ_REF_0_チリ**
シークレット テールネット ポリシーには `tag:suporte` (`tagOwners` ブロック) がなくなりました。
一般的な原因: ポリシーが間違ったテールネットで変更されました。選択したテールネットをチェックアウトします


**`-l` にはマシンが表示されませんが、リモートが接続されています**



**`ERRO: ts_authkey não definido` 非技術系**


*





*


*


*


*



*


*


*



---

## 


