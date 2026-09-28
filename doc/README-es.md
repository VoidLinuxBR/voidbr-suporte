<div align="centro">

# 🔵 soporte voidbr

**Soporte remoto rápido estilo tmate (Tailscale SSH + tmux compartido o upterm)**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge)](LICENCIA)

</div>

---

Soporte remoto rápido para **VoidBR Linux**, en el espíritu del antiguo `tmate`:
el usuario escribe **un comando** (incluso en tty, sin entorno gráfico) y el técnico
ingrese al **mismo terminal**, viendo y escribiendo juntos.

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

- Sin cuenta de usuario, inicio de sesión, contraseña o clave SSH
- Nadie necesita dictar fichas: la máquina se presenta sola al técnico
- Nada permanece instalado o ejecutándose después del servicio.
- Mensajes traducibles con gettext (pt_BR y en)
- Modo alternativo vía **upterm**, para cuando no hay tailnet

---

## Índice

- CHILE_REF_0_CHILI
- CHILE_REF_0_CHILI
- CHILE_REF_0_CHILI
- CHILE_REF_0_CHILI
- CHILE_REF_0_CHILI
- CHILE_REF_0_CHILI
- CHILE_REF_0_CHILI
- CHILE_REF_0_CHILI
- CHILE_REF_0_CHILI

---

## como funciona

### Lado del usuario

Para rodar `voidbr-suporte`, el script:

1. Cree una carpeta temporal y cargue un **`tailscaled` solo para el servicio**
(modo espacio de usuario, estado solo en memoria, socket y puerto propios).
2. Ingrese a la red de cola del técnico usando el secreto `/etc/voidbr-suporte.conf`,
como un nodo **efímero** con la etiqueta `tag:suporte` y el nombre `suporte-<usuario>-<hostname>-<xxxx>`.
El usuario introduce el nombre para que el técnico lo introduzca sin tener que informarle.
3. Active **Tailscale SSH**: el propio tailscaled sirve SSH, sin sshd.
4. Abre el shell del usuario dentro de **tmux** con una barra de color que muestra
usuario, nombre de la máquina, IP y `exit = encerrar`.
5. Al salir (`exit` o Ctrl+C): haga `tailscale logout`, la máquina **sale de la red trasera inmediatamente**,
el demonio finaliza y la carpeta temporal se elimina.

### Por qué no es necesario ejecutar el servicio `tailscaled` en el control remoto

El servicio runit solo se usa para activar `tailscaled` en el arranque. `voidbr-suporte` se enciende
tu propio `tailscaled`, sólo durante el servicio:

| | Servicio del sistema | El demonio `voidbr-suporte` |

| Zócalo | `/run/tailscale/tailscaled.sock` | CHILE_REF_1_CHILI |

| Puerto UDP | 41641 | cualquiera gratis (`--port=0`) |





- 
- 
- 
- 

- 

### 

```bash
voidbr-suporte -l      # máquinas aguardando suporte
voidbr-suporte -c      # conecta no tmux do usuário
```




### 

```
 SUPORTE  vcatafesta   0:bash         suporte-voidbr-liteon-nwhf  100.89.95.104  exit = encerrar
 vermelho azul         janelas        roxo                        verde          amarelo
```



---

## Instalación

### 

```bash
git clone https://github.com/voidlinuxbr/voidbr-suporte.git
cd voidbr-suporte
pkgmake -s -i        # instala as dependências, gera o .xbps e instala
```

### 

```bash
sudo xbps-install -S tailscale tmux curl qrencode gettext
sudo install -Dm755 usr/bin/voidbr-suporte   /usr/bin/voidbr-suporte
sudo install -Dm644 etc/voidbr-suporte.conf  /etc/voidbr-suporte.conf
```

### 















---

## 










### 







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











### 

1. 
2. 
3. 
4. 
5. 
6. 



### 



```bash
ts_authkey="tskey-client-SEU_SECRET?ephemeral=true&preauthorized=true"
```

- 
- 
- 

### 

```bash
sudo xbps-install -S tailscale tmux
sudo ln -s /etc/sv/tailscaled /var/service/
sudo tailscale up        # conta admin da tailnet do suporte
```



```bash
tailscale switch --list          # o * marca a ativa
sudo tailscale switch <ID>       # troca de tailnet
```

---

## Usar

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

### Usuario

```bash
voidbr-suporte
```







### 

```bash
voidbr-suporte -l                          # quem está aguardando suporte
voidbr-suporte -c                          # se houver só uma máquina online
voidbr-suporte -c 100.95.244.100           # pelo IP
voidbr-suporte -c liteon                   # por parte do nome
voidbr-suporte -c 100.95.244.100 fulano    # forçando outro usuário
voidbr-suporte -n                          # aguarda chamados via ntfy
```



```
IP               USUÁRIO      NOME                                     ESTADO
100.95.244.100   vcatafesta   suporte-vcatafesta-voidbr-liteon-xe03    -
100.95.1.7       root         suporte-root-voidbr-live-ab12            active; direct
```










- 
- 
- 
- 




---

## Ajustes



















---

## Idiomas


*

- 
- 
- 



```bash
LANGUAGE=en voidbr-suporte -h
```




---

## 

- 
- 
- 

- 

- 


### 




- 
- 






### 





### 

1. 
2. 
3. 

---

## 



```bash
voidbr-suporte -u
```

- 
- 
- 
- 

---

## 

*




*



*


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


