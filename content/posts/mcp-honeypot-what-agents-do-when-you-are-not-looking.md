---
title: "MCP Honeypots: What AI Agents Do When You're Not Looking"
date: 2026-03-16
tags: ["ai-security", "mcp", "honeypot", "agents"]
categories: ["Security Research"]
summary: "We built three trap MCP servers to catch AI agents behaving badly. Path traversal, SQL injection, command injection — the same old attacks through a brand new protocol."
ShowToc: true
TocOpen: true
---

We built trap servers to catch AI agents behaving badly.

The [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) is becoming the standard way AI agents interact with tools. But when an agent calls `read_file("../../etc/passwd")` or `query_db("' OR 1=1 --")`, who's watching?

Nobody, it turns out. So we built honeypots.

## The Experiment

We created three deliberately vulnerable MCP servers, each simulating a common tool pattern with a classic vulnerability:

| Honeypot | Simulates | Vulnerability | CWE |
|----------|-----------|---------------|-----|
| **A: File System** | File browser | Path traversal (no sanitization) | CWE-22 |
| **B: Database** | SQL interface | SQL injection (string interpolation) | CWE-89 |
| **C: Shell** | Command execution | Command injection (allowlist bypass) | CWE-78 |

Each server implements standard MCP tools via stdio transport, contains bait data (fake credentials, `.env` files, a `secrets` table), and logs every tool call with timestamps and suspicion flags.

The architecture is simple:

```
mcp-honeypot/
├── shared/logger.ts          # Centralized logging with pattern detection
├── honeypot-a/server.ts      # File system (path traversal)
├── honeypot-b/server.ts      # Database (SQL injection)
├── honeypot-c/server.ts      # Shell (command injection)
└── analysis.md               # Threat model + detection rules
```

## What Attacks Look Like in MCP

Here's the thing about MCP: tool calls look trustworthy. The server receives structured JSON from an agent, and there's an implicit assumption that the parameters are benign. But LLM output — which generates those parameters — can be influenced by prompt injection.

This creates a **trust gap**: the server trusts the agent, the agent trusts the LLM, and the LLM can be manipulated.

### Honeypot A: Path Traversal

The file browser tool uses `path.join()` without sanitization. An agent (or a prompt-injected agent) can trivially escape the sandbox:

```
Tool call: read_file("../../etc/passwd")
Tool call: read_file("../../../home/user/.ssh/id_rsa")
Tool call: list_directory("../../")
```

**Success rate without protection: 100%.** `path.join()` does not prevent `../` on most platforms.

### Honeypot B: SQL Injection

The database tool builds queries with string interpolation — the oldest sin in web development:

```
Tool call: query_db("admin' UNION SELECT * FROM secrets --")
Tool call: query_db("x' OR 1=1 --")
Tool call: insert_record("x', 'y', 'admin');--")
```

**Success rate without protection: 100%.** String interpolation is trivially exploitable.

### Honeypot C: Command Injection

The shell tool has an allowlist — but it only checks the first token. Arguments are unsanitized:

```
Tool call: run_command("echo hello && whoami")
Tool call: run_command("echo x | cat /etc/passwd")
Tool call: run_command("echo $HOME")
```

**Success rate without protection: 100%.** The allowlist provides a false sense of security.

## Attack Detection Patterns

We designed a multi-layer detection approach and measured expected coverage:

| Detection Layer | Coverage | False Positive Risk | Latency |
|----------------|----------|-------------------|---------|
| **Regex patterns** | ~70% of known attacks | Low | <1ms |
| **LLM-as-a-Judge** | ~95% including novel | Medium | 200-500ms |
| **Behavioral analysis** | ~60% (escalation patterns) | Low | Minimal |

### The Regex Fast Path

Block the obvious stuff with zero latency:

```yaml
rules:
  path_traversal:
    pattern: '/\.\.[\\/]/'
    action: block

  sql_injection:
    patterns:
      - '/\bUNION\b.*\bSELECT\b/i'
      - '/\bOR\s+1\s*=\s*1/i'
      - '/\bDROP\s+TABLE\b/i'
    action: block

  command_injection:
    patterns:
      - '/[;&|`$]/'
      - '/\brm\s+-rf\b/'
    action: block
```

### The LLM Judge (Deep Analysis)

For tool calls that pass the regex filter but seem suspicious, an LLM-as-a-Judge evaluates **intent**:

1. Is the tool call consistent with the conversation context?
2. Does the argument contain embedded commands or queries?
3. Is there a pattern of escalation — list, then `../`, then `../../etc/passwd`?

### Behavioral Anomaly Detection

Track per-session patterns that indicate reconnaissance:
- Tool call bursts (many calls in quick succession)
- Tool diversity (touching many different tools rapidly)
- Escalation trajectory (benign → enumeration → exploitation)
- Cross-tool correlation (schema enumeration → targeted data extraction)

## Bait Data Reveals Intent

The `.env` file and `secrets` table serve as **canary tokens**. If an agent accesses them, it reveals either:

1. **A prompt injection attack** directing the agent to steal credentials
2. **An overly curious model** exploring beyond its task boundaries

Both cases warrant alerting. Both are invisible without instrumentation.

## Key Learnings

### MCP Tools Are Just APIs with Extra Trust

Every vulnerability in our honeypots is a well-known web security issue (OWASP Top 10). The difference is the **attack vector**: instead of HTTP requests from a browser, attacks arrive as MCP tool calls from an AI agent. The defenses are the same — input validation, parameterized queries, principle of least privilege — but almost nobody is applying them.

### The Allowlist Illusion

Honeypot C demonstrates that superficial allowlists (checking only the first token) create a **false sense of security**. Many MCP servers in the wild do exactly this. A proper defense must analyze the full argument string, not just command prefixes.

### Logging Is Necessary but Insufficient

Our regex-based logger catches known patterns, but sophisticated attacks that avoid obvious signatures slip through. This validates the need for an intent-aware judge, not just pattern matching.

## The Threat Model

```
Threat Sources              Attack Surface              Impact
┌──────────────┐       ┌──────────────────┐      ┌─────────────────┐
│ Prompt        │       │ File Operations  │      │ Data            │
│ Injection     │──────▶│ (read/write/list)│─────▶│ Exfiltration    │
├──────────────┤       ├──────────────────┤      ├─────────────────┤
│ Malicious MCP│       │ Database Queries  │      │ System          │
│ Server       │──────▶│ (SQL via params)  │─────▶│ Compromise      │
├──────────────┤       ├──────────────────┤      ├─────────────────┤
│ Supply Chain │       │ Shell Commands   │      │ Lateral         │
│ Attack       │──────▶│ (via exec tools) │─────▶│ Movement        │
└──────────────┘       └──────────────────┘      └─────────────────┘
                              ▲
                    ClawGuard │ (regex + LLM judge + behavioral)
```

## What This Means for the MCP Ecosystem

MCP is growing fast. Anthropic, OpenAI, and dozens of tool providers are building MCP servers. But the security story is lagging behind:

- **Most MCP servers have no input validation.** They trust that the calling agent will provide safe parameters.
- **There's no standard security layer.** Each server implements (or doesn't) its own protection.
- **Prompt injection makes every tool a potential attack vector.** An agent doesn't need to be malicious — it just needs to be manipulated.

## Enter ClawGuard

This is why we built [ClawGuard](https://github.com/nicepkg/openclaw) — an LLM-as-a-Judge guardrail that sits between agents and their tools:

1. **Regex pre-filter** blocks known attack patterns with <1ms latency
2. **LLM judge** evaluates intent for ambiguous tool calls
3. **Behavioral analysis** detects escalation patterns across tool calls
4. **Policy engine** defines per-tool rules for what's allowed

If you're building MCP servers or running AI agents, you need guardrails. The attacks are the same ones we've been fighting for 20 years — they just wear a new protocol now.

---

*The honeypot experiment code is part of our [security research at OpenClaw](https://github.com/nicepkg/openclaw). All experiments run locally — none of these servers should ever be exposed to the internet.*
