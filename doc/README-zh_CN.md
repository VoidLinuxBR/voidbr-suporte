<div对齐=“中心”>

# 🔵 voidbr-支持

**快速 tmate 式远程支持（Tailscale SSH + 共享 tmux 或 upterm）**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge)]（许可证）

</div>

---

本着旧 `tmate` 的精神，对 **VoidBR Linux** 进行快速远程支持：
用户键入 *命令**（包括在 tty 中，没有图形环境），技术人员
进入**同一终端**，一起查看和输入。

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

- 无用户端帐户、登录名、密码或 SSH 密钥
- 没有人需要口授令牌：机器对技术人员来说是单独出现的
- 维修后没有任何东西保持安装或运行
- 可使用 gettext 翻译的消息（pt_BR 和 en）
- 通过 **upterm** 的替代模式，适用于没有尾网的情况

---

## 指数

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

## 它是如何运作的

### 用户侧

奥罗德`voidbr-suporte`，奥脚本：

1. 创建一个临时文件夹并上传 **`tailscaled` 仅用于服务**
（用户空间模式，状态仅在内存中，拥有自己的套接字和端口）。
2. 使用`/etc/voidbr-suporte.conf`秘密进入技术人员的尾网，
作为具有标签 `tag:suporte` 和名称 `suporte-<usuario>-<hostname>-<xxxx>` 的 **临时** 节点。
用户输入姓名供技术人员输入，而无需通知他。
3. 打开 **Tailscale SSH**：tailscaled 本身提供 SSH，无需 sshd。
4. 在 **tmux** 内打开用户的 shell，并显示彩色条
用户、机器名、IP 和 `exit = encerrar`。
5. 退出时（`exit`或Ctrl+C）：执行`tailscale logout`，机器**立即退出尾网**，
守护进程被终止，临时文件夹被删除。

### 为什么`tailscaled`服务不需要在远程运行

runit 服务仅用于在启动时打开 `tailscaled`。 `voidbr-suporte` 开启
您自己的 `tailscaled`，仅在服务期间：

| |系统服务| `voidbr-suporte` 守护进程 |
|---|---|---|
|插座|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 |
|状态 | `/var/lib/tailscale/`（磁盘）|只存在记忆中|
| UDP端口| 41641 | 41641任何免费的 (`--port=0`) |
|网络| `tailscale0` 接口和路线 |用户空间，没有接口或路由|
|持续时间 |总是|仅在支持期间|

这就是为什么：

- **仅在用户需要帮助时保持连接**；技术人员无法在此之外访问机器。
- **无需root**：普通用户可以寻求支持。
- **不留下任何痕迹**：没有任何内容写入磁盘。
- **与用户的 Tailscale 共存**：如果他已经在其 tailnet 上使用 Tailscale，则两者
它们并排运行，没有混合。
- **适用于 ISO live**，无需启用任何服务。

### 教练侧

```bash
voidbr-suporte -l      # máquinas aguardando suporte
voidbr-suporte -c      # conecta no tmux do usuário
```

`-c` 通过尾网本身发现 IP 和用户（无需 MagicDNS 即可工作）
并始终通过 IP 直接连接到共享 tmux。

### 不要出去

```
 SUPORTE  vcatafesta   0:bash         suporte-voidbr-liteon-nwhf  100.89.95.104  exit = encerrar
 vermelho azul         janelas        roxo                        verde          amarelo
```

东京之夜调色板中的颜色。在 tty 中，tmux 会适应控制台颜色。

---

## 安装

### 通过 pkgmake

```bash
git clone https://github.com/voidlinuxbr/voidbr-suporte.git
cd voidbr-suporte
pkgmake -s -i        # instala as dependências, gera o .xbps e instala
```

### 手动的

```bash
sudo xbps-install -S tailscale tmux curl qrencode gettext
sudo install -Dm755 usr/bin/voidbr-suporte   /usr/bin/voidbr-suporte
sudo install -Dm644 etc/voidbr-suporte.conf  /etc/voidbr-suporte.conf
```

### 依赖关系

|套餐 |用途 |
|---|---|
|辣椒_REF_0_辣椒 |尾秤模式（用户和技术人员）|
|辣椒_REF_0_辣椒 |共享终端|
|辣椒_REF_0_辣椒 |翻译后的消息（没有它，所有内容都会在 pt_BR 中显示）|
|辣椒_REF_0_辣椒 |通过 ntfy 向技术人员发出通知（可选）|
|辣椒_REF_0_辣椒 | QR 码 no modo upterm（可选）|
|辣椒_REF_0_辣椒 |仅适用于 upterm 模式（不在 Void 存储库中）|

|机器|套餐 `tailscale` | `tailscaled` 服务 |尾网登录 |
|---|---|---|---|
| **技术** |是的 | **是的，始终运行** |是的，`tailscale up` 具有管理员帐户 |
| **用户** |是的 | **不需要** |否：脚本与秘密一起单独输入 |

---

## 尾网准备（仅一次）

Tailscale 管理控制台中的所有内容：<https://login.tailscale.com/admin>

> **推荐：**使用**尾网专用**来支持（例如组织的），
> 与您的个人尾网分开。邀请您的个人帐户作为**管理员**。

> **注意：**如果您的账户参与多个尾网，**请查看顶部的
> 保存策略或创建凭据之前选择的控制台**。
> 更改历史记录位于 **日志 → 配置**。

### 1. 准入政策

在**访问控制**中，在 JSON 编辑器中。 **请先备份您当前的政策。**

默认策略（`"src": ["*"], "dst": ["*"]`）向所有人发布所有内容，包括
标记的机器。有了它，支持机器就可以看到整个网络。
以下政策让**人**免费访问，并隔离支持：

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

|规则|效果|
|---|---|
|辣椒_REF_0_辣椒 |创建 `tag:suporte` 标签（没有它，支持将无法连接） |
|授予|成员和访客访问一切：机器、子网、出口节点 |
| `ssh` 接受 | Tailscale SSH 无需浏览器确认即可接受管理员管理 |
|没有规则离开`tag:suporte` |支持机器什么也看不见|

> 要使 SSH 工作，您需要**两件事**：对端口 22 的网络访问
>（由拨款覆盖）**和** `ssh` 中的规则。

### 2. OAuth 客户端（“邀请”）

1. **设置 → 信任凭证 → 凭证 → OAuth**
2. 范围：**仅限身份验证密钥**，带有**写入**
3. 标签：**仅 `tag:suporte`**（仅在保存策略后出现）
4. 说明：`voidbr-suporte`
5. **生成凭证**
6. 复制 **客户端密钥** (`tskey-client-...`)：它 **不会再次出现**

使用**唯一**凭证来获得支持；不要重复使用 CI 或其他 CI。

### 3. 将秘密放入`.conf`

在用户计算机上（或在 ISO 中），在 `/etc/voidbr-suporte.conf` 中：

```bash
ts_authkey="tskey-client-SEU_SECRET?ephemeral=true&preauthorized=true"
```

- 由于 `?` 和 `&`，因此需要引号
- 未使用客户端 ID
- 技术人员**不需要**需要秘密

### 4. 教练方

```bash
sudo xbps-install -S tailscale tmux
sudo ln -s /etc/sv/tailscaled /var/service/
sudo tailscale up        # conta admin da tailnet do suporte
```

如果您的帐户位于多个 tailnet 上，则一次只有一个在该服务上处于活动状态：

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

### 用户

```bash
voidbr-suporte
```

使用`auto=1`（软件包默认值），带有名称和IP的帧会出现2秒，然后
tmux 自行打开。关闭：`exit`。

用户可以是任何人：实时 ISO 中的 `root`、`maria`、`anon`... 机器名称
与运行该脚本的人一起退出，例如 `suporte-anon-notebook-a1b2`。

### 技术的

```bash
voidbr-suporte -l                          # quem está aguardando suporte
voidbr-suporte -c                          # se houver só uma máquina online
voidbr-suporte -c 100.95.244.100           # pelo IP
voidbr-suporte -c liteon                   # por parte do nome
voidbr-suporte -c 100.95.244.100 fulano    # forçando outro usuário
voidbr-suporte -n                          # aguarda chamados via ntfy
```

`-l` 显示每台机器的用户：

```
IP               USUÁRIO      NOME                                     ESTADO
100.95.244.100   vcatafesta   suporte-vcatafesta-voidbr-liteon-xe03    -
100.95.1.7       root         suporte-root-voidbr-live-ab12            active; direct
```

`-c` **通过计算机名称发现用户**。用户空间模式下的 Tailscale SSH
它只允许您以**在远程运行脚本的同一用户身份**输入，这就是名称中的内容：

|谁在远程运行|机器名称 | `-c` 使用 |
|---|---|---|
| `root`（ISO 实时）|辣椒_REF_1_辣椒 |辣椒_REF_2_辣椒 |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 |辣椒_REF_2_辣椒 |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 |辣椒_REF_2_辣椒 |

- 技术人员**不需要在远程拥有帐户**：tailnet 授权管理员以该用户身份登录，无需密码。
- 现场技术人员的用户并不重要。
- 会话内，技术人员拥有用户的权限；对于 root，`sudo`（用户在共享终端上输入密码）。
- 字符位于 `a-z0-9` 之外的用户（例如：`joao.silva`）的名称将被简化（`joaosilva`）；
在本例中，输入用户：`voidbr-suporte -c <ip> joao.silva`。

当有多台机器在线并且没有参数时，`-c` 显示列表并询问名称或 IP。

---

## 设置

文件：`/etc/voidbr-suporte.conf`（在包更新中保留）。

|选项|标准|描述 |
|---|---|---|
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 | `tailscale` 或 `upterm` |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 | `1` = 同一终端 (tmux)； `0` = 技术人员打开自己的终端 |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 | tmux 会话名称 |
|辣椒_REF_0_辣椒 | `0`（`1` 无包）| `1` = abre o tmux direto，sem pedir Enter |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 | `1` = 栏中的分隔符（需要 Nerd 字体；不会出现在 tty 中）|
|辣椒_REF_0_辣椒 | — | OAuth 客户端 com `?ephemeral=true&preauthorized=true` | segredo
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 |标签应用于机器|
|辣椒_REF_0_辣椒 | — | (upterm) 授权 GitHub 用户 |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 | (upterm) 查韦斯演员 |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 | (upterm) 服务商 |
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 | (upterm) `1` = 用户批准每个连接 |
|辣椒_REF_0_辣椒 | *（空）* | ntfy.sh主题通知技术人员；空关掉|
|辣椒_REF_0_辣椒 |辣椒_REF_1_辣椒 |服务器 ntfy |

---

## 语言

消息使用 **gettext**（域 `voidbr-suporte`）。原文位于
**pt_BR**：如果没有安装目录，或者没有 `gettext` 命令，所有内容都会以葡萄牙语显示。

- 目录：`/usr/share/locale/<idioma>/LC_MESSAGES/voidbr-suporte.mo`
- 存储库中的翻译：`po/<idioma>.po`
- 脚本中的翻译函数为`_`（提取关键字：`-k_`）

要查看英文版：

```bash
LANGUAGE=en voidbr-suporte -h
```

> 当变量为空时，脚本头设置 `LANGUAGE=pt_BR`。这就是为什么，
> 在具有 `LANG=en_US.UTF-8` 的系统上，仅在定义了 `LANGUAGE=en` 时才显示英语。

---

## 安全

- **没有访问令牌。**只有**尾网管理员**才能进入。
- 连接是 **WireGuard 端到端**； Tailscale 服务器仅将一侧呈现给另一侧。
- 支持机器是**短暂的**：它们将尾网留在`exit`并且不留下任何密钥，服务
或向后配置。
- `-c`不会在`known_hosts`中记录主机密钥，因为每个服务都会生成一个新机器
并且IP可以重复；身份已经由 tailnet 保证。
- 支持机器**在尾网上没有实现任何目标**（已测试：从远程连接到
技术人员未通过）。

### 秘密在实践中是公开的

`ts_authkey` 位于包/ISO 内，**任何人都可以提取它**。和他一起，有人
您可以将机器放置在尾网上，但是：

- 始终带有标签 `tag:suporte`，蜉蝣；
- **没有看到任何东西**，感谢上述政策。

在最坏的情况下，`-l` 中会出现奇怪的机器。

> 切勿在具有默认策略的尾网上使用秘密（所有内容均已发布）：那里，机器
> 支持人员可以访问一切内容。

### 切勿在 GitHub 上发布秘密

Tailscale 参与 GitHub 秘密扫描：真实的 `tskey-client-...` 合二为一
公共存储库往往会被**自动撤销**。在存储库中，保留
示例值；真正的秘密必须放在 ISO 版本中或安装后。

### 如果秘密被泄露或滥用

1. **设置 → 信任凭证** → 关闭凭证 `voidbr-suporte`
2. **机器** → 按 `tag:suporte` 过滤并删除您不认识的内容
3. 生成新凭证并更新`.conf`

---

## 上行模式

没有 tailnet 的替代方案，使用 [upterm](https://github.com/owenthereal/upterm) 公共服务器：

```bash
voidbr-suporte -u
```

- 显示`ssh ...@uptermd.upterm.dev`命令和QR码（带有`qrencode`）
- 仅包含技术人员的钥匙 (`tecnicos_github` / `tecnicos_chaves`)
- 用户需要将命令传递给技术人员（二维码、ntfy 或听写）
- 需要安装 `upterm` 二进制文件（不在 Void 存储库中）

---

## 常见问题

**`backend error: key tagged with non-existent tag: tag:suporte`**
秘密tailnet策略不再具有`tag:suporte`（`tagOwners`块）。
常见原因：在错误的尾网上更改了策略。 Check out the selected tailnet on
控制台并替换上述策略。

**`-l`不显示任何机器，但远程连接**
技术人员的 `tailscaled` 服务位于另一个尾网上。检查与
`tailscale switch --list` 与 `sudo tailscale switch <ID>` 相连。

**`ERRO: ts_authkey não definido` 非技术**
如果没有选项，`voidbr-suporte` 启动 **用户** 端。在技术人员中使用 `-l` 和 `-c`。

**`can't switch user` / `não consegui entrar como '...' (código 255)`**
登录被拒绝，因为用户不是在远程运行脚本的用户。
常见原因：遥控器有一个**旧版本**的`voidbr-suporte`（没有用户的名称，
`suporte-<host>-xxxx`)，`-c` 以用户身份读取主机名的开头。
更新遥控器或输入用户：`voidbr-suporte -c <ip> <usuario>`。

**`entrou, mas a sessão 'suporte' do tmux não abriu` / `no sessions`**
用户的tmux还没有打开（使用`auto=0`，他需要按回车）。

**`ssh: Could not resolve hostname suporte-...`**
MagicDNS 未激活。使用 `voidbr-suporte -c`，它将 IP 解析为 tailnet 本身。

**重音显示为`_`**
使用`voidbr-suporte -c`（它使用`tmux -u`强制使用UTF-8），而不是直接使用`ssh`。

**尾网客人失去访问权限**

（获得共享机器的人不是会员）。

*


*


*



---

## 


