---
title: "Research Paper: Trust Boundary Failures in AI Coding Agents"
date: 2026-03-14
description: "Empirical analysis of MCP configuration attacks in Claude Code with enterprise defensive architecture recommendations."
tags: ["security-research", "claude-code", "mcp", "research-paper"]
categories: ["security-research"]
---

## Trust Boundary Failures in AI Coding Agents: Empirical Analysis of MCP Configuration Attacks in Claude Code

**DOI:** [10.5281/zenodo.19011781](https://doi.org/10.5281/zenodo.19011781)

**Full Paper:** [Zenodo](https://zenodo.org/records/19011781)

### Abstract

AI coding agents grant large language models access to file systems, terminals, and external services through protocols such as the Model Context Protocol (MCP). The trust models governing that access were designed for human users, not autonomous agents processing attacker-controlled input. This paper presents three empirical findings in Anthropic's Claude Code (v2.1.63) demonstrating systemic trust boundary failures in MCP server configuration handling, tool confirmation prompts, and workspace trust escalation. All findings were reported through Anthropic's HackerOne Vulnerability Disclosure Program and closed as Informative. Rather than contesting that design decision, this paper reframes the findings from an enterprise defensive perspective and proposes compensating controls including virtual desktop infrastructure (VDI) isolation, MCP configuration integrity monitoring, and credential management practices adapted for AI-assisted development workflows.

### Findings

**Finding 1: Silent Command Execution via .mcp.json Trust Persistence.** When a user accepts an MCP server, the trust decision is cached by server name, not configuration. Modified .mcp.json files execute silently on the next launch without re-prompting. CWE-78, CWE-356.

**Finding 2: Blanket Trust Escalation via enableAllProjectMcpServers.** The "Use this and all future MCP servers" option grants permanent, unbounded trust to any MCP server definition added after the initial consent, with no expiration or revocation mechanism. CWE-356.

**Finding 3: Tool Confirmation Prompt Misrepresentation.** MCP tool confirmation prompts display tool name, parameters, and description provided entirely by the MCP server with no validation, enabling a complete disconnect between what the user approves and what actually executes. CWE-451.

### Enterprise Defensive Architecture

The paper proposes six categories of compensating controls for enterprises deploying AI coding agents: VDI and cloud development environment isolation, MCP configuration integrity monitoring, workspace trust policy enforcement, network segmentation for MCP processes, developer environment hardening, and detection and response capabilities.

### Proof-of-Concept Repositories

- [claude-code-mcp-rce](https://github.com/jashidsany/claude-code-mcp-rce)
- [claude-code-mcp-trust-escalation](https://github.com/jashidsany/claude-code-mcp-trust-escalation)
- [claude-code-mcp-prompt-mismatch-poc](https://github.com/jashidsany/claude-code-mcp-prompt-mismatch-poc)
