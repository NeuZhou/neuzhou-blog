---
title: "Why Your AI Agent Needs a Test Suite (and How to Build One)"
date: 2026-03-16
description: "Eval tools test outputs. But who tests behavior? Introducing AgentProbe — record, replay, and assert agent workflows like Playwright does for web apps."
tags: ["ai-agents", "testing", "agentprobe", "developer-tools", "open-source"]
categories: ["AI Engineering"]
author: "Kang Zhou"
draft: false
---

## The Testing Gap in AI Agents

You build an agent. It calls tools, searches the web, writes code, orchestrates sub-agents. You demo it. It works beautifully.

You ship it. It breaks in ways you never imagined.

Maybe it called the wrong tool. Maybe it called the right tool with garbage parameters. Maybe it skipped a critical step entirely. The final output *looked* fine in your eval — but the behavior was wrong.

This is the testing gap in AI agents: **we evaluate outputs but don't test behavior**.

## Why Eval ≠ Testing

Eval frameworks (promptfoo, DeepEval, Braintrust) answer: *"Did the agent produce a good response?"*

That's valuable. But it's like testing a web app by taking screenshots. You'd never ship a checkout flow verified only by "the page looks right." You'd test that clicking "Buy" triggers the payment API, updates inventory, and sends a confirmation email.

Agents are the same. The interesting bugs aren't in what they *say* — they're in what they *do*:

| Eval catches | Eval misses |
|---|---|
| Wrong final answer | Called deprecated API |
| Poor response quality | Skipped validation step |
| Hallucinated facts | Wrong tool call order |
| Format violations | Missing required parameters |

Eval is **necessary**. But it's not **sufficient**.

## AgentProbe's Approach

AgentProbe brings three ideas from traditional software testing to the agent world:

### 1. Trace Recording

Instrument your agent (any framework) and record full execution traces — every tool call, every parameter, every response. Think of it as `console.log` for agent behavior, structured as OpenTelemetry-compatible spans.

```python
from agentprobe import Recorder

recorder = Recorder()
with recorder.session("booking-flow"):
    agent.run("Book a flight from SFO to NRT for March 20")

recorder.save("traces/booking-flow.json")
```

### 2. Behavioral Assertions in YAML

Define what *should* happen — not what the output should look like, but what tools should be called, in what order, with what constraints:

```yaml
name: booking-flow
assertions:
  - assert: tool_called
    name: search_flights
    before: book_flight

  - assert: parameter_matches
    tool: search_flights
    param: origin
    value: "SFO"

  - assert: parameter_matches
    tool: search_flights
    param: destination
    value: "NRT"

  - assert: tool_called
    name: confirm_booking
    after: book_flight

  - assert: tool_not_called
    name: cancel_booking
```

Readable. Declarative. Version-controllable.

### 3. Deterministic Replay

Recorded traces replay without LLM calls. Your CI pipeline runs in seconds, costs nothing, and produces the same result every time. When you change your agent's prompts or tool definitions, replay existing traces to catch regressions instantly.

```bash
# Record once
agentprobe record --agent my_agent --suite booking

# Test forever (no LLM needed)
agentprobe test --suite booking
# ✓ search_flights called before book_flight
# ✓ origin parameter = "SFO"
# ✓ confirm_booking called after book_flight
# ✗ cancel_booking was not expected but was called
```

## Architecture Overview

```
┌─────────────┐     ┌──────────────┐     ┌───────────────┐
│  Your Agent  │────▶│   Recorder   │────▶│  Trace Store  │
│ (any framework)    │  (OTel spans)│     │  (.json files) │
└─────────────┘     └──────────────┘     └───────┬───────┘
                                                  │
                    ┌──────────────┐     ┌────────▼────────┐
                    │  Assertions  │◀────│  Replay Engine  │
                    │  (.yaml)     │     │  (deterministic)│
                    └──────┬───────┘     └─────────────────┘
                           │
                    ┌──────▼───────┐
                    │  Test Report │
                    │  (pytest)    │
                    └──────────────┘
```

- **Framework-agnostic**: Works with LangChain, OpenAI Agents SDK, CrewAI, or raw function-calling loops
- **pytest integration**: Runs with your existing test infrastructure
- **~2k lines of Python**: Small, auditable, no heavy dependencies

## Getting Started

```bash
pip install agentprobe

# Initialize a test suite
agentprobe init --name my-agent-tests

# Record a trace
agentprobe record --agent my_agent.py --input "Search for recent AI papers"

# Write assertions
# (edit tests/my-agent-tests/assertions.yaml)

# Run tests
pytest --agentprobe
```

## When Should You Use This?

AgentProbe makes sense when your agent:

- **Calls external tools** (APIs, databases, file systems)
- **Has multi-step workflows** where order matters
- **Runs in production** where silent behavioral changes cost money or trust
- **Evolves frequently** — prompt changes, tool changes, model upgrades

It's less useful for simple single-turn Q&A (eval tools handle that fine).

## What's Next

This is an early release. The core works, but I'm actively looking for feedback on:

- What assertions are missing?
- What frameworks need better integration?
- What's confusing in the API?

The goal: make agent testing as natural as `pytest` made Python testing.

**GitHub:** [github.com/NeuZhou/agentprobe](https://github.com/NeuZhou/agentprobe)

Star it, try it, break it, tell me what's wrong. That's how good tools get built.
