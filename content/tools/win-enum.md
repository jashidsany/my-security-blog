---
title: "Win-Enum: Windows & Active Directory Auto-Enumerator"
date: 2026-02-22
draft: false
description: "A Python tool to automate Windows and Active Directory enumeration for penetration testing and OSCP preparation."
tags: ["tools", "python", "active-directory", "windows", "enumeration", "oscp"]
---

## Introduction

Win-Enum is an automated enumeration tool I built to speed up the initial reconnaissance phase when targeting Windows machines and Active Directory environments. It runs common enumeration tools in sequence and organizes the output for easy review.

- **Language:** Python 3
- **Target:** Windows / Active Directory
- **Purpose:** OSCP preparation, penetration testing

**GitHub:** [github.com/jashidsany/win-enum](https://github.com/jashidsany/win-enum)

---

## Why I Built This

During OSCP preparation, I found myself running the same enumeration commands repeatedly:

1. Nmap scan
2. SMB null session check
3. User enumeration
4. AS-REP roasting attempt
5. Web directory brute forcing

This tool automates all of that and saves output in an organized structure.

---

## Features

| Feature | Description |
|---------|-------------|
| **Auto-detection** | Asks if target is AD and adjusts scans accordingly |
| **Nmap scanning** | Quick scan + full port scan |
| **SMB enumeration** | Null session, shares, guest access |
| **WinRM check** | Test for remote PowerShell access |
| **Web enumeration** | Gobuster on common ports |
| **AD user enum** | RID brute, LDAP, Kerbrute |
| **AS-REP roasting** | Automatic hash extraction |
| **Summary report** | Quick findings + next steps |

---

## Usage
```bash
# Basic usage - will prompt if AD
python3 win-enum.py 192.168.1.100

# Active Directory target
python3 win-enum.py 192.168.1.100 --ad -d domain.local

# Non-AD Windows target
python3 win-enum.py 192.168.1.100 --no-ad

# Custom output directory
python3 win-enum.py 192.168.1.100 -o ./target-name
```

---

## Output Structure

The tool creates an organized directory structure:
```bash
target-ip/
├── nmap/
│   ├── quick.txt
│   └── full.txt
├── smb/
│   ├── shares_null.txt
│   ├── netexec_shares.txt
│   └── winrm_check.txt
├── web/
│   └── gobuster_80.txt
├── ldap/
│   ├── rid_brute.txt
│   └── users.txt
├── kerberos/
│   ├── kerbrute_users.txt
│   └── asrep.txt
└── notes.md
```

---

## Installation

The tool comes with an install script that checks and installs all dependencies:
```bash
# Clone the repo
git clone https://github.com/jashidsany/win-enum.git
cd win-enum

# Run installer
chmod +x install.sh
./install.sh

# Run the tool
python3 win-enum.py --help
```

### Required Tools

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| smbclient | SMB enumeration |
| netexec | Multi-protocol enumeration |
| gobuster | Web directory brute forcing |
| ldapsearch | LDAP queries |
| kerbrute | Kerberos user enumeration |
| impacket | AS-REP roasting |

---

## Example Workflow

Here's how I use this tool during a penetration test:
```bash
# 1. Run enumeration
python3 win-enum.py 192.168.235.172 --ad -d vault.offsec

# 2. Check summary for quick wins
cat 192.168.235.172/notes.md

# 3. Review discovered users
cat 192.168.235.172/ldap/users.txt

# 4. Check for AS-REP hashes
cat 192.168.235.172/kerberos/asrep.txt

# 5. Crack any hashes found
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

---

## What I Learned Building This

- **Python subprocess module** - Running external tools and capturing output
- **Concurrent execution** - Threading for parallel scans
- **Error handling** - Gracefully handling tool failures and timeouts
- **Output parsing** - Extracting useful data from tool outputs

---

## Future Improvements

- Add Kerberoasting
- Add password spraying module
- HTML report generation
- Integration with BloodHound
- SMB share file searching

---

## Download

**GitHub:** [github.com/jashidsany/win-enum](https://github.com/jashidsany/win-enum)
```bash
git clone https://github.com/jashidsany/win-enum.git
```

---

*This tool is for authorized penetration testing and educational purposes only.*
