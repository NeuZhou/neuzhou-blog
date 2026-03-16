---
title: "repo2skill: Convert Any GitHub Repo into an AI Agent Skill in One Command"
date: 2026-03-16
tags: ["open-source", "ai-tools", "agents", "developer-tools"]
categories: ["Announcements"]
summary: "repo2skill analyzes any GitHub repository and generates a ready-to-use AI agent skill — no LLM needed, 13 languages supported, quality-scored output."
ShowToc: true
TocOpen: true
---

The AI agent ecosystem has a bottleneck: **skills are hard to create**. Every useful tool, library, or API needs a hand-crafted SKILL.md before an agent can use it effectively. That's thousands of hours of manual documentation work standing between agents and the tools they need.

**repo2skill** eliminates that bottleneck entirely.

```bash
npm install -g repo2skill
repo2skill https://github.com/expressjs/express
```

That's it. One command. Out comes a complete, quality-scored AI agent skill.

## The Problem: Skill Creation Doesn't Scale

AI agents are only as capable as their skills. An agent without skills is just a chatbot with opinions. But creating skills today means:

1. Reading through an entire codebase
2. Understanding the API surface, CLI flags, and configuration
3. Writing structured documentation an agent can parse
4. Testing that the agent actually uses it correctly
5. Maintaining it as the upstream project evolves

This is **artisanal work** in an era that needs **industrial output**. There are 400+ million repositories on GitHub. We need skills for thousands of them. Manual creation will never keep up.

## How repo2skill Works

repo2skill uses **heuristic analysis** — no LLM calls, no API keys, no cloud dependency. It:

1. **Clones** the repository (or reads a local path)
2. **Detects** the language, framework, and project type
3. **Analyzes** the structure: entry points, exports, CLI commands, configuration files
4. **Extracts** documentation from README, docstrings, comments, and type signatures
5. **Generates** a SKILL.md following the AgentSkill specification
6. **Scores** the output quality so you know how much manual polish is needed

The entire process runs locally in seconds. No tokens burned. No rate limits hit.

## 13 Language Support

repo2skill understands projects written in:

| Language | Detection | Notes |
|----------|-----------|-------|
| JavaScript | package.json, .js | npm scripts, exports |
| TypeScript | tsconfig.json, .ts | Type-aware extraction |
| Python | setup.py, pyproject.toml | Entry points, CLI |
| Rust | Cargo.toml | Binary/lib detection |
| Go | go.mod | Command packages |
| Java | pom.xml, build.gradle | Maven/Gradle |
| C# | .csproj, .sln | .NET projects |
| Ruby | Gemfile, .gemspec | Gem structure |
| PHP | composer.json | Autoload mapping |
| Swift | Package.swift | SPM packages |
| Kotlin | build.gradle.kts | Android/JVM |
| C/C++ | CMakeLists.txt, Makefile | Build system detection |
| Shell | bin/, scripts/ | Script discovery |

## Live Demo: 3 Popular Repos

Let's see what repo2skill produces for real-world projects:

### 1. Express.js

```bash
$ repo2skill https://github.com/expressjs/express

✔ Cloned expressjs/express
✔ Detected: JavaScript (Node.js web framework)
✔ Analyzed 187 files, 34 exports
✔ Generated SKILL.md (Quality: 87/100)

Output: ./express-skill/SKILL.md
```

The generated skill includes route creation patterns, middleware usage, common configurations, and CLI commands — everything an agent needs to scaffold and configure Express apps.

### 2. FastAPI

```bash
$ repo2skill https://github.com/tiangolo/fastapi

✔ Cloned tiangolo/fastapi
✔ Detected: Python (ASGI web framework)
✔ Analyzed 243 files, 56 exports
✔ Generated SKILL.md (Quality: 91/100)

Output: ./fastapi-skill/SKILL.md
```

High quality score here — FastAPI has excellent type hints and docstrings, which repo2skill leverages heavily.

### 3. Tokio (Rust)

```bash
$ repo2skill https://github.com/tokio-rs/tokio

✔ Cloned tokio-rs/tokio
✔ Detected: Rust (async runtime)
✔ Analyzed 412 files, 89 public items
✔ Generated SKILL.md (Quality: 78/100)

Output: ./tokio-skill/SKILL.md
```

Lower score for Tokio — complex Rust crates with many feature flags need more manual refinement. But 78/100 is still a solid starting point that saves hours of work.

## The Quality Scoring System

Every generated skill gets a score from 0-100 based on:

| Factor | Weight | What It Measures |
|--------|--------|-----------------|
| **Documentation coverage** | 30% | README quality, inline docs, examples |
| **API surface clarity** | 25% | Export structure, type signatures |
| **Example availability** | 20% | Code examples, tests as usage patterns |
| **Configuration detection** | 15% | Config files, env vars, CLI flags |
| **Structure consistency** | 10% | Project organization, naming conventions |

**Score interpretation:**

- **90-100**: Ready to use as-is
- **75-89**: Minor edits needed (add examples, clarify edge cases)
- **50-74**: Good skeleton, needs human review
- **Below 50**: Complex project, use as starting point only

## Get Started

```bash
# Install globally
npm install -g repo2skill

# Convert a GitHub repo
repo2skill https://github.com/user/repo

# Convert a local project
repo2skill ./my-project

# Output to specific directory
repo2skill https://github.com/user/repo -o ./skills/
```

## Why This Matters

The agent ecosystem is growing fast. OpenClaw alone has dozens of community skills, and every week people ask for more. But skill creation has been the bottleneck — it requires deep understanding of both the target project and the agent skill format.

repo2skill turns this from a **days-long manual process** into a **seconds-long automated one**. It won't replace human expertise for complex skills, but it gives you an 80% head start every time.

## Contribute

repo2skill is fully open source. We'd love contributions for:

- **New language analyzers** — especially for Elixir, Scala, and Dart
- **Framework-specific heuristics** — Django, Rails, Spring Boot
- **Quality scoring improvements** — better calibration across project types
- **Output format options** — different skill spec versions

⭐ **Star the repo**: [github.com/nicepkg/repo2skill](https://github.com/nicepkg/repo2skill)

📦 **Install now**: `npm install -g repo2skill`

🐛 **Report issues**: [github.com/nicepkg/repo2skill/issues](https://github.com/nicepkg/repo2skill/issues)

---

*repo2skill is part of the [OpenClaw](https://github.com/nicepkg/openclaw) ecosystem — building the tools that make AI agents actually useful.*
