---
title: "Building an AI Knowledge Base from Scratch: 200 Chunks in 3 Hours"
date: 2026-03-16
tags: ["ai", "knowledge-base", "rag", "chromadb", "ollama"]
categories: ["AI Engineering"]
summary: "Why an AI agent needs its own knowledge base, how we built one with Ollama + ChromaDB + Python, and what we learned about chunking the hard way."
ShowToc: true
TocOpen: true
---

An AI agent without a knowledge base is like a developer without Stack Overflow — technically functional, but missing most of the answers.

We spent 3 hours building a local knowledge base from scratch. 200 chunks of curated AI security research, queryable via semantic search, running entirely offline. Here's what we built and what we learned.

## Why an AI Agent Needs Its Own Knowledge Base

Large language models know a lot. But they don't know *your* things — your experiments, your notes, your team's decisions, your specific codebase quirks.

RAG (Retrieval-Augmented Generation) fixes this by letting the agent search a curated collection before answering. Instead of hallucinating, it retrieves actual documents.

Our use case: we're building [OpenClaw](https://github.com/nicepkg/openclaw), an AI agent platform, and we run security experiments. We needed the agent to reference our own research — honeypot results, attack taxonomies, experiment logs — not generic internet knowledge.

## Architecture

The stack is deliberately minimal:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Documents   │────▶│  Ingest      │────▶│  ChromaDB    │
│  (.md files) │     │  Pipeline    │     │  (vectors)   │
└──────────────┘     └──────────────┘     └──────────────┘
                                                │
┌──────────────┐     ┌──────────────┐          │
│  User Query  │────▶│  Ollama      │◀─────────┘
│              │     │  (embeddings │   semantic
└──────────────┘     │  + generate) │   search
                     └──────────────┘
```

- **Ollama** — Local LLM inference. We use `nomic-embed-text` for embeddings and `qwen2.5:7b` for generation.
- **ChromaDB** — Vector database that runs embedded in Python. No server to manage.
- **Python** — The glue. ~100 lines for ingest, ~50 for search.

Why this stack? **Zero cloud dependencies.** Everything runs on a single machine. No API keys, no billing surprises, no data leaving your network.

## The Ingest Pipeline

### Step 1: Gather Documents

We pointed the pipeline at our experiment directories:

```python
sources = [
    "experiments/mcp-honeypot/EXPERIMENT-REPORT.md",
    "experiments/mcp-honeypot/analysis.md",
    "experiments/mini-agent/LEARNINGS.md",
    "notes/attack-taxonomy.md",
    "notes/tool-security-patterns.md",
    # ... 15 more files
]
```

Total: ~50 markdown files, ranging from 500 to 8000 words each.

### Step 2: Chunk

This is where it gets interesting. You can't just shove entire documents into a vector database — the embeddings lose specificity. You need chunks.

We tried three approaches:

| Strategy | Chunk Size | Overlap | Result |
|----------|-----------|---------|--------|
| Fixed 500 chars | 500 | 50 | Broke mid-sentence, terrible retrieval |
| Fixed 1000 chars | 1000 | 100 | Better, but still split logical sections |
| **Markdown headers** | Variable | 0 | Best — each section is a natural chunk |

The winner: **split on markdown headers.** Each `##` section becomes a chunk, with the header hierarchy preserved as metadata. A section about "SQL Injection Detection Rules" stays intact instead of being split across two chunks.

```python
def chunk_markdown(text: str, source: str) -> list[dict]:
    chunks = []
    current_chunk = ""
    current_headers = []

    for line in text.split("\n"):
        if line.startswith("## "):
            if current_chunk.strip():
                chunks.append({
                    "text": current_chunk.strip(),
                    "source": source,
                    "headers": " > ".join(current_headers),
                })
            current_chunk = line + "\n"
            current_headers = [line.lstrip("# ").strip()]
        else:
            current_chunk += line + "\n"

    if current_chunk.strip():
        chunks.append({
            "text": current_chunk.strip(),
            "source": source,
            "headers": " > ".join(current_headers),
        })
    return chunks
```

### Step 3: Embed and Store

ChromaDB handles embedding and storage in one call:

```python
import chromadb

client = chromadb.PersistentClient(path="./knowledge_db")
collection = client.get_or_create_collection(
    name="ai_security",
    metadata={"hnsw:space": "cosine"}
)

for i, chunk in enumerate(all_chunks):
    collection.add(
        ids=[f"chunk_{i}"],
        documents=[chunk["text"]],
        metadatas=[{
            "source": chunk["source"],
            "headers": chunk["headers"]
        }]
    )
```

ChromaDB uses its own default embedding model, or you can plug in Ollama's `nomic-embed-text` for consistency with your generation model.

**Result: 200 chunks from 50 documents, indexed in ~45 seconds.**

## Semantic Search in Action

Once indexed, querying is simple:

```python
results = collection.query(
    query_texts=["How does path traversal work in MCP?"],
    n_results=3
)
```

This returns the 3 most semantically similar chunks — not keyword matches, but *meaning* matches. A query about "escaping the file sandbox" would match our honeypot documentation about `../` path traversal, even though those exact words don't appear.

### Real Examples

| Query | Top Result | Relevance |
|-------|-----------|-----------|
| "MCP attack detection" | Honeypot analysis: detection rules section | ✅ Exact match |
| "How to stop SQL injection in tool calls" | Honeypot B: SQL injection patterns + ClawGuard recommendations | ✅ Great |
| "Agent memory poisoning" | Memory attack taxonomy notes | ✅ Perfect |
| "What is ReAct?" | Mini-agent learnings: ReAct loop section | ✅ Correct |
| "Kubernetes deployment" | ❌ No relevant results (correct — not in our knowledge) | ✅ Expected |

The last case is important: the system correctly returns low-similarity results when the query is outside its knowledge domain. This lets the agent know when to say "I don't know" instead of hallucinating.

## What We Learned About Chunking

### 1. Respect Document Structure

Markdown headers are natural chunk boundaries. Splitting on character count destroys logical units. A table split across two chunks is useless.

### 2. Include Context in Metadata

Each chunk stores its source file and header hierarchy. When the agent retrieves a chunk, it knows *where* it came from and can cite it. Without metadata, you get answers without attribution.

### 3. Smaller Isn't Always Better

We initially tried 300-character chunks for "more precise" retrieval. But short chunks lack context — you get a sentence fragment that's technically relevant but incomprehensible without its surroundings. Our markdown-header approach produces chunks of 200-2000 characters, and the variable size is fine.

### 4. Deduplication Matters

We had overlapping content across experiment reports and analysis docs. Without dedup, the same information appears multiple times in search results, wasting the agent's context window. A simple hash-based dedup on chunk text eliminated ~15% of chunks.

### 5. Embedding Quality Is the Bottleneck

The difference between a good and bad embedding model is dramatic. `nomic-embed-text` (137M params) outperformed larger models for our domain because it handles code and technical text well. Test your specific data — don't assume bigger is better.

## The Full Stack in Numbers

| Metric | Value |
|--------|-------|
| Documents ingested | 50 |
| Chunks created | 200 |
| Ingest time | ~45 seconds |
| Database size on disk | ~12 MB |
| Average query time | <100ms |
| Setup time (total) | ~3 hours |
| Dependencies | 3 (chromadb, ollama, requests) |

## Future: Knowledge Graphs

Vector search finds *similar* content. But it doesn't understand *relationships* — that Experiment A relates to ClawGuard, which depends on the detection rules defined in the analysis doc.

The next step is adding a lightweight knowledge graph layer:

```
[Honeypot A] --tests--> [Path Traversal]
[Path Traversal] --detected-by--> [ClawGuard Regex Filter]
[ClawGuard] --protects--> [MCP Tools]
[MCP Tools] --used-by--> [AI Agents]
```

This would let the agent traverse relationships: "What defends against the attacks our honeypots found?" → follow edges from honeypot findings to detection rules to ClawGuard components.

We're exploring [NetworkX](https://networkx.org/) for the graph structure and a hybrid retrieval approach: vector search for initial candidates, then graph traversal for related context.

## Try It Yourself

The full pipeline runs on any machine with Python and Ollama installed:

```bash
# Install dependencies
pip install chromadb requests

# Pull embedding model
ollama pull nomic-embed-text

# Run ingest (point at your own docs)
python ingest.py --source ./your-documents/

# Query
python search.py "your question here"
```

The beauty of this setup: **your data never leaves your machine.** No cloud APIs, no embeddings sent to third parties, no billing. Just local inference on local data.

---

*200 chunks might sound small, but it's enough to transform an agent from "I'll make something up" to "according to your honeypot experiment..." — and that's the whole point.*
