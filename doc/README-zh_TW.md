<div對齊=“中心”>

# 🔵 voidbr-支持

**快速 tmate 式遠端支援（Tailscale SSH + 共用 tmux 或 upterm）**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge)]（許可證）

</div>

---

本著舊 `tmate` 的精神，對 **VoidBR Linux** 進行快速遠端支援：
使用者鍵入 *命令**（包括在 tty 中，沒有圖形環境），技術人員
進入**同一終端機**，一起查看和輸入。

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

- 無用戶端帳戶、登入名稱、密碼或 SSH 金鑰
- 沒有人需要口授令牌：機器對技術人員來說是單獨出現的
- 維修後沒有任何東西保持安裝或運行
- 可使用 gettext 翻譯的訊息（pt_BR 和 en）
- 透過 **upterm** 的替代模式，適用於沒有尾網的情況

---

## 指數

- 辣椒_REF_0_辣椒
- 辣椒_REF_0_辣椒
- 辣椒_REF_0_辣椒
- 辣椒_REF_0_辣椒
- 辣椒_REF_0_辣椒
- 辣椒_REF_0_辣椒
- 辣椒_REF_0_辣椒
- 辣椒_REF_0_辣椒
- 辣椒_REF_0_辣椒

---

## 它是如何運作的

### 使用者側

奧羅德`voidbr-suporte`，奧腳本：

1. 建立一個臨時資料夾並上傳 **`tailscaled` 僅用於服務**
（使用者空間模式，狀態僅在記憶體中，擁有自己的套接字和連接埠）。
2. 使用`/etc/voidbr-suporte.conf`秘密進入技術人員的尾網，
作為具有標籤 `tag:suporte` 和名稱 `suporte-<usuario>-<hostname>-<xxxx>` 的 **暫存** 節點。

3. 開啟 **Tailscale SSH**：tailscaled 本身提供 SSH，無需 sshd。
4. 在 **tmux** 內開啟使用者的 shell，並顯示彩色條
使用者、機器名稱、IP 和 `exit = encerrar`。
5. 退出時（`exit`或Ctrl+C）：執行`tailscale logout`，機器**立即登出尾網**，
守護程序被終止，臨時資料夾被刪除。

### 為什麼`tailscaled`服務不需要在遠端運行

runit 服務僅用於在啟動時開啟 `tailscaled`。 `voidbr-suporte` 开启
您自己的 `tailscaled`，僅在服務期間：

| |系統服務| `voidbr-suporte` 守護程式 |
|---|---|---|
|插座|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 |
|狀態 | `/var/lib/tailscale/`（磁碟）|只存在記憶中|
| UDP連接埠| 41641 | 41641任何免費的 (`--port=0`) |
|網路| `tailscale0` 介面與路線 |使用者空間，沒有介面或路由|
|持續時間 |總是|僅在支援期間|

這就是為什麼：

- **僅在使用者需要協助時保持連線**；技術人員無法在此之外存取機器。
- **無需root**：普通用戶可以尋求支援。
- **不留下任何痕跡**：沒有任何內容寫入磁碟。
- **與使用者的 Tailscale 共存**：如果他已經在其 tailnet 上使用 Tailscale，則兩者
它們並排運行，沒有混合。
- **適用於 ISO live**，無需啟用任何服務。

### 教練側

```bash
voidbr-suporte -l      # máquinas aguardando suporte
voidbr-suporte -c      # conecta no tmux do usuário
```

`-c` 透過尾網本身發現 IP 和使用者（無需 MagicDNS 即可運作）
並且始終透過 IP 直接連接到共用 tmux。

### 不要出去

```
 SUPORTE  vcatafesta   0:bash         suporte-voidbr-liteon-nwhf  100.89.95.104  exit = encerrar
 vermelho azul         janelas        roxo                        verde          amarelo
```

東京之夜調色盤中的顏色。在 tty 中，tmux 會適應控制台顏色。

---

## 安裝

### 

```bash
git clone https://github.com/voidlinuxbr/voidbr-suporte.git
cd voidbr-suporte
pkgmake -s -i        # instala as dependências, gera o .xbps e instala
```

### 手動的

```bash
sudo xbps-install -S tailscale tmux curl qrencode gettext
sudo install -Dm755 usr/bin/voidbr-suporte   /usr/bin/voidbr-suporte
sudo install -Dm644 etc/voidbr-suporte.conf  /etc/voidbr-suporte.conf
```

### 依賴關係

|套餐 |用途 |
|---|---|
|辣椒_REF_0_辣椒 |尾秤模式（使用者和技術人員）|
|辣椒_REF_0_辣椒 |共享終端|
|辣椒_REF_0_辣椒 |翻譯後的訊息（沒有它，所有內容都會在 pt_BR 中顯示）|
|辣椒_REF_0_辣椒 |透過 ntfy 向技術人員發出通知（可選）|
|辣椒_REF_0_辣椒 | QR 碼 no modo upterm（可選）|
|辣椒_REF_0_辣椒 |僅適用於 upterm 模式（不在 Void 儲存庫中）|

|機|套件 `tailscale` | `tailscaled` 服務 |尾網登入 |
|---|---|---|---|
| **技術** |是的 | **是的，始終運行** |是的，`tailscale up` 具有管理員帳戶 |
| **使用者** |是的 | **不需要** |否：腳本與秘密一起單獨輸入 |

---

## 尾網準備（僅一次）

Tailscale 管理控制台中的所有內容：<https://login.tailscale.com/admin>

> **推薦：**使用**尾網專用**來支援（例如組織的），
> 與您的個人尾網分開。邀請您的個人帳戶作為**管理員**。

> **注意：**如果您的帳戶參與多個尾網，**請查看頂部的
> 儲存策略或建立憑證之前選擇的控制台**。
> 變更歷史記錄位於 **日誌 → 設定**。

### 1. 准入政策

在**存取控制**中，在 JSON 編輯器中。 **請先備份您目前的政策。 **

預設策略（`"src": ["*"], "dst": ["*"]`）向所有人發布所有內容，包括
标记的机器。有了它，支援機器就可以看到整個網路。


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

|規則|效果|
|---|---|
|辣椒_REF_0_辣椒 |建立 `tag:suporte` 標籤（沒有它，支援將無法連接） |
|授予|會員與訪客進入一切：機器、子網路、出口節點 |
| `ssh` 接受 | Tailscale SSH 無需瀏覽器確認即可接受管理員管理 |
|沒有規則離開`tag:suporte` |支援機器什麼也看不見|

> 要讓 SSH 工作，您需要**兩件事**：對連接埠 22 的網路存取
>（由撥款涵蓋）**和** `ssh` 中的規則。

### 2. OAuth 客戶端（「邀請」）

1. **設定 → 信任憑證 → 憑證 → OAuth**
2. 範圍：**僅限身份驗證金鑰**，帶有**寫入**
3. 標籤：**僅 `tag:suporte`**（僅在儲存策略後出現）
4. 說明：`voidbr-suporte`
5. **產生憑證**
6. 複製 **客戶端密鑰** (`tskey-client-...`)：它 **不會再出現**

使用**唯一**憑證來獲得支援；不要重複使用 CI 或其他 CI。

### 3. 將秘密放入`.conf`

在使用者電腦上（或在 ISO 中），在 `/etc/voidbr-suporte.conf` 中：

```bash
ts_authkey="tskey-client-SEU_SECRET?ephemeral=true&preauthorized=true"
```

- 由於 `?` 和 `&`，因此需要引號
- 未使用客戶端 ID
- 技術人員**不需要**需要秘密

### 4. 教練方

```bash
sudo xbps-install -S tailscale tmux
sudo ln -s /etc/sv/tailscaled /var/service/
sudo tailscale up        # conta admin da tailnet do suporte
```

如果您的帳戶位於多個 tailnet 上，則一次只有一個在該服務上處於活動狀態：

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

### 使用者

```bash
voidbr-suporte
```

使用`auto=1`（軟體包預設值），帶有名稱和IP的訊框會出現2秒，然後
tmux 自行打開。關閉：`exit`。

使用者可以是任何人：實時 ISO 中的 `root`、`maria`、`anon`... 機器名稱
與正在執行該腳本的人一起退出，例如 `suporte-anon-notebook-a1b2`。

### 技術的

```bash
voidbr-suporte -l                          # quem está aguardando suporte
voidbr-suporte -c                          # se houver só uma máquina online
voidbr-suporte -c 100.95.244.100           # pelo IP
voidbr-suporte -c liteon                   # por parte do nome
voidbr-suporte -c 100.95.244.100 fulano    # forçando outro usuário
voidbr-suporte -n                          # aguarda chamados via ntfy
```

`-l` 顯示每台機器的使用者：

```
IP               USUÁRIO      NOME                                     ESTADO
100.95.244.100   vcatafesta   suporte-vcatafesta-voidbr-liteon-xe03    -
100.95.1.7       root         suporte-root-voidbr-live-ab12            active; direct
```

`-c` **透過電腦名稱發現使用者**。使用者空間模式下的 Tailscale SSH
它只允許您以**在遠端執行腳本的相同使用者身分**輸入，這就是名稱中的內容：

|誰在遠端執行|機器名稱 | `-c` 使用 |
|---|---|---|
| `root`（ISO 實時）|辣椒_REF_1_辣椒 |辣椒_REF_2_辣椒 |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 |辣椒_REF_2_辣椒 |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 |辣椒_REF_2_辣椒 |

- 技術人員**不需要在遠端擁有帳戶**：tailnet 授權管理員以該使用者身分登錄，無需密碼。
- 現場技術人員的使用者並不重要。
- 在會話內，技術人員擁有使用者的權限；對於 root，`sudo`（使用者在共用終端機上輸入密碼）。
- 字元位於 `a-z0-9` 以外的使用者（例如：`joao.silva`）的名稱將會簡化（`joaosilva`）；
在本例中，輸入使用者：`voidbr-suporte -c <ip> joao.silva`。

當有多台機器在線且沒有參數時，`-c` 顯示清單並詢問名稱或 IP。

---

## 設定

檔案：`/etc/voidbr-suporte.conf`（在套件更新中保留）。

|選項|標準|說明 |
|---|---|---|
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 | `tailscale` 或 `upterm` |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 | `1` = 相同終端機 (tmux); `0` = 技術人員開啟自己的終端 |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 | tmux 會話名稱 |
|辣椒_REF_0_辣椒 | `0`（`1` 無包）| `1` = abre o tmux direto，sem pedir Enter |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 | `1` = 欄位中的分隔符號（需要 Nerd 字型；不會出現在 tty 中）|
|辣椒_REF_0_辣椒 | — | OAuth 用戶端 com `?ephemeral=true&preauthorized=true` | segredo
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 |標籤應用於機器|
|辣椒_REF_0_辣椒 | — | (upterm) 授權 GitHub 使用者 |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 | (upterm) 查維斯演員 |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 | (upterm) 服務商 |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 | (upterm) `1` = 用戶批准每個連接 |
|辣椒_REF_0_辣椒 | *（空）* | ntfy.sh主題通知技術人員；空關掉|
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 |服務器 ntfy |

---

## 語言

訊息使用 **gettext**（域 `voidbr-suporte`）。原文位於
**pt_BR**：如果沒有安裝目錄，或沒有 `gettext` 指令，則所有內容都會以葡萄牙文顯示。

- 目錄：`/usr/share/locale/<idioma>/LC_MESSAGES/voidbr-suporte.mo`
- 儲存庫中的翻譯：`po/<idioma>.po`
- 腳本中的翻譯函數為`_`（擷取關鍵字：`-k_`）

要看英文版：

```bash
LANGUAGE=en voidbr-suporte -h
```

> 當變數為空時，腳本頭會設定 `LANGUAGE=pt_BR`。這就是為什麼，
> 在具有 `LANG=en_US.UTF-8` 的系統上，僅在定義了 `LANGUAGE=en` 時才顯示英文。

---

## 安全

- **沒有訪問令牌。 **只有**尾網管理員**才能進入。
- 連接是 **WireGuard 端對端**； Tailscale 伺服器僅將一側呈現給另一側。
- 支援機器是**短暫的**：它們將尾網留在`exit`並且不留下任何密鑰，服務
或向後配置。
- `-c`不會在`known_hosts`中記錄主機金鑰，因為每個服務都會產生一台新機器
且IP可以重複；身分已經由 tailnet 保證。
- 支援機器**在尾網上沒有實現任何目標**（已測試：從遠端連接到


### 秘密在實踐上是公開的

`ts_authkey` 位於包/ISO 內，**任何人都可以提取它**。和他一起，有人
您可以將機器放置在尾網上，但是：

- 始終帶有標籤 `tag:suporte`，蜉蝣；
- **沒有看到任何東西**，感謝上述政策。

在最壞的情況下，`-l` 中會出現奇怪的機器。

> 切勿在具有預設策略的尾網上使用秘密（所有內容均已發布）：那裡，機器
> 支援人員可以存取一切內容。

### 切勿在 GitHub 上發布秘密

Tailscale 參與 GitHub 秘密掃描：真實的 `tskey-client-...` 合而為一
公共儲存庫往往會被**自動撤銷**。在儲存庫中，保留
範例值；真正的秘密必須放在 ISO 版本或安裝後。

### 如果秘密外洩或濫用

1. **設定 → 信任憑證** → 關閉憑證 `voidbr-suporte`
2. **機器** → 按 `tag:suporte` 過濾並刪除您不認識的內容
3. 產生新憑證並更新`.conf`

---

## 

沒有 tailnet 的替代方案，使用 [upterm](https://github.com/owenthereal/upterm) 公共伺服器：

```bash
voidbr-suporte -u
```

- 顯示`ssh ...@uptermd.upterm.dev`指令和QR碼（帶有`qrencode`）
- 僅包含技術人員的鑰匙 (`tecnicos_github` / `tecnicos_chaves`)
- 使用者需要將命令傳遞給技術人員（二維碼、ntfy 或聽寫）
- 需要安裝 `upterm` 二進位（不在 Void 儲存庫中）

---

## 常見問題

**`backend error: key tagged with non-existent tag: tag:suporte`**
秘密tailnet策略不再具有`tag:suporte`（`tagOwners`區塊）。
常見原因：在錯誤的尾網上更改了策略。檢查選定的尾網
控制台並替換上述策略。

**`-l`不顯示任何機器，但遠端連接**
技術人員的 `tailscaled` 服務位於另一個尾網上。檢查與
`tailscale switch --list` 與 `sudo tailscale switch <ID>` 連接。

**`ERRO: ts_authkey não definido` 非技術**
如果沒有選項，`voidbr-suporte` 啟動 **使用者** 端。在技術人員中使用 `-l` 和 `-c`。

**`can't switch user` / `não consegui entrar como '...' (código 255)`**
登入被拒絕，因為使用者不是在遠端執行腳本的使用者。
常見原因：遙控器有一個**舊版**的`voidbr-suporte`（沒有使用者的名稱，
`suporte-<host>-xxxx`)，`-c` 以使用者身分讀取主機名稱的開頭。
更新遙控器或輸入使用者：`voidbr-suporte -c <ip> <usuario>`。

**`entrou, mas a sessão 'suporte' do tmux não abriu` / `no sessions`**
使用者的tmux還沒打開（使用`auto=0`，他需要按回車）。

**`ssh: Could not resolve hostname suporte-...`**
MagicDNS 未啟用。使用 `voidbr-suporte -c`，它將 IP 解析為 tailnet 本身。

*
使用`voidbr-suporte -c`（它使用`tmux -u`強制使用UTF-8），而不是直接使用`ssh`。

*



*


*


*



---

## 


