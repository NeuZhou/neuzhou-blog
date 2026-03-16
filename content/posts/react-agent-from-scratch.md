---
title: "I Built a ReAct Agent in 200 Lines of Python (No Frameworks)"
date: 2026-03-16
tags: ["agents", "python", "react", "from-scratch"]
categories: ["AI Engineering"]
summary: "No LangChain. No frameworks. Just Python, Ollama, and regex — to understand what AI agents actually are under the hood."
ShowToc: true
TocOpen: true
---

I wanted to understand how AI agents actually work. Not the LangChain abstraction. Not the marketing pitch. The actual loop.

So I built one from scratch in ~200 lines of Python. It uses Ollama for inference, has 4 tools, and taught me more about agents in a weekend than months of reading docs.

## Why Build from Scratch?

Understanding beats convenience.

Frameworks like LangChain are thousands of lines of abstraction over a concept that's surprisingly simple. When something breaks — and it will — you need to understand what's happening underneath. Building from scratch strips away the magic.

The goal wasn't to build a production agent. It was to answer: **what is the minimum viable agent?**

## The ReAct Loop Explained

[ReAct](https://arxiv.org/abs/2210.03629) (Reason + Act) is an elegant pattern. The LLM alternates between thinking and doing:

```
Question → Thought → Action → Observation → Thought → Action → ... → Final Answer
```

In code, it's a loop:

```python
for i in range(MAX_ITERATIONS):
    response = call_llm(conversation)      # Think
    parsed = parse_response(response)       # What did it decide?

    if parsed[0] == "final":
        return parsed[1]                    # Done!

    elif parsed[0] == "action":
        observation = execute_tool(...)     # Do the thing
        conversation += observation         # Tell it what happened

    else:
        conversation += "Please follow the format."  # Nudge
```

That's it. The core loop is about 30 lines. The LLM does the hard work — reasoning about what tool to use and what to do with the result.

## The 4 Tools

I kept the tool set minimal:

```python
TOOLS = {
    "search_knowledge": search_knowledge,  # Query local knowledge base
    "calculate": calculate,                # Safe math evaluation
    "read_file": read_file,                # Read local files
    "web_fetch": web_fetch,                # Fetch URL content
}
```

Tool dispatch is trivial — a dictionary mapping names to functions:

```python
def execute_tool(tool_name: str, tool_input: str) -> str:
    if tool_name not in TOOLS:
        return f"Error: Unknown tool '{tool_name}'"
    return TOOLS[tool_name](tool_input)
```

The LLM gets a description of each tool in its system prompt, then outputs which tool to call and with what input. The format is rigid:

```
Thought: I need to calculate compound interest.
Action: calculate
Action Input: 10000 * (1 + 0.05) ** 10 - 10000
```

The agent parses this with regex, calls the function, and feeds the result back:

```
Observation: 6288.946267774414
```

Then the LLM continues reasoning with the real data.

## The Hallucinated Observation Problem

Here's the most interesting bug I encountered: **the LLM generates fake observations.**

Instead of stopping after `Action Input:` and waiting for the real tool output, the model sometimes keeps going:

```
Thought: I need to search for MCP security threats.
Action: search_knowledge
Action Input: MCP security threats
Observation: The main threats are prompt injection, tool misuse...  ← HALLUCINATED
Thought: Based on this information...
Final Answer: ...
```

That "Observation" is completely made up. The LLM never waited for the real tool to execute — it imagined what the result would be and continued reasoning on fabricated data.

This is a fundamental problem with text-based agent protocols. The LLM doesn't know the difference between text it should generate and text it should wait for. In our experiments with `qwen2.5:7b`, this happened in about 1 in 5 multi-step queries.

**The fix:** Parse strictly. Stop at `Action Input:`, truncate everything after it, execute the tool, then append the real observation. Production frameworks solve this with structured output (JSON mode) or native function calling — but it's worth understanding *why* they need to.

## What Worked Immediately

- **The math tool.** The LLM correctly formulated `10000 * (1 + 0.05) ** 10 - 10000` for a compound interest question on the first try. Simple, deterministic tools are agent-friendly.
- **Tool dispatch.** A dict mapping names to functions. No classes, no schemas, no registration. It just works.
- **Ollama's API.** POST JSON, get JSON back. No auth, no SDKs, no ceremony.

## What Was Harder Than Expected

### Parsing Is Fragile

Regex-based parsing of LLM output breaks constantly:

```python
action_m = re.search(r"Action:\s*(\w+)", text)
input_m = re.search(r"Action Input:\s*(.+?)(?:\n|$)", text, re.DOTALL)
```

This works when the LLM follows format perfectly. It breaks when:
- The LLM puts Action and Action Input on the same line
- Extra text appears between them
- The LLM uses slightly different labels ("Tool:" instead of "Action:")
- Unicode or special characters appear in the output

### Prompt Engineering Is Everything

Small changes to the system prompt dramatically affect behavior:

| Change | Effect |
|--------|--------|
| Added "Use exactly one Action per step" | Reduced multi-action outputs |
| Added explicit format examples | Critical — without them, LLM invents own format |
| Temperature 0.1 → 0.7 | Format compliance dropped significantly |
| Added nudge mechanism | Recovers from parse errors, but wastes iterations |

### Knowing When to Stop

The LLM sometimes generates "Final Answer" prematurely — after one tool call when it needs three. Other times it loops forever, retrying the same failing tool. The `MAX_ITERATIONS = 10` ceiling is a crude but necessary safety net.

## The Gap: Toy → Production

Here's the key insight from building this: **the ReAct logic is maybe 5% of a production agent. The other 95% is everything else.**

| Aspect | mini_agent.py | Production Agent |
|--------|--------------|-----------------|
| Parsing | Regex (fragile) | Structured output / function calling |
| Memory | None | Short-term + long-term + cross-session |
| Error handling | Basic nudge | Retry chains, fallbacks, circuit breakers |
| Safety | Minimal | Input validation, approval flows, sandboxing |
| Observability | Print statements | Structured logging, tracing, token tracking |
| Tool integration | 4 hardcoded tools | Dynamic tool discovery, schemas, auth |

Production agent = 5% logic + 95% reliability engineering.

The actual agent loop? Trivial. Making it work reliably across thousands of queries with diverse tools, unpredictable models, and real users? That's distributed systems, security engineering, and UX design — all at once.

## Code Walkthrough: The Interesting Parts

### The System Prompt

The system prompt is the most important code in the file:

```python
SYSTEM_PROMPT = f"""You are a helpful AI assistant that uses a ReAct loop.

{TOOL_DESCRIPTIONS}

For each step, output EXACTLY this format:

Thought: <your reasoning>
Action: <tool_name>
Action Input: <input>

When you have enough information:

Thought: I now have enough information to answer.
Final Answer: <your complete answer>

Rules:
- Always start with a Thought
- Use exactly one Action per step
- Wait for the Observation before continuing
"""
```

Every sentence is load-bearing. Remove "Use exactly one Action per step" and the LLM starts chaining multiple actions in one response. Remove the explicit format and it invents its own.

### Safe Math Evaluation

The `calculate` tool uses `eval()` — dangerous by default, but sandboxed:

```python
def calculate(expression: str) -> str:
    allowed = set("0123456789+-*/.() %eE")
    cleaned = expression.strip()
    if not all(c in allowed for c in cleaned):
        return f"Error: unsafe expression '{cleaned}'"
    result = eval(cleaned, {"__builtins__": {}}, {})
    return str(result)
```

Character-level filtering + empty builtins. Not bulletproof (a production version should use AST parsing), but good enough for a learning exercise.

### The Nudge Mechanism

When the LLM doesn't follow format, instead of crashing, we nudge it:

```python
else:  # parse error
    conversation += (
        f"{response}\n\n"
        "You must respond with either:\n"
        "Action: <tool_name>\\nAction Input: <input>\n"
        "OR\n"
        "Final Answer: <your answer>\n"
    )
```

This works surprisingly well — about 80% of the time, the LLM self-corrects on the next iteration. The other 20%, it's lost and will hit `MAX_ITERATIONS`.

## What I'd Do Differently

1. **Use JSON mode** from the start. Regex parsing is a dead end.
2. **Add observation truncation.** Long tool outputs blow the context window silently.
3. **Track token count.** I had no idea how much context I was using until things started breaking.
4. **Test with multiple models.** `qwen2.5:7b` has its quirks. GPT-4 and Claude behave very differently in the same loop.
5. **Add streaming.** Watching the agent think in real-time is both useful for debugging and satisfying to watch.

## Try It Yourself

The entire agent is a single Python file. Requirements: Python 3.10+, Ollama running locally, and `requests`.

```bash
# Pull a model
ollama pull qwen2.5:7b

# Run the agent
python mini_agent.py
```

It'll run through three test queries: a knowledge base search, a math calculation, and a file summary. Watch the iterations — that's where the learning happens.

---

*Building a toy agent is the best way to appreciate a real one. The loop is simple. Making it reliable is the life's work.*
