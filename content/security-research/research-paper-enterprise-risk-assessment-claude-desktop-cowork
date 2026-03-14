---
title: "Research Paper: Enterprise Risk Assessment - Security Risks of Deploying Claude Desktop and Cowork in Regulated Environments"
date: 2026-03-14
categories: ["security-research"]
tags: ["security-research", "claude-desktop", "cowork", "enterprise-security", "risk-assessment", "prompt-injection", "agentic-ai", "hipaa", "gdpr", "pii"]
summary: "A comprehensive security risk assessment for organizations considering deployment of Claude Desktop and Cowork, covering publicly disclosed vulnerabilities, independent research findings, regulatory compliance risks, and a defensive architecture for enterprise environments."
---

**Author:** Jashid Sany
**Date:** March 2026
**Contact:** [github.com/jashidsany](https://github.com/jashidsany) | [jashidsany.com](https://jashidsany.com)
**DOI:** [10.5281/zenodo.19024890](https://doi.org/10.5281/zenodo.19024890)

## Abstract

Claude Desktop and its Cowork feature, released by Anthropic in January and February 2026, represent a new class of agentic AI productivity tools that operate directly on users' local filesystems, execute code in virtual machines, and interact with enterprise productivity suites through desktop extensions. This paper presents a comprehensive security risk assessment for organizations considering deployment of these tools, particularly those handling personally identifiable information (PII), protected health information (PHI), financial data, or other regulated datasets. Drawing on publicly disclosed vulnerabilities, independent security research, and architectural analysis, we identify systemic security deficiencies in the Claude Desktop and Cowork platform that pose material risks to enterprise data confidentiality, system integrity, and regulatory compliance. We propose a defensive architecture consisting of compensating controls, deployment constraints, and monitoring capabilities that organizations should implement before, during, and after any adoption of these tools.

## 1. Introduction

### 1.1 Background

Anthropic released Claude Desktop for macOS in January 2026 and for Windows in February 2026. The Cowork feature, integrated into Claude Desktop, provides an agentic AI assistant capable of autonomous file analysis, code execution, and interaction with enterprise tools through the Model Context Protocol (MCP). Unlike traditional chatbot interfaces, Cowork operates with direct access to the user's local filesystem, can execute arbitrary code inside a managed virtual machine, and can chain together multiple tools autonomously to fulfill user requests.

### 1.2 Scope

This assessment covers security vulnerabilities and architectural risks specific to Claude Desktop (the native application) and its Cowork feature. It does not cover Claude Code (the command-line developer tool), the Claude API, or the claude.ai web interface, which have distinct threat models and attack surfaces. The assessment draws on the following sources:

1. Publicly disclosed vulnerabilities by PromptArmor (January 2026), LayerX (February 2026), and Koi Research (February 2026)
2. Independent security research by the author, submitted to Anthropic via HackerOne (March 2026)
3. Anthropic's own system card disclosures and product documentation
4. Architectural analysis of the Windows CoworkVMService and Desktop Extensions (DXT) framework

### 1.3 Intended Audience

This paper is intended for Chief Information Security Officers (CISOs), enterprise security architects, IT governance committees, compliance officers, and technical leaders evaluating the adoption of AI productivity tools. It provides both executive-level risk summaries and technical detail sufficient for security engineers to implement the recommended controls.

## 2. Architecture Overview

### 2.1 Claude Desktop

Claude Desktop is an Electron-based native application distributed as an MSIX package on Windows and a standard application bundle on macOS. It provides the chat interface, manages user authentication via OAuth, and communicates with Anthropic's cloud API for model inference.

### 2.2 Cowork and the Virtual Machine

The Cowork feature executes Claude Code inside a sandboxed virtual machine. On Windows, this VM is managed by CoworkVMService (`cowork-svc.exe`), a service running as NT AUTHORITY\SYSTEM. The VM is a lightweight Linux environment running on Hyper-V, booting from VHDX disk images stored in the user's `%APPDATA%` directory. Host-to-VM file sharing is accomplished through Plan9/virtio-fs shares, which map host directories into the VM with the user's access token.

Key architectural components on Windows:

- **CoworkVMService** (SYSTEM context): manages VM lifecycle, Plan9 shares, and IPC
- **Named pipe** (`\\.\pipe\cowork-vm-service`): IPC channel between Claude Desktop and the SYSTEM service
- **VM boot media**: rootfs.vhdx, sessiondata.vhdx, smol-bin.vhdx, vmlinuz, initrd (all user-owned)
- **Plan9 shares**: bidirectional file access between VM and host filesystem

### 2.3 Desktop Extensions (DXT)

Claude Desktop Extensions are packaged MCP servers distributed through Anthropic's extension marketplace as `.mcpb` bundles. Each extension runs unsandboxed with full operating system privileges, bridging Claude's AI model and the user's operating system. Extensions can read files, execute commands, access credentials, and modify system settings. Claude autonomously selects and chains extensions to fulfill user requests.

## 3. Vulnerability Landscape

### 3.1 PromptArmor: File Exfiltration via Indirect Prompt Injection (January 2026)

Two days after Cowork's public release, security firm PromptArmor demonstrated that a malicious document containing hidden prompt injection instructions could trick Cowork into uploading sensitive files to an attacker's Anthropic account. The attack exploits the fact that the Cowork VM sandbox allowlists the Anthropic API domain (`api.anthropic.com`), permitting data egress through what appears to be legitimate API traffic.

**Key findings:**

- The attack requires zero additional user approval once Cowork has been granted folder access
- Hidden instructions can be concealed in `.docx` files using 1-point font, white text, and 0.1 line spacing
- Both Claude Haiku and Claude Opus 4.5 were successfully exploited
- The underlying vulnerability was first reported by researcher Johann Rehberger in October 2025; Anthropic acknowledged it but did not remediate before shipping Cowork
- Anthropic's mitigation guidance places responsibility on users to "avoid granting access to local files with sensitive information"

**Enterprise impact:** Any organization where employees connect Cowork to directories containing PII, financial records, source code, or trade secrets is exposed to silent data exfiltration. The attack is invisible to the user and bypasses the VM's network restrictions by using a trusted API endpoint.

### 3.2 LayerX: Zero-Click RCE via Desktop Extension MCP Chaining (February 2026)

LayerX researchers discovered that a single malicious Google Calendar event could trigger arbitrary code execution on systems running Claude Desktop Extensions, without any user interaction beyond a routine prompt like "take care of it." The vulnerability received a CVSS score of 10.0 and affects over 10,000 active Claude Desktop users and more than 50 extensions.

**Key findings:**

- Claude Desktop Extensions run unsandboxed with full OS privileges
- Claude autonomously chains low-risk connectors (e.g., Google Calendar) to high-risk local executors without user awareness
- A calendar event with embedded instructions can trigger code execution, SSH key theft, credential exfiltration, and system modification
- Anthropic declined to fix the vulnerability, stating it "falls outside our current threat model"
- The architecture lacks trust boundary enforcement between data sources and code executors

**Enterprise impact:** Any employee using Claude Desktop with Google Calendar integration is vulnerable to remote code execution from a calendar invite. In enterprise environments where calendars are shared across organizations, this creates an externally reachable attack vector that requires no phishing, no malware installation, and no user error beyond normal AI assistant usage.

### 3.3 Koi Research: Command Injection in Official Anthropic Extensions (February 2026)

Koi Research identified command injection vulnerabilities in three official Anthropic-published Desktop Extensions: the Chrome, iMessage, and Apple Notes connectors. These extensions, sitting at the top of Anthropic's extension marketplace with over 350,000 combined downloads, contained unsanitized input passed directly into AppleScript commands.

**Key findings:**

- All three vulnerabilities were rated CVSS 8.9 (High) by Anthropic
- A malicious website visited during normal browsing could trigger arbitrary code execution when a user asked Claude a question referencing web content
- SSH keys, AWS credentials, and browser passwords could be exfiltrated through a normal AI assistant interaction
- The vulnerabilities were basic command injection flaws (unsanitized string interpolation), raising questions about Anthropic's security review process for their own extensions
- Fixed in Claude Desktop version 0.1.9 (August 2025)

**Enterprise impact:** Organizations that deployed Claude Desktop with official extensions prior to the August 2025 patch were exposed to remote code execution from any website an employee visited. The fact that these were Anthropic's own extensions, not third-party, undermines the trust model of the extension marketplace.

### 3.4 Author's Research: CoworkVMService Access Control and Boot Media Vulnerabilities (March 2026)

The author's independent security research, submitted to Anthropic via HackerOne in March 2026, identified seven vulnerabilities in the Windows CoworkVMService that chain together to allow a standard, non-administrative user to control the SYSTEM service and achieve persistent code execution.

**Report 1: Named Pipe Access Control Failure Chain.** The CoworkVMService named pipe (`\\.\pipe\cowork-vm-service`) has a hardcoded security descriptor granting `FILE_ALL_ACCESS` to the Everyone well-known SID (SDDL: `O:SYD:(A;;FA;;;WD)`). The service authenticates clients by checking Authenticode signatures, which is trivially bypassable by copying the signed Claude.exe binary to a user-writable location and injecting shellcode into a suspended instance. Once authenticated, the attacker can create arbitrary VM sessions (`spawn`), mount arbitrary host filesystem paths (`mountPath`), kill other users' active Cowork sessions (`stopVM`), and reconfigure the SYSTEM service with attacker-controlled parameters (`configure`).

On multi-user systems (shared workstations, terminal servers, CI/CD runners), any local user can hijack any other user's Cowork session without admin privileges and without any trust dialog being presented to the victim.

**Report 2: SYSTEM Service Lifecycle Control and Boot Media Tampering.** A standard user can stop and start the CoworkVMService SYSTEM service. All VM boot media is stored in the user's `%APPDATA%` directory with Full Control permissions. The service performs no integrity verification on `rootfs.vhdx` before booting the VM. By stopping the service, patching `/etc/bash.bashrc` inside the rootfs.vhdx, and restarting, an attacker achieves persistent automatic code execution in every subsequent Cowork session. The payload writes files to the host filesystem through the Plan9 share, and persists indefinitely across service restarts and reboots.

**Enterprise impact:** On shared development machines, any user or malware can backdoor the Cowork environment for all users. The persistence mechanism survives service restarts and operates silently in the bash initialization phase. The lack of integrity verification on boot media makes this functionally equivalent to a rootkit implanted in a SYSTEM service's execution path.

## 4. Systemic Risk Analysis

### 4.1 Trust Model Assumptions vs. Enterprise Reality

Anthropic's security model for Claude Desktop and Cowork rests on several assumptions that do not hold in enterprise environments:

**Assumption 1: Single-user workstations.** Anthropic's responses to vulnerability reports consistently reference a threat model that assumes the local user is the only user. In enterprise environments, shared workstations, terminal servers, VDI pools, and CI/CD runners are common. The CoworkVMService pipe DACL, service control permissions, and boot media storage all fail in multi-user scenarios.

**Assumption 2: Users can detect prompt injection.** Anthropic's mitigation guidance advises users to "monitor Claude for suspicious actions that may indicate prompt injection." Security researcher Simon Willison noted that it is "not fair to tell regular non-programmer users to watch out for 'suspicious actions.'" In enterprise environments, Cowork is marketed to non-technical knowledge workers who cannot reasonably be expected to identify prompt injection attacks.

**Assumption 3: Users will restrict folder access.** Anthropic advises users to "avoid granting access to local files with sensitive information." Cowork's entire value proposition is organizing and analyzing desktop files. Telling users not to connect it to sensitive files contradicts the product's marketed purpose.

**Assumption 4: Extensions are sandboxed.** Claude Desktop Extensions run with full OS privileges, not in a sandboxed environment. Any vulnerability in any extension grants full system access. Anthropic's extension marketplace does not provide the same security review, permission model, or sandboxing that browser extension stores do.

### 4.2 Regulatory and Compliance Risks

Organizations subject to regulatory frameworks face specific risks from Claude Desktop and Cowork deployment:

**HIPAA (Healthcare):** Cowork's ability to access and process local files, combined with the proven file exfiltration vulnerability, creates a direct risk of unauthorized disclosure of PHI. A hidden prompt injection in a document received from any source can silently exfiltrate patient records through the Anthropic API.

**PCI DSS (Payment Card Industry):** The CoworkVMService running as SYSTEM with world-writable IPC, combined with the ability to mount arbitrary host paths into the VM, violates PCI DSS requirements for access control (Requirement 7), system hardening (Requirement 2), and monitoring (Requirement 10).

**SOX (Financial Reporting):** The boot media tampering vulnerability allows persistent, undetectable modification of the code execution environment. In financial institutions where Cowork might be used to analyze spreadsheets or generate reports, an attacker could silently alter the execution environment to produce manipulated outputs.

**GDPR (Data Protection):** The silent file exfiltration vulnerability constitutes unauthorized processing and transfer of personal data. The data flows through Anthropic's API to an attacker's account, potentially crossing jurisdictional boundaries without the data subject's knowledge or consent.

**CCPA / State Privacy Laws:** Similar to GDPR, the exfiltration of PII through prompt injection constitutes a data breach under most state privacy laws, triggering notification obligations.

### 4.3 Supply Chain Risk

Claude Desktop introduces multiple supply chain vectors:

1. **Extension marketplace:** Third-party and first-party extensions execute with full OS privileges. A compromised extension update affects all users who installed it.
2. **MCP server ecosystem:** Cowork connects to external MCP servers that can influence Claude's behavior. A compromised MCP server can inject malicious tool responses.
3. **Boot media distribution:** VM disk images are downloaded from Anthropic and stored in user-writable locations. No integrity verification on rootfs.vhdx means a supply chain compromise at the distribution level would be undetectable.
4. **Anthropic API dependency:** The PromptArmor exfiltration leverages the trusted API domain to bypass network restrictions, meaning the trust relationship with Anthropic's infrastructure is itself an attack vector.

### 4.4 Incident Response Challenges

Claude Desktop and Cowork create significant challenges for incident response:

- **Limited logging:** No integration with Windows Event Log, SIEM, or EDR telemetry. Prompt injection attacks leave no artifacts in standard security monitoring tools.
- **Ephemeral VM sessions:** Each Cowork task runs in a transient VM session with a randomly generated username. Correlating VM activity to specific user actions requires parsing application-specific logs.
- **Opaque tool chaining:** Claude autonomously selects and chains tools with no audit trail of which tools were invoked, what data was passed between them, or what decisions the model made.
- **Persistence detection:** Boot media tampering is undetectable by standard endpoint security tools because the modification is inside a virtual disk image, not a host filesystem binary.

## 5. Defensive Architecture

### 5.1 Pre-Deployment Controls

**Risk Acceptance and Governance.** Before deploying Claude Desktop or Cowork, organizations should conduct a formal risk assessment against their specific regulatory obligations, obtain explicit risk acceptance from the CISO (and compliance officer for regulated industries), document the accepted residual risk, establish a clear data classification policy defining which data categories are permitted in Cowork interactions, and define incident response procedures specific to AI agent compromise scenarios.

**Deployment Architecture.** Run Claude Desktop in disposable VDI containers or cloud development environments (AWS WorkSpaces, Azure Virtual Desktop, Citrix) rather than on developer endpoints. If VDI is not feasible, restrict Claude Desktop to single-user machines where the multi-user access control failures are not exploitable. Isolate machines running Claude Desktop on a dedicated network segment with restricted access to sensitive data stores.

**Extension Policy.** Disable all Desktop Extensions (DXT) unless specifically approved through a security review process. Maintain an allowlist of approved extensions. Block installation from the marketplace on managed devices via MDM policy. Monitor extension updates, as a previously safe extension can become dangerous after an update.

### 5.2 Runtime Controls

**File Access Restrictions.** Configure Cowork to access only dedicated working directories, never home directories, Desktop, or shared drives. Implement DLP rules that alert on sensitive file types being accessed by Claude.exe or cowork-svc.exe. Use Windows filesystem auditing (SACL) on directories containing PII or financial data to detect access by Cowork processes.

**Service Hardening (Windows).** Restrict the CoworkVMService DACL to prevent standard users from stopping or starting the service. Monitor for modifications to the VHDX bundle directory using File Integrity Monitoring (FIM). Implement application control (AppLocker, WDAC) to prevent execution of copied Claude.exe binaries outside the official installation path. Enable Code Integrity Guard policies to block unsigned code injection.

**Network Controls.** Monitor and log all traffic to `api.anthropic.com` with payload inspection where feasible. Implement anomaly detection for unusual data volumes in API traffic. Consider proxying Anthropic API traffic through an enterprise gateway for logging and DLP inspection. Block or alert on unexpected outbound connections from the Cowork VM.

**Credential Protection.** Remove persistent credentials from workstations running Claude Desktop (SSH keys, AWS credentials, API tokens). Use credential vaulting solutions with just-in-time access. Rotate credentials regularly on machines where Cowork has filesystem access.

### 5.3 Detection and Response

**Monitoring.** Forward CoworkVMService logs to SIEM for correlation. Create detection rules for service stop/start outside normal hours, VHDX modification timestamps changing, new Plan9 shares being created, and unauthorized RPC method calls. Monitor process creation for Claude.exe copies running from non-standard paths. Implement user behavior analytics for anomalous Cowork usage patterns.

**Incident Response Playbook.** Organizations should develop an AI agent-specific incident response playbook covering prompt injection detection, data exfiltration assessment (determining what data was accessed and potentially exfiltrated through the Anthropic API), boot media forensics (inspecting rootfs.vhdx for unauthorized modifications via hash comparison), session reconstruction (parsing CoworkVMService logs), and containment procedures for stopping the service and preserving logs while maintaining forensic integrity.

### 5.4 Ongoing Governance

Subscribe to Anthropic's security advisories and HackerOne disclosure notifications. Re-evaluate the risk assessment after each significant Claude Desktop update or new vulnerability disclosure. Conduct periodic penetration testing specifically targeting the Cowork attack surface. Maintain a baseline hash of rootfs.vhdx and verify it periodically. Review and update the extension allowlist quarterly.

## 6. Recommendations Summary

### For CISOs and Security Leaders

1. **Do not deploy Cowork on machines that store or process PII, PHI, financial data, or trade secrets** until Anthropic addresses the systemic prompt injection and file exfiltration vulnerabilities.
2. **Require VDI or containerized deployment** for any Cowork usage, isolating it from persistent user data and production networks.
3. **Disable Desktop Extensions entirely** unless a specific, security-reviewed extension is required for a documented business need.
4. **Implement the service hardening controls** described in Section 5.2 on all Windows machines running Claude Desktop.
5. **Establish AI-specific incident response procedures** before deployment, not after an incident.

### For IT and Infrastructure Teams

1. **Restrict the CoworkVMService DACL** and VHDX file permissions immediately on any existing installations.
2. **Deploy FIM** on the `%APPDATA%\Claude\vm_bundles\` directory to detect boot media tampering.
3. **Forward Cowork logs to SIEM** and create detection rules for the attack patterns described in this paper.
4. **Implement DLP rules** for Claude.exe and cowork-svc.exe file access patterns.
5. **Baseline the rootfs.vhdx hash** and verify it on a scheduled basis.

### For Executives Evaluating Adoption

1. **Understand that Cowork is a research preview**, not a production-ready enterprise tool. Anthropic's own documentation states it "can take potentially destructive actions."
2. **The productivity benefits are real but the security risks are material.** The file exfiltration vulnerability was known before Cowork shipped and Anthropic launched anyway.
3. **Anthropic's threat model assumes single-user, non-enterprise scenarios.** Their responses to vulnerability reports consistently defer to "the user chose to grant access" and "falls outside our current threat model."
4. **Budget for compensating controls.** The cost of VDI deployment, FIM, DLP integration, and security monitoring is part of the total cost of Cowork adoption, not optional.
5. **Consider the regulatory exposure.** A data breach through prompt injection may trigger notification obligations under HIPAA, GDPR, CCPA, and state privacy laws. The fact that the vulnerability was publicly known before your deployment weakens any reasonable security defense.

## 7. Conclusion

Claude Desktop and Cowork deliver compelling productivity capabilities, but their security architecture contains systemic deficiencies that create material risk for enterprise environments. The combination of a SYSTEM service with excessive access controls, unsandboxed extensions with full OS privileges, proven file exfiltration via prompt injection, and Anthropic's pattern of acknowledging but not remediating known vulnerabilities means that organizations deploying these tools must treat them as high-risk applications requiring significant compensating controls.

The vulnerabilities documented in this paper are not theoretical. Every finding discussed here has been demonstrated with working proof-of-concept code. The PromptArmor file exfiltration, the LayerX zero-click RCE, the Koi Research command injection, and the author's CoworkVMService access control failures all represent independently verified, reproducible attack paths that exist in production deployments today.

Organizations that choose to deploy Claude Desktop and Cowork should do so with full awareness of these risks, with the compensating controls described in this paper in place, and with clear risk acceptance from security and compliance leadership.

## References

1. PromptArmor. "Claude Cowork Exfiltrates Files." January 15, 2026. [promptarmor.com](https://www.promptarmor.com/resources/claude-cowork-exfiltrates-files)
2. LayerX. "Claude Desktop Extensions Exposes Over 10,000 Users to Remote Code Execution Vulnerability." February 9, 2026. [layerxsecurity.com](https://layerxsecurity.com/blog/claude-desktop-extensions-rce/)
3. Koi Research. "PromptJacking: The Critical RCEs in Claude Desktop That Turn Questions Into Exploits." February 2026. [koi.ai](https://www.koi.ai/blog/promptjacking-the-critical-rce-in-claude-desktop-that-turn-questions-into-exploits)
4. Anthropic. "Claude Cowork Research Preview." January 2026. [claude.ai](https://claude.ai)
5. Sany, J. "CoworkVMService Named Pipe Access Control Failure." HackerOne Report, March 2026.
6. Sany, J. "CoworkVMService SYSTEM Service Lifecycle Control and Boot Media Tampering." HackerOne Report, March 2026.
7. Rehberger, J. "Files API Exfiltration in Claude Code." HackerOne Report, October 2025.
8. Sany, J. "Trust Boundary Failures in AI Coding Agents: Empirical Analysis of MCP Configuration Attacks in Claude Code." Zenodo, 2026. DOI: [10.5281/zenodo.19011781](https://doi.org/10.5281/zenodo.19011781)
9. Willison, S. "Claude Cowork for Non-Programmers." Datasette Blog, January 2026.
10. Infosecurity Magazine. "New Zero-Click Flaw in Claude Extensions, Anthropic Declines Fix." March 2026.
