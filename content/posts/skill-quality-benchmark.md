---
title: "We Analyzed 31 GitHub Repos to Find What Makes a Great AI Skill"
date: 2026-03-16
tags: ["ai-tools", "agents", "open-source", "developer-experience"]
categories: ["Tools & Research"]
summary: "We ran repo2skill against 31 popular GitHub repos across 7 categories. Here's what separates the best from the rest — and what you can do about it."
ShowToc: true
TocOpen: true
---

What if any GitHub repo could become an AI agent skill with a single command? That's what [repo2skill](https://github.com/nicepkg/openclaw) does — and we wanted to know: which repos actually produce *good* skills?

We analyzed 31 popular open-source repos to find out.

## What is repo2skill?

`repo2skill` is a tool that converts a GitHub repository into an [AgentSkill](https://github.com/nicepkg/openclaw) — a structured package that AI agents can discover and use. It reads the repo's README, extracts key information (description, install instructions, usage examples, CLI commands), and produces a `SKILL.md` file.

The quality of the output depends entirely on how well the repo's README is structured. Which raises the question: **what makes a repo "skill-friendly"?**

## Methodology

We selected 31 popular npm-ecosystem repos across 7 categories:

| Category | Repos | Examples |
|----------|-------|---------|
| Web Frameworks | 5 | Express, Fastify, Nest, Koa, Hono |
| Databases | 5 | TypeORM, Knex, Drizzle, Prisma, Sequelize |
| CLI Tools | 5 | Commander, Chalk, Ora, Inquirer, Yargs |
| Utilities | 5 | Zod, Day.js, Lodash, Ajv, Joi |
| Build Tools | 5 | Webpack, esbuild, Vite, Rollup, Turbo |
| Testing | 2 | Jest, Vitest |
| AI/ML | 4 | Ollama-js, Transformers.js, LangChain, Vercel AI |

Each repo was scored on 12 fields that matter for skill quality: description, rich description, when to use, when not to use, trigger phrases, CLI commands, install instructions, usage section, usage examples, features, API section, and config section.

## Key Findings

### The Numbers

| Metric | Value |
|--------|-------|
| Average quality score | **52%** |
| Highest score | **68%** (Fastify, TypeORM) |
| Lowest score | **17%** (Vercel AI SDK) |
| Repos scoring ≥60% | **8 of 31** |
| Repos scoring <50% | **10 of 31** |

**No repo scored above 70%.** Even the best repos leave significant room for improvement in skill generation.

### Top 10

| Rank | Repo | Category | Score |
|------|------|----------|-------|
| 1 | fastify/fastify | Web Frameworks | 68% |
| 2 | typeorm/typeorm | Databases | 68% |
| 3 | jestjs/jest | Testing | 67% |
| 4 | expressjs/express | Web Frameworks | 65% |
| 5 | webpack/webpack | Build Tools | 65% |
| 6 | iamkun/dayjs | Utilities | 63% |
| 7 | tj/commander.js | CLI Tools | 61% |
| 8 | chalk/chalk | CLI Tools | 60% |
| 9 | sindresorhus/ora | CLI Tools | 58% |
| 10 | knex/knex | Databases | 58% |

### The Monorepo Penalty

One of the most striking findings:

- **Monorepo average**: 47%
- **Single-repo average**: 56%

13 of our 31 repos were monorepos. They consistently scored lower because documentation is spread across packages. The root README often just says "see packages/" — which gives repo2skill almost nothing to work with.

### The Missing Fields

| Field | Fill Rate | Notes |
|-------|-----------|-------|
| whenToUse | 100% | Always extractable |
| description | 97% | Almost always present |
| installInstructions | 65% | Frequently missing |
| usageExamples | 42% | **Big gap** |
| features | 26% | Often buried in docs sites |
| cliCommands | 16% | Only relevant for CLI tools |
| apiSection | 6% | Almost never in README |
| configSection | 6% | Almost never in README |

The biggest opportunity? **Usage examples.** Less than half of repos had extractable code examples in their README. This is the single highest-impact improvement most repos could make.

### Category Rankings

| Category | Avg Score |
|----------|-----------|
| Web Frameworks | 56% |
| Testing | 56% |
| CLI Tools | 55% |
| Utilities | 53% |
| Databases | 52% |
| Build Tools | 51% |
| AI/ML | **43%** |

AI/ML repos scored the lowest. The Vercel AI SDK scored just 17% — its README is essentially a redirect to a docs site. Compare that to Fastify at 68%, which has comprehensive inline documentation.

## The Quality Formula

Based on our analysis, here's what makes a repo produce a great skill:

### Must-Haves (High Impact)

1. **Clear first paragraph** — repo2skill extracts the description from here. Make it count.
2. **Install section** — Labeled "Installation" or "Getting Started" with copy-paste commands.
3. **Usage section with code** — At least 2-3 real code examples. This becomes the skill's usage guidance.
4. **Alternatives mentioned** — Discussing when to use (and *not* use) your tool helps agents route correctly.

### Nice-to-Haves (Medium Impact)

5. **CLI documentation** — If your tool has a CLI, document the commands.
6. **API summary** — Even a brief overview of key methods enriches the skill.
7. **Features list** — Bullet points that help identify trigger phrases.
8. **Configuration section** — Labeled options/config for extraction.

### Anti-Patterns

- **"See docs at X"** — Redirecting to external docs gives repo2skill nothing
- **Badge walls** — Rows of shields.io badges before the description confuse extraction
- **Thin monorepo roots** — If your monorepo README is 3 lines, the skill will be 3 lines
- **Non-standard headers** — Stick to "Installation", "Usage", "API", "Configuration"

## What Makes a Repo "Skill-Friendly"

Think of your README as documentation for both humans **and** machines. The repos that scored highest share these traits:

1. **Self-contained README** — Key information is *in* the README, not behind a link
2. **Structured sections** — Standard headers that tools can parse
3. **Real code examples** — Not pseudo-code, not "see examples/ directory"
4. **Clear positioning** — "Use this when X, use Y instead when Z"
5. **Concise but complete** — The Goldilocks zone between a one-liner and a novel

## Try It Yourself

```bash
npx repo2skill https://github.com/your/repo
```

repo2skill is part of the [OpenClaw](https://github.com/nicepkg/openclaw) ecosystem. It's open source, and we'd love to see the community improve it.

**For repo maintainers**: Run repo2skill on your repo and see what comes out. If the skill is thin, your README might need some love — and your human users will benefit too.

**For the AI agent community**: We think standardized skill quality matters. As agents become more capable, the quality of the skills they use becomes a bottleneck. Better READMEs → better skills → better agents.

---

*Full dataset and analysis scripts available at [NeuZhou/skill-benchmark](https://github.com/NeuZhou).*
