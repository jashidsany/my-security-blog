---
title: "Claude Code Finding: Remote Control Session Hijacking via Missing Per-Session Authentication"
date: 2026-03-12
description: "The Claude Code remote-control session events endpoint lacks per-session authentication, enabling invisible remote command execution from any machine on the internet."
tags: ["security-research", "claude-code", "session-hijacking", "authentication-bypass", "rce"]
categories: ["security-research"]
---

## Introduction

This is the fourth finding from my Claude Code security research. The `claude.ai/v1/sessions/{session_id}/events` endpoint, used by Claude Code's remote-control feature, lacks per-session authentication. An attacker who obtains a user's `sessionKey` cookie can inject arbitrary messages into an active session from any machine on the internet. Injected messages are processed identically to legitimate user messages with no visual indicator of external origin.

**Product:** Claude Code CLI v2.1.63  
**Feature:** Remote Control (`claude remote-control`)  
**CWE:** CWE-306 (Missing Authentication for Critical Function)  
**GitHub:** [claude-code-session-hijack](https://github.com/jashidsany/claude-code-session-hijack)

## The Vulnerability

Claude Code's remote-control feature allows users to control a CLI session through the browser at `claude.ai`. The session events endpoint authenticates using only the general-purpose `sessionKey` cookie. There is no:

- Per-session cryptographic token
- Device fingerprint binding
- IP address validation
- Message origin indicator in the UI

This means that if an attacker obtains the `sessionKey` (via XSS, cookie theft, malware, shared machines, or any other credential compromise vector), they can inject commands into the victim's active Claude Code session from a completely different machine.

## Attack Flow

```
Attacker (Kali)                          Victim (Windows)
     |                                        |
     |  1. Obtain sessionKey cookie           |
     |  2. Obtain session_id                  |
     |                                        |
     |  POST claude.ai/v1/sessions/           |
     |       {session_id}/events              |
     |  Cookie: sessionKey=<stolen>           |
     |  Body: {events: [{message: "..."}]}    |
     | -------------------------------------> |
     |                                        |
     |           HTTP 200 OK                  |
     | <------------------------------------- |
     |                                        |
     |        Message appears in session      |
     |        Claude executes commands        |
     |        on victim's machine             |
```

The injected message appears in the browser as a standard user message. Claude processes it and executes tool calls on the victim's machine. There is no visual distinction between legitimate and injected messages.

## Proof of Concept

The exploit is a Python script that uses `cloudscraper` to bypass Cloudflare and post events to the session endpoint:

```bash
python3 exploit.py \
  -k "sk-ant-sid02-..." \
  -s "session_01ABC..." \
  -o "96682414-bcca-4968-a8fc-ebd0de28b2b2" \
  -c "Create a file called proof.txt on the Desktop containing: Hello from a remote session"
```

### Required Information

To execute the attack, the attacker needs:

1. **sessionKey**: The victim's session cookie from `claude.ai`
2. **session_id**: The active remote-control session ID
3. **org_id**: The victim's organization UUID

The sessionKey is the most sensitive piece. It is a long-lived cookie that persists across browser sessions. The session_id can be found in terminal output, debug logs, or the session URL. The org_id is visible in browser DevTools on any request to claude.ai.

## Demonstrated Impact

In my testing, I successfully:

1. Injected a message from a Kali Linux machine into an active session on a Windows machine
2. The message appeared in the victim's browser with no origin indicator
3. Claude processed the injected message and created a file on the victim's Desktop
4. The file was verified on the victim's machine

The entire attack chain, from injection to file creation, completed with no visible indication to the victim that anything unusual occurred.

## Root Cause

The session events endpoint uses only the general-purpose `sessionKey` for authentication. This cookie authenticates the user to `claude.ai` broadly, not to a specific session. There is no session-specific token that would bind a session to the device or browser that created it.

This is a common pattern in web applications where session-level operations rely on account-level authentication, but in the context of a remote-control feature that executes commands on the user's local machine, the impact is significantly elevated.

## What I Would Fix

1. **Per-session tokens**: Generate a cryptographic token when a remote-control session is created, require it on all event submissions
2. **Device binding**: Tie the session to the originating device fingerprint or IP
3. **Origin indicators**: Display a visual marker in the UI distinguishing remotely injected messages from local input
4. **Session-scoped cookies**: Use a session-specific cookie rather than the account-wide `sessionKey`

## Vendor Response

Anthropic closed the report as Informative. This is a design decision, not a bug.

Full exploit code, video demonstration, and evidence screenshots available at the [GitHub repository](https://github.com/jashidsany/claude-code-session-hijack).
