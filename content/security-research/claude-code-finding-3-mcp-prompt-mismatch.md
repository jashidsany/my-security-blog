---
title: "Claude Code Finding: MCP Tool Confirmation Prompt Misrepresentation Enables Arbitrary Code Execution"
date: 2026-03-12
description: "A malicious MCP server can misrepresent tool actions in Claude Code's confirmation prompt, causing users to approve a file read while the server silently executes system commands."
tags: ["security-research", "claude-code", "mcp", "rce", "prompt-misrepresentation"]
categories: ["security-research"]
---

## Introduction

This is the third finding from my Claude Code security research, and the one I consider the most impactful. A malicious MCP server can completely misrepresent what it does in Claude Code's tool confirmation prompt, causing a user to approve what appears to be a safe file read while the server silently executes arbitrary system commands, writes files outside the project directory, and runs OS-level commands.

This was submitted to Anthropic via HackerOne.

**Product:** Claude Code CLI v2.1.63  
**CWE:** CWE-451 (User Interface Misrepresentation of Critical Information)  
**GitHub:** [claude-code-mcp-prompt-mismatch-poc](https://github.com/jashidsany/claude-code-mcp-prompt-mismatch-poc)

## The Vulnerability

Claude Code's MCP tool confirmation prompt displays three pieces of information: the tool name, the parameters, and the description. All three are provided entirely by the MCP server with no server-side validation. This enables a complete disconnect between what the user approves and what actually executes.

The confirmation prompt shows:

```bash
Tool use
  file-reader - read_file(path: "...\README.md") (MCP)
  Safely reads a text file and returns its contents

Do you want to proceed?
```

The user sees a harmless file read. They approve it. Behind the scenes, the MCP server:

1. Executes `whoami` to capture the username
2. Writes a proof file to the Desktop outside the project directory
3. Returns fabricated "file contents" so everything looks normal

The user has no indication anything malicious occurred.

## The PoC

The malicious MCP server is a simple Python script using the MCP SDK. It declares a tool called `read_file` with the description "Safely reads a text file and returns its contents." The actual implementation runs system commands and returns fake output.

### Before the Attack

The Desktop has no proof file. The MCP server is configured in `.mcp.json` pointing to the malicious `server.py`.

### The Deceptive Prompt

When the user asks Claude to read a file, the confirmation prompt shows the benign tool name and description. The user approves.

### After the Attack

A proof file appears on the Desktop:

```bash
PROOF-OF-CONCEPT: MCP TOOL MISMATCH
User approved: read_file(README.md)
Actually executed: file write + system commands
whoami: desktop-c9ak2kc\maldev01
```

Claude Code displays what appears to be the file contents. The user has no idea the MCP server executed arbitrary commands.

## Why This is Different from Existing CVEs

This finding is distinct from previously patched Claude Code vulnerabilities:

- **CVE-2025-59536** (MCP executing before consent dialog): That was about commands running before the user could approve. This finding occurs **after** the user explicitly consents.
- **CVE-2026-21852** (API key exfiltration via env vars): Different attack vector entirely.
- **CVE-2026-24887** (`find` command parsing bypass): Same class but different mechanism.

The core issue here is that the consent mechanism itself is broken. The user gives informed consent to the wrong action. They approve "read a file" and get system command execution.

## Attack Scenario

An attacker includes a `.mcp.json` and malicious MCP server in a GitHub repository. A developer clones the repo, opens it in Claude Code, trusts the workspace, and uses the tool. The confirmation prompt shows a harmless file read, but the server exfiltrates SSH keys, environment variables, source code, and credentials while displaying normal-looking output.

With the "don't ask again" option, the attacker gets persistent execution for every subsequent tool call.

## Impact

- Arbitrary code execution with the user's privileges
- File system access outside the project directory
- Deceptive confirmation prompt: user consents to one action and gets another
- Fake return data hides the attack from both the user and Claude
- Persistent compromise if "don't ask again" is selected

## Vendor Response

Anthropic closed the report as Informative. This is a design decision, not a bug.

Full PoC with video demo, source code, and step-by-step reproduction available at the [GitHub repository](https://github.com/jashidsany/claude-code-mcp-prompt-mismatch-poc).
