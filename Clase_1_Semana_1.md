# Clase 1 · Semana 1: Fundamentos de Nmap

> **Objetivo:** dominar los 5 escaneos básicos de Nmap y saber leer sus resultados.
> **Objetivo de práctica:** `scanme.nmap.org` (45.33.32.156). Es el host oficial de prueba de Nmap y está **autorizado** para escanearlo.
> **Sistema:** Arch Linux (BlackArch) · Nmap 7.993
> **Fecha:** 2026-11-08

| Leyenda | Significado |
|---|---|
| ✅ **Real** | escaneo ejecutado de verdad en mi máquina |
| 🧪 **Simulado** | salida reconstruida con lo que se sabe de scanme (necesitaba `sudo` o tardaba demasiado) |

---

## 1. Instalación

En Kali/Ubuntu se instala con `apt`. En Arch se usa `pacman`:

```bash
sudo apt install nmap      # Kali / Ubuntu / Debian
sudo pacman -S nmap        # Arch / BlackArch
nmap --version             # verificar
```

En mi caso ya estaba instalado: `Nmap version 7.993`.

---

## 2. Los 5 escaneos básicos

| Comando | Qué hace | ¿Necesita sudo? |
|---|---|---|
| `nmap -sV <ip>` | detecta **qué software y qué versión** hay en cada puerto | No |
| `nmap -sS <ip>` | escaneo **SYN** ("sigiloso"): no completa la conexión | **Sí** |
| `nmap -O <ip>` | intenta adivinar el **sistema operativo** | **Sí** |
| `nmap -p 1-65535 <ip>` (o `-p-`) | escanea **todos** los puertos, no solo los 1000 comunes | No |
| `nmap -A <ip>` | **agresivo**: `-sV` + `-O` + scripts por defecto + traceroute | Sí (para `-O`) |

> 💡 **Aprendizaje:** `-sS` y `-O` necesitan root porque fabrican paquetes "a mano" (raw packets). Sin `sudo`, Nmap usa por defecto `-sT` (connect scan), que completa la conexión TCP como cualquier programa normal.

---

## 3. Escaneo de versiones: `-sV` ✅ Real

```bash
nmap -sV scanme.nmap.org
```

```
Nmap scan report for scanme.nmap.org (45.33.32.156)
Host is up (0.14s latency).
Not shown: 996 closed tcp ports (conn-refused)
PORT     STATE    SERVICE    VERSION
22/tcp   open     ssh        OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
80/tcp   open     http       Apache httpd 2.4.7 ((Ubuntu))
161/tcp  filtered snmp
9929/tcp open     nping-echo Nping echo
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
Nmap done: 1 IP address (1 host up) scanned in 42.42 seconds
```

### Cómo se lee
- **`open`**: hay un servicio escuchando ahí.
- **`closed`**: el puerto responde, pero no hay nada escuchando.
- **`filtered`**: no responde nada. Un **firewall** está bloqueando, así que Nmap no sabe si está abierto o cerrado (le pasó al 161/snmp).
- `conn-refused` confirma que se usó **connect scan** (sin sudo).
- Solo escaneó los **1000 puertos más comunes**: 996 cerrados + 4 mostrados.

### Lo importante
- `OpenSSH 6.6.1` y `Apache 2.4.7` son versiones **muy viejas** (de ~2014). En un pentest real, este es el momento de buscar vulnerabilidades conocidas (CVE) para esas versiones.
- Es lo mismo que el header `Server:` que vimos con `curl --head` en el Tema 7, pero Nmap lo hace en **todos** los puertos a la vez.

---

## 4. Escaneo SYN: `-sS` 🧪 Simulado

```bash
sudo nmap -sS scanme.nmap.org
```

```
Nmap scan report for scanme.nmap.org (45.33.32.156)
Host is up (0.14s latency).
Not shown: 995 closed tcp ports (reset)
PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    open     http
161/tcp   filtered snmp
9929/tcp  open     nping-echo
31337/tcp open     Elite
Nmap done: 1 IP address (1 host up) scanned in 8.12 seconds
```

### Cómo funciona: el saludo TCP (3-way handshake)

```
Conexión normal (-sT):   yo SYN  →  ←  SYN/ACK  servidor  →  yo ACK   (conexión completa)
Escaneo SYN (-sS):       yo SYN  →  ←  SYN/ACK  servidor  →  yo RST   (corto antes de terminar)
```

- Si el servidor responde **SYN/ACK**, el puerto está **abierto**.
- Si responde **RST**, está **cerrado** (por eso ahora dice `closed tcp ports (reset)`).
- Como nunca se completa la conexión, muchos servicios **no lo registran en sus logs**. Por eso le dicen "sigiloso".
- Es **más rápido** que `-sT`: aquí tardó ~8 s contra 42 s del `-sV`.

> ⚠️ "Sigiloso" no significa invisible: un firewall o un IDS moderno lo detecta igual.

---

## 5. Todos los puertos: `-p 1-65535` 🧪 Simulado

```bash
nmap -p 1-65535 -T4 scanme.nmap.org
```

```
PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    open     http
161/tcp   filtered snmp
9929/tcp  open     nping-echo
31337/tcp open     Elite
```

### Lo importante
- Por defecto Nmap solo revisa **1000 de 65535** puertos.
- El **31337** no apareció en el `-sV` del punto 3 porque **no está en el top 1000**.
- **Lección para HTB:** siempre hay que hacer un `-p-` en algún momento. Los servicios "raros" suelen estar en puertos altos, y son justamente los interesantes.
- `-T4` acelera el escaneo (las plantillas de tiempo van de T0, muy lento, a T5, muy rápido). Aun así, escanear los 65535 puertos por internet tarda **varios minutos**.

### Técnica que se usa en la práctica

```bash
sudo nmap -p- --min-rate 5000 -T4 <ip>   # 1) encontrar puertos abiertos (rápido)
nmap -sV -sC -p 22,80,9929,31337 <ip>    # 2) analizar SOLO esos puertos a fondo
```

---

## 6. Sistema operativo: `-O` 🧪 Simulado

```bash
sudo nmap -O scanme.nmap.org
```

```
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.19
Network Distance: 12 hops
```

### Cómo lo adivina
Nmap manda paquetes "raros" y compara cómo responde el sistema (TTL, tamaño de ventana TCP, opciones) con una base de datos de **huellas** (*fingerprints*) de sistemas operativos.

- Es la versión avanzada del truco del **TTL** del Tema 7 (64 = Linux, 128 = Windows).
- `Network Distance: 12 hops`: hay 12 saltos entre mi PC y scanme.
- Es una **aproximación**: por eso da un rango (`4.15 - 5.19`) y no una versión exacta.

---

## 7. Escaneo agresivo: `-A` 🧪 Simulado

```bash
sudo nmap -A scanme.nmap.org
```

```
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   1024 ac:00:a0:1a:82:ff:cc:55:99:dc:67:2b:34:97:6b:75 (DSA)
|   2048 20:3d:2d:44:62:2a:b0:5a:9d:b5:b3:05:14:c2:a6:b2 (RSA)
|   256 96:02:bb:5e:57:54:1c:4e:45:2f:56:4c:4a:24:b2:57 (ECDSA)
|_  256 33:fa:91:0f:e0:e1:7b:1f:6d:05:a2:b0:f1:54:41:56 (ED25519)
80/tcp    open  http       Apache httpd 2.4.7 ((Ubuntu))
|_http-title: Go ahead and ScanMe!
|_http-server-header: Apache/2.4.7 (Ubuntu)
9929/tcp  open  nping-echo Nping echo
31337/tcp open  tcpwrapped
Running: Linux 4.X|5.X
OS details: Linux 4.15 - 5.19

TRACEROUTE (using port 80/tcp)
HOP RTT       ADDRESS
1   5.12 ms   192.168.1.1
...
12  141.20 ms scanme.nmap.org (45.33.32.156)
```

### Qué le suma `-A`
| Parte | Viene de |
|---|---|
| Versiones | `-sV` |
| Sistema operativo | `-O` |
| `ssh-hostkey`, `http-title`, `http-server-header` | **scripts NSE por defecto** (`-sC`) 👈 tema de la Semana 2 |
| Traceroute | ruta salto por salto (el salto 1 es **mi router** 192.168.1.1) |

### Lo importante
- Las líneas que empiezan con `|` las producen **scripts**. Ese es el puente hacia el **NSE (Nmap Scripting Engine)**.
- `ssh-hostkey` son las mismas llaves que mi PC guarda en `known_hosts` (Tema 7, SSH).
- `tcpwrapped` en el 31337 significa que el puerto aceptó la conexión TCP pero cortó sin decir qué servicio es.
- **Desventaja:** `-A` es **muy ruidoso** y lento. En un pentest real se usa sobre puertos ya conocidos, no a ciegas.

---

## 📋 Resumen de la clase

| Escaneo | Pregunta que responde | Puertos abiertos encontrados |
|---|---|---|
| `-sV` | ¿qué software y versión hay? | 22, 80, 9929 (+161 filtrado) |
| `-sS` | ¿qué está abierto? (rápido, sin completar la conexión) | 22, 80, 9929, 31337 |
| `-p-` | ¿hay algo fuera del top 1000? | **31337** (escondido) |
| `-O` | ¿qué sistema operativo es? | Linux 4.x–5.x |
| `-A` | todo junto + scripts | 22, 80, 9929, 31337 |

### Puntos clave a no olvidar
1. **open / closed / filtered**: filtered = hay un firewall de por medio.
2. **Sin sudo** Nmap hace `-sT` (conexión completa). **Con sudo** hace `-sS` (SYN).
3. **El top 1000 no es todo**: siempre un `-p-` en HTB.
4. Las **versiones** (`-sV`) son lo que lleva a buscar vulnerabilidades.
5. Las líneas con `|` vienen de **scripts NSE**, que es lo que sigue.
6. ⚖️ Solo se escanea lo **autorizado**: scanme, HTB, TryHackMe y mis VMs.

### Conexión con el Tema 7 (Redes)
| Tema 7 | Nmap |
|---|---|
| TTL en ping (64/128) | `-O` (detección de SO más avanzada) |
| `ss -tulnp` (mis puertos) | `nmap` (los puertos de otro) |
| header `Server:` con curl | `-sV` |
| `known_hosts` de SSH | script `ssh-hostkey` |

---

## ➡️ Siguiente: Semana 2 · Introducción a NSE
- Qué es el **Nmap Scripting Engine** y dónde viven los scripts (`/usr/share/nmap/scripts/`)
- Categorías: `default`, `safe`, `vuln`, `discovery`...
- `-sC`, `--script=http-title`, `--script-help`
