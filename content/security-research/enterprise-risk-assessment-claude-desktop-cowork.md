---
title: "Enterprise Risk Assessment: Claude Desktop and Cowork Security"
date: 2026-03-14
draft: false
categories: ["security-research"]
tags: ["security-research", "claude-desktop", "cowork", "enterprise-security", "risk-assessment", "research-paper"]
summary: "Comprehensive security risk assessment for organizations deploying Claude Desktop and Cowork in regulated environments, covering publicly disclosed vulnerabilities and independent research findings."
---

## Enterprise Risk Assessment: Security Risks of Deploying Claude Desktop and Cowork in Regulated Environments

**DOI:** [10.5281/zenodo.19024890](https://doi.org/10.5281/zenodo.19024890)

**Full Paper:** [Zenodo](https://zenodo.org/records/19024890)

### Abstract

Claude Desktop and its Cowork feature represent a new class of agentic AI productivity tools that operate directly on users' local filesystems, execute code in virtual machines, and interact with enterprise productivity suites through desktop extensions. This paper presents a comprehensive security risk assessment for organizations considering deployment of these tools, particularly those handling PII, PHI, financial data, or other regulated datasets. Drawing on publicly disclosed vulnerabilities, independent security research, and architectural analysis, we identify systemic security deficiencies that pose material risks to enterprise data confidentiality, system integrity, and regulatory compliance.

### Vulnerability Landscape

**PromptArmor: File Exfiltration via Indirect Prompt Injection (January 2026).** A malicious document containing hidden instructions can trick Cowork into uploading sensitive files to an attacker's Anthropic account. The attack requires zero additional user approval and bypasses VM network restrictions by using the allowlisted Anthropic API domain. The vulnerability was known before Cowork shipped.

**LayerX: Zero-Click RCE via Desktop Extension MCP Chaining (February 2026).** A single malicious Google Calendar event can trigger arbitrary code execution on systems running Claude Desktop Extensions, without any user interaction. CVSS 10.0. Affects 10,000+ users and 50+ extensions. Anthropic declined to fix, stating it "falls outside our current threat model."

**Koi Research: Command Injection in Official Anthropic Extensions (February 2026).** Three official Anthropic-published Desktop Extensions (Chrome, iMessage, Apple Notes) contained unsanitized command injection. CVSS 8.9. Over 350,000 combined downloads. Fixed in v0.1.9.

**Author's Research: CoworkVMService Access Control and Boot Media Vulnerabilities (March 2026).** Seven vulnerabilities in the Windows CoworkVMService allowing a standard user to: connect to the SYSTEM service pipe (world-writable DACL), bypass signature authentication, create arbitrary VM sessions, mount arbitrary host paths, stop/start the SYSTEM service, and tamper with VM boot media for persistent automatic code execution. Submitted to Anthropic via HackerOne.

### Key Risk Findings

**Trust model mismatch.** Anthropic assumes single-user workstations. Enterprise environments have shared machines, terminal servers, and CI/CD runners where the access control failures are directly exploitable.

**Regulatory exposure.** The proven file exfiltration vulnerability creates direct risks under HIPAA, PCI DSS, SOX, GDPR, and CCPA. A data breach through prompt injection triggers notification obligations.

**Unsandboxed extensions.** Desktop Extensions run with full OS privileges, not in a sandbox. Any vulnerability in any extension grants full system access.

**No integrity verification.** VM boot media (rootfs.vhdx) is user-owned with no hash check before boot, enabling persistent backdoor implantation.

### Defensive Architecture

The paper proposes compensating controls across five categories: VDI/containerized deployment to isolate Cowork from production data, extension allowlisting and security review, CoworkVMService DACL hardening and file integrity monitoring, network controls for API traffic inspection, and AI-specific incident response procedures.

### Recommendations

1. Do not deploy Cowork on machines that store or process PII, PHI, or financial data
2. Require VDI or containerized deployment for any Cowork usage
3. Disable Desktop Extensions unless security-reviewed and approved
4. Implement service hardening and boot media integrity monitoring
5. Establish AI-specific incident response procedures before deployment

### Related Work

* [Trust Boundary Failures in AI Coding Agents](https://doi.org/10.5281/zenodo.19011781) (prior paper on Claude Code MCP attacks)
* [cowork-pipe-access-control](https://github.com/jashidsany/cowork-pipe-access-control) (private, pending disclosure)
* [cowork-boot-media-tampering](https://github.com/jashidsany/cowork-boot-media-tampering) (private, pending disclosure)
