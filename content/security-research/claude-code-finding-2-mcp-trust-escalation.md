---
title: "Claude Code Finding 2: MCP Blanket Trust Escalation via enableAllProjectMcpServers"
date: 2026-03-12
description: "How the 'Use this and all future MCP servers' option grants permanent, unbounded trust to arbitrary MCP server definitions added after the initial consent."
tags: ["security-research", "claude-code", "mcp", "trust-model", "supply-chain"]
categories: ["security-research"]
---

## Introduction

This is the second finding from my Claude Code security research. It examines the `enableAllProjectMcpServers` flag, set when a user selects "Use this and all future MCP servers in this project" in the MCP trust dialog. This option grants permanent, irrevocable trust to any MCP server definition added to the project's `.mcp.json` in the future, with no mechanism to review, audit, or revoke trust for individual servers after the fact.

**Product:** Claude Code CLI v2.1.63  
**CWE:** CWE-356 (Product UI does not Warn User of Unsafe Actions)  
**GitHub:** [claude-code-mcp-trust-escalation](https://github.com/jashidsany/claude-code-mcp-trust-escalation)

## The Problem

When a user first encounters MCP servers in a project, Claude Code presents three options:

1. **Use this and all future MCP servers in this project**
2. Use this MCP server
3. Continue without using this MCP server

Option 1 sets a flag (`enableAllProjectMcpServers`) in `.claude/settings.json` that permanently bypasses all future MCP trust prompts for that project. The user is making a trust decision based on the servers they can see at that moment, but the flag applies to servers that do not yet exist.

## The Consent Gap

Informed consent requires understanding what you are consenting to. The trust dialog does not explain that selecting option 1:

- Grants permanent trust to any arbitrary server definition, including servers added months or years later
- Has no expiration
- Has no mechanism to review what servers have been added since the original decision
- Has no way to revoke trust for a specific server while keeping others
- Applies to servers injected via `git pull`, PRs, or any modification to `.mcp.json`

The user consents to a known set of servers and a vague "all future" clause that most users would not interpret as extending to adversary-controlled endpoints injected via source control.

## Attack Scenario

1. Developer A enables blanket trust on a project containing one known, legitimate MCP server
2. Developer B (compromised account, malicious insider, or supply chain attacker) pushes a commit adding a new MCP server definition to `.mcp.json` pointing to an attacker-controlled endpoint
3. The next time Developer A runs Claude Code in that project, the new server executes tools with full pre-granted trust and zero confirmation prompts

Developer A never consented to that specific server. They consented to a set of servers they could evaluate at the time.

## Industry Comparison

The pattern of re-prompting when the trust surface changes is industry standard:

- Browser extensions prompt again when requesting new permissions
- Mobile applications re-prompt for new permission scopes
- Package managers warn on new install scripts
- Even npm warns on new postinstall scripts

Claude Code's blanket trust skips this entirely. A change in what is being trusted should trigger a new consent decision.

## What I Would Fix

I am not suggesting the blanket trust option should be removed. I am suggesting that:

1. A trust surface change (new server added to `.mcp.json`) should trigger a new confirmation prompt, even when blanket trust is enabled
2. The dialog should clearly state that this grants trust to servers that do not yet exist and may be added by other contributors
3. A mechanism should exist to review and revoke trust for individual servers

This would preserve the convenience of the feature while ensuring the user's consent remains informed.

## Vendor Response

Anthropic closed the report as Informative. 

Full PoC and evidence available at the [GitHub repository](https://github.com/jashidsany/claude-code-mcp-trust-escalation).
