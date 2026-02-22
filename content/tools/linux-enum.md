---
title: "Linux-Enum: Linux Auto-Enumerator"
date: 2026-02-22
draft: false
description: "A Python tool to automate Linux enumeration for penetration testing and OSCP preparation."
tags: ["tools", "python", "linux", "enumeration", "oscp"]
---

## Introduction

Linux-Enum is an automated enumeration tool I built to speed up the initial reconnaissance phase when targeting Linux machines. It detects open services and runs the appropriate enumeration tools, organizing all output for easy review.

- **Language:** Python 3
- **Target:** Linux systems
- **Purpose:** OSCP preparation, penetration testing

**GitHub:** [github.com/jashidsany/linux-enum](https://github.com/jashidsany/linux-enum)

---

## Why I Built This

During OSCP preparation, I found myself running the same enumeration sequence on every Linux box:

1. Nmap scan
2. Gobuster if web is open
3. enum4linux if SMB is open
4. showmount if NFS is open
5. snmpwalk if SNMP is open

This tool automates all of that and intelligently runs tools based on what ports are open.

---

## Features

| Feature | Tools Used |
|---------|------------|
| **Port Scanning** | nmap (quick, full TCP, UDP top 20) |
| **Web Enumeration** | gobuster, dirsearch, ffuf, nikto, curl |
| **SMB Enumeration** | smbclient, enum4linux-ng, smbmap |
| **NFS Enumeration** | showmount, rpcinfo |
| **SNMP Enumeration** | snmpwalk, snmp-check |
| **FTP Enumeration** | Anonymous check, nmap scripts |
| **SMTP Enumeration** | smtp-user-enum, nmap scripts |

---

## Usage
```bash
# Basic scan - runs everything
python3 linux-enum.py 192.168.1.100

# Quick scan - skip full port scan
python3 linux-enum.py 192.168.1.100 --quick

# Skip nikto for faster web enum
python3 linux-enum.py 192.168.1.100 --skip-nikto

# Skip web enumeration entirely
python3 linux-enum.py 192.168.1.100 --skip-web

# Custom output directory
python3 linux-enum.py 192.168.1.100 -o ./target-name
```

---

## Output Structure

The tool creates an organized directory structure:
```bash
target-ip/
├── nmap/
│   ├── quick.txt
│   ├── full.txt
│   └── udp.txt
├── web/
│   ├── headers_80.txt
│   ├── robots_80.txt
│   ├── gobuster_80.txt
│   ├── dirsearch_80.txt
│   ├── ffuf_80.json
│   └── nikto_80.txt
├── smb/
│   ├── shares.txt
│   ├── enum4linux.*
│   └── smbmap.txt
├── nfs/
│   └── showmount.txt
├── snmp/
│   ├── snmpwalk_public.txt
│   └── snmp-check.txt
├── ftp/
│   ├── anonymous.txt
│   └── nmap_ftp.txt
├── rpc/
│   └── rpcinfo.txt
└── notes.md
```

---

## Intelligent Service Detection

The tool parses nmap output and only runs relevant enumeration:

| Service Detected | Tools Run |
|------------------|-----------|
| HTTP (80, 443, 8080) | gobuster, dirsearch, ffuf, nikto |
| SMB (139, 445) | smbclient, enum4linux-ng, smbmap |
| NFS (2049) | showmount, rpcinfo |
| SNMP (161/udp) | snmpwalk, snmp-check |
| FTP (21) | Anonymous check, nmap scripts |
| SMTP (25) | smtp-user-enum, nmap scripts |

---

## Installation

The tool comes with an install script that checks and installs all dependencies:
```bash
# Clone the repo
git clone https://github.com/jashidsany/linux-enum.git
cd linux-enum

# Run installer
chmod +x install.sh
./install.sh

# Run the tool
python3 linux-enum.py --help
```

### Required Tools

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| gobuster | Directory brute forcing |
| dirsearch | Directory brute forcing |
| ffuf | Fast web fuzzing |
| nikto | Web vulnerability scanning |
| enum4linux-ng | SMB enumeration |
| smbmap | SMB share mapping |
| showmount | NFS enumeration |
| snmpwalk | SNMP enumeration |

---

## Example Workflow

Here's how I use this tool during a penetration test:
```bash
# 1. Run enumeration
python3 linux-enum.py 192.168.235.71

# 2. Check summary for quick wins
cat 192.168.235.71/notes.md

# 3. Review web directories
cat 192.168.235.71/web/gobuster_80.txt

# 4. Check for SMB null session
cat 192.168.235.71/smb/shares.txt

# 5. Look for NFS exports to mount
cat 192.168.235.71/nfs/showmount.txt

# 6. Check SNMP for system info
cat 192.168.235.71/snmp/snmpwalk_public.txt
```

---

## Summary Report

The tool generates a `notes.md` file with:

- List of open services
- Quick wins found (FTP anonymous, SMB null session, NFS exports, etc.)
- Suggested next steps

Example output:
```markdown
# Enumeration Summary
Target: 192.168.235.71

## Open Services
- Web: ports 80
- SMB: ports 139/445
- FTP: port 21

## Quick Findings
- [!] FTP anonymous access allowed
- [!] SMB null session allowed

## Next Steps
1. Review nmap output for all versions
2. Check web directories for interesting files
3. Look for default credentials
```

---

## What I Learned Building This

- **Service detection** - Parsing nmap output to determine what's running
- **Tool orchestration** - Running multiple tools and handling timeouts
- **Output organization** - Structuring results for quick review
- **Error handling** - Gracefully handling missing tools and failed commands

---

## Download

**GitHub:** [github.com/jashidsany/linux-enum](https://github.com/jashidsany/linux-enum)
```bash
git clone https://github.com/jashidsany/linux-enum.git
```

---

*This tool is for authorized penetration testing and educational purposes only.*
