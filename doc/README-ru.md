<div align="center">

# 🔵 поддержка voidbr

**Быстрая удаленная поддержка в стиле tmate (Tailscale SSH + общий tmux или upterm)**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge)](ЛИЦЕНЗИЯ)

</div>

---

Быстрая удаленная поддержка **VoidBR Linux** в духе старого `tmate`:
пользователь вводит **команду** (в том числе в tty, без графической среды), а техник
войдите в **тот же терминал**, одновременно просматривая и печатая.

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

- Нет учетной записи пользователя, логина, пароля или ключа SSH.
- Никому не нужно диктовать жетоны: автомат предстает перед техническим специалистом один
- Ничто не остается установленным или работающим после обслуживания.
- Переводимые сообщения с помощью gettext (pt_BR и en)
- Альтернативный режим через **upterm**, когда нет хвостовой сети.

---

## Индекс

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

## Как это работает

### Пользовательская сторона

В списке `voidbr-suporte`, в скрипте:

1. Создайте временную папку и загрузите **`tailscaled` только для службы**
(режим пользовательского пространства, состояние только в памяти, собственный сокет и порт).
2. Войдите в сеть технического специалиста, используя секрет `/etc/voidbr-suporte.conf`,
как **эфемерный** узел с тегом `tag:suporte` и именем `suporte-<usuario>-<hostname>-<xxxx>`.
Пользователь вводит имя технического специалиста, не сообщая ему об этом.
3. Включите **Tailscale SSH**: Tailscaled сам обслуживает SSH без sshd.
4. Открывает оболочку пользователя внутри **tmux** с цветной полосой, показывающей
пользователь, имя машины, IP и `exit = encerrar`.
5. При выходе (`exit` или Ctrl+C): выполните `tailscale logout`, машина **немедленно выйдет из хвостовой сети**,
демон завершается и временная папка удаляется.

### Почему службу `tailscaled` не обязательно запускать на удаленном компьютере

Служба runit используется только для включения `tailscaled` при загрузке. `voidbr-suporte` включается
свой `tailscaled`, только во время службы:

| | Системный сервис | Демон `voidbr-suporte` |
|---|---|---|
| Розетка | `/run/tailscale/tailscaled.sock` | `/tmp/voidbr-suporte.XXXX/tailscaled.sock` |
| Статус | `/var/lib/tailscale/` (диск) | только в памяти |
| UDP-порт | 41641 | любой бесплатный (`--port=0`) |
| Сеть | `tailscale0` интерфейс и маршруты | пользовательское пространство, без интерфейса и маршрутов |
| Продолжительность | всегда | только во время поддержки |

Вот почему:

- **Остается на связи только тогда, когда пользователю нужна помощь**; технический специалист не имеет доступа к машине за пределами этого места.
- **Нет необходимости в root-доступе**: обычный пользователь может обратиться за поддержкой.
- **Не оставляет следов**: ничего не попадает на диск.
- **Сосуществует с Tailscale пользователя**: если он уже использует Tailscale в своей хвостовой сети, оба
они бегут бок о бок, не смешиваясь.
- **Работает в формате ISO Live** без включения каких-либо служб.

### Тренерская сторона

```bash
voidbr-suporte -l      # máquinas aguardando suporte
voidbr-suporte -c      # conecta no tmux do usuário
```

`-c` определяет IP и пользователя через хвостовую сеть (работает без MagicDNS)
и подключается напрямую к общему tmux, всегда через IP.

### Не выходи на улицу

```
 SUPORTE  vcatafesta   0:bash         suporte-voidbr-liteon-nwhf  100.89.95.104  exit = encerrar
 vermelho azul         janelas        roxo                        verde          amarelo
```

Цвета палитры «Токио Ночь». В tty tmux адаптируется к цветам консоли.

---

## Установка

### Автор: pkgmake

```bash
git clone https://github.com/voidlinuxbr/voidbr-suporte.git
cd voidbr-suporte
pkgmake -s -i        # instala as dependências, gera o .xbps e instala
```

### Руководство

```bash
sudo xbps-install -S tailscale tmux curl qrencode gettext
sudo install -Dm755 usr/bin/voidbr-suporte   /usr/bin/voidbr-suporte
sudo install -Dm644 etc/voidbr-suporte.conf  /etc/voidbr-suporte.conf
```

### Зависимости

| Пакет | Использование |
|---|---|
| `tailscale` | режим Tailscale (пользователь и техник) |
| `tmux` | общий терминал |
| `gettext` | переведенные сообщения (без него все выходит в pt_BR) |
| `curl` | уведомление техническому специалисту через ntfy (необязательно) |
| `qrencode` | QR-код без режима upterm (опционально) |
| `upterm` | только для режима upterm (нет в репозитории Void) |

| Машина | Пакет `tailscale` | `tailscaled` Сервис | Вход в Tailnet |
|---|---|---|---|
| **Технический** | да | **да, всегда работает** | да, `tailscale up` с учетной записью администратора |
| **Пользователь** | да | **нет необходимости** | нет: скрипт входит один с секретом |

---

## Подготовка хвостовой сети (только один раз)

Все в консоли администратора Tailscale: <https://login.tailscale.com/admin>.

> **Рекомендуется** использовать **выделенную** сеть хвостовой части** для поддержки (например, организации),
> отдельно от вашей личной хвостовой сети. Пригласите в него свой личный аккаунт в качестве **Администратора**.

> **Внимание:** если ваша учетная запись участвует в более чем одной хвостовой сети, **отметьте это в верхней части
> консоль, которая выбрана**, прежде чем сохранять политику или создавать учетные данные.
> История изменений находится в **Журналы → Конфигурация**.

### 1. Политика доступа

В разделе **Управление доступом** в редакторе JSON. **Сначала создайте резервную копию текущей политики.**

Политика по умолчанию (`"src": ["*"], "dst": ["*"]`) предоставляет доступ всем всем, включая
отмеченные машины. С его помощью машина поддержки будет видеть всю сеть.
Приведенная ниже политика обеспечивает бесплатный доступ для **людей** и оставляет поддержку изолированной:

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

| Правило | Эффект |
|---|---|
| `tagOwners` | создает тег `tag:suporte` (без него поддержка не подключится) |
| грант | участники и гости имеют доступ ко всему: машинам, подсетям, выходным узлам |
| `ssh` принять | Tailscale SSH принимает администратора, не запрашивая подтверждение браузера |
| никаких правил, выход из `tag:suporte` | машина поддержки ничего не видит |

> Для работы SSH вам нужны **обе вещи**: сетевой доступ к порту 22.
> (покрывается грантом) **и** правило в `ssh`.

### 2. Клиент OAuth («приглашение»)

1. **Настройки → Доверительные учетные данные → Учетные данные → OAuth**
2. Область применения: **Только ключи аутентификации** с возможностью **Записи**.
3. Тег: **только `tag:suporte`** (появляется только после сохранения политики)
4. Описание: `voidbr-suporte`
5. **Сгенерировать учетные данные**
6. Скопируйте **секрет клиента** (`tskey-client-...`): он **больше не появляется**

Используйте **уникальные** учетные данные для поддержки; не используйте повторно CI или другой.

### 3. Поместите секрет в `.conf`.

На пользовательском компьютере (или в ISO) в `/etc/voidbr-suporte.conf`:

```bash
ts_authkey="tskey-client-SEU_SECRET?ephemeral=true&preauthorized=true"
```

- Кавычки необходимы, поскольку `?` и `&`
- Идентификатор клиента не используется
- Технику **не** нужен секрет

### 4. Тренерская сторона

```bash
sudo xbps-install -S tailscale tmux
sudo ln -s /etc/sv/tailscaled /var/service/
sudo tailscale up        # conta admin da tailnet do suporte
```

Если ваша учетная запись находится в более чем одной хвостовой сети, в службе одновременно активна только одна:

```bash
tailscale switch --list          # o * marca a ativa
sudo tailscale switch <ID>       # troca de tailnet
```

---

## Использовать

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

### Пользователь

```bash
voidbr-suporte
```

При `auto=1` (пакет по умолчанию) на 2 секунды появляется рамка с именем и IP, а затем
tmux открывается сам. Чтобы закрыть: `exit`.

Пользователем может быть кто угодно: `root` в активном ISO, `maria`, `anon`… Имя машины.
завершается с тем, кто запустил сценарий, например `suporte-anon-notebook-a1b2`.

### Технический

```bash
voidbr-suporte -l                          # quem está aguardando suporte
voidbr-suporte -c                          # se houver só uma máquina online
voidbr-suporte -c 100.95.244.100           # pelo IP
voidbr-suporte -c liteon                   # por parte do nome
voidbr-suporte -c 100.95.244.100 fulano    # forçando outro usuário
voidbr-suporte -n                          # aguarda chamados via ntfy
```

`-l` показывает пользователя каждой машины:

```
IP               USUÁRIO      NOME                                     ESTADO
100.95.244.100   vcatafesta   suporte-vcatafesta-voidbr-liteon-xe03    -
100.95.1.7       root         suporte-root-voidbr-live-ab12            active; direct
```

`-c` **обнаруживает пользователя по имени компьютера**. Tailscale SSH в режиме пользовательского пространства
Он позволяет вам войти только как **тот же пользователь, который запустил скрипт** на удаленном компьютере, и это то, что указано в имени:

| Кто работает удаленно | Имя машины | `-c` использует |
|---|---|---|
| `root` (ISO в реальном времени) | `suporte-root-voidbr-live-x9z8` | `root` |
| `vcatafesta` | `suporte-vcatafesta-voidbr-liteon-xe03` | `vcatafesta` |
| `anon` | `suporte-anon-notebook-a1b2` | `anon` |

- Техническому специалисту **не обязательно иметь учетную запись** на удаленном компьютере: Tailnet разрешает администратору войти в систему под этим пользователем без пароля.
- Пользователь технического специалиста на месте не имеет значения.
- В рамках сеанса техник имеет права пользователя; для root — `sudo` (пользователь вводит пароль на общем терминале).
- Имя пользователей с символами, отличными от `a-z0-9` (например: `joao.silva`), будет упрощено (`joaosilva`);
В этом случае введите пользователя: `voidbr-suporte -c <ip> joao.silva`.

Если в сети находится более одной машины и без аргументов, `-c` показывает список и запрашивает имя или IP-адрес.

---

## Настройки

Файл: `/etc/voidbr-suporte.conf` (сохраняется при обновлении пакета).

| Вариант | Стандарт | Описание |
|---|---|---|
| `modo` | `tailscale` | `tailscale` или `upterm` |
| `compartilhar` | `1` | `1` = тот же терминал (tmux); `0` = техник открывает собственный терминал |
| `tmux_sessao` | `suporte` | имя сеанса tmux |
| `auto` | `0` (`1` без пакета) | `1` = перейти непосредственно к tmux, затем ввести Enter |
| `tmux_powerline` | `0` | `1` = разделители на панели (требуется шрифт Nerd; не отображается в tty) |
| `ts_authkey` | — | изолировано от клиента OAuth с `?ephemeral=true&preauthorized=true` |
| `ts_tag` | `tag:suporte` | бирка, прикрепленная к машине |
| `tecnicos_github` | — | (upterm) авторизованные пользователи GitHub |
| `tecnicos_chaves` | `/etc/voidbr-suporte/authorized_keys` | (досрочно) дополнительные услуги |
| `servidor` | `ssh://uptermd.upterm.dev:22` | (срочный) сервидор |
| `perguntar` | `0` | (upterm) `1` = пользователь одобряет каждое соединение |
| `ntfy_topico` | *(пусто)* | тема ntfy.sh для уведомления технического специалиста; пустой выключить |
| `ntfy_url` | `https://ntfy.sh` | сервер ntfy |

---

## Языки

В сообщениях используется **gettext** (домен `voidbr-suporte`). Оригинальные тексты находятся в
**pt_BR**: без установленного каталога или без команды `gettext` все отображается на португальском языке.

- Каталоги: `/usr/share/locale/<idioma>/LC_MESSAGES/voidbr-suporte.mo`
- Переводы в репозитории: `po/<idioma>.po`
- Функция перевода в скрипте — `_` (ключевое слово для извлечения: `-k_`)

Чтобы увидеть на английском языке:

```bash
LANGUAGE=en voidbr-suporte -h
```

> Заголовок скрипта устанавливает `LANGUAGE=pt_BR`, когда переменная пуста. Вот почему,
> в системе с `LANG=en_US.UTF-8` английский появляется только с определенным `LANGUAGE=en`.

---

## Безопасность

- **Токен доступа отсутствует.** Входить могут только те, кто является **администратором хвостовой сети**.
- Соединение **сквозное** WireGuard**; Сервер Tailscale представляет только одну сторону другой.
- Машины поддержки **эфемерны**: они покидают хвостовую сеть по адресу `exit` и не оставляют ключа, обслуживания.
или обратная конфигурация.
- `-c` не записывает ключ хоста в `known_hosts`, поскольку каждая служба создает новую машину.
и IP может повторяться; идентичность уже гарантирована Tailnet.
- Машины поддержки **ничего не достигают** в хвостовой сети (проверено: подключение от удаленного к
техник не проходит).

### На практике секрет открыт

`ts_authkey` находится внутри пакета/ISO, и **любой может его извлечь**. С ним кто-то
Вы можете размещать машины в своей хвостовой сети, но:

- всегда с тегом `tag:suporte`, эфемерный;
- **ничего не видя** благодаря указанным выше правилам.

В худшем случае в `-l` появятся странные машины.

> Никогда не используйте секрет в хвостовой сети с политикой по умолчанию (все выпущено): вот машина
> Поддержка имела бы доступ ко всему.

### Никогда не публикуйте секрет на GitHub.

Tailscale участвует в сканировании секретов GitHub: настоящий `tskey-client-...` в одном
публичный репозиторий имеет тенденцию быть **автоматически отозван**. В репозитории сохраните
пример значения; настоящий секрет должен быть помещен в сборку ISO или после установки.

### Если секрет раскрыт или злоупотреблен

1. **Настройки → Доверенные учетные данные** → отобразить учетные данные `voidbr-suporte`
2. **Машины** → отфильтруйте по `tag:suporte` и удалите все, что вы не узнаете
3. Создайте новые учетные данные и обновите `.conf`.

---

## Режим работы в режиме ожидания

Альтернатива без хвостовой сети с использованием общедоступного сервера [upterm](https://github.com/owenthereal/upterm):

```bash
voidbr-suporte -u
```

- Показывает команду `ssh ...@uptermd.upterm.dev` и QR-код (с `qrencode`)
- Включены только ключи технического специалиста (`tecnicos_github` / `tecnicos_chaves`).
- Пользователю необходимо передать команду технику (QR-код, ntfy или под диктовку)
- Требуется установленный двоичный файл `upterm` (нет в репозитории Void).

---

## Распространенные проблемы

**`backend error: key tagged with non-existent tag: tag:suporte`**
Секретная политика хвостовой сети больше не имеет `tag:suporte` (блок `tagOwners`).
Общая причина: политика была изменена не в той хвостовой сети. Проверьте выбранную хвостовую сеть на
console и замените вышеуказанную политику.

**`-l` не показывает ни одного компьютера, но подключен удаленный компьютер**
Служба `tailscaled` технического специалиста находится в другой хвостовой сети. Проверьте с
`tailscale switch --list` и тройка с `sudo tailscale switch <ID>`.

**`ERRO: ts_authkey não definido` нетехнический**
Без опций `voidbr-suporte` запускает сторону **пользователя**. В технике используйте `-l` и `-c`.

**`can't switch user` / `não consegui entrar como '...' (código 255)`**
Во входе было отказано, поскольку пользователь не запускал сценарий на удаленном компьютере.
Распространенная причина: на пульте дистанционного управления установлена **старая версия** `voidbr-suporte` (имя без имени пользователя,
`suporte-<host>-xxxx`), а `-c` читает начало имени хоста как пользователя.
Обновите пульт или введите пользователя: `voidbr-suporte -c <ip> <usuario>`.

**`entrou, mas a sessão 'suporte' do tmux não abriu` / `no sessions`**
tmux пользователя еще не открылся (при `auto=0` ему нужно нажать Enter).

**`ssh: Could not resolve hostname suporte-...`**
MagicDNS не активен. Используйте `voidbr-suporte -c`, который разрешает IP-адрес самой хвостовой сети.

*
Используйте `voidbr-suporte -c` (он принудительно использует UTF-8 с `tmux -u`) вместо прямого `ssh`.

**Гости Tailnet потеряли доступ**



*


*


*



---

## 


