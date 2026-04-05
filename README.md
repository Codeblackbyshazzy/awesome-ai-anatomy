<p align="center">
  <img src="https://img.shields.io/github/stars/NeuZhou/awesome-ai-anatomy?style=social" alt="Stars">
  <img src="https://img.shields.io/github/forks/NeuZhou/awesome-ai-anatomy?style=social" alt="Forks">
  <img src="https://img.shields.io/badge/teardowns-1-blue" alt="Teardowns">
  <img src="https://img.shields.io/badge/updated-daily-brightgreen" alt="Updated Daily">
  <img src="https://img.shields.io/badge/license-MIT-orange" alt="License">
</p>

# 🔬 Awesome AI Anatomy

**Staff-engineer-level architecture teardowns of the most important AI projects — one per day.**

Most "awesome" lists link to repos. We **dissect** them — architecture diagrams, design decisions, trade-off analysis, God Object code smells, hidden easter eggs, and engineering insights you won't find in the docs.

> ⭐ **Star this repo** to get daily teardowns in your GitHub feed.

---

## 📋 Table of Contents

- [Project Index](#-project-index)
- [Latest: Claude Code](#-latest-claude-code--510k-lines-dissected)
- [What Makes This Different](#-what-makes-this-different)
- [Each Teardown Includes](#-each-teardown-includes)
- [Video Series](#-video-series)
- [Contributing](#-contributing)
- [Stay Updated](#-stay-updated)

---

## 📊 Project Index

| # | Project | Lines | Key Findings | Status |
|---|---------|-------|-------------|--------|
| 001 | [**Claude Code**](claude-code/) | 510K TS | 4-layer context management, streaming tool execution, hidden pet system (18 species!), 1729-line God Object | ✅ Published |
| 002 | **DeerFlow** (ByteDance) | TBD | Coming soon | 🔜 Next |
| 003 | **Cursor** | TBD | — | 📋 Planned |
| 004 | **Dify** | TBD | — | 📋 Planned |
| 005 | **AutoGPT** | TBD | — | 📋 Planned |

---

## 🔥 Latest: Claude Code — 510K Lines Dissected

Anthropic's AI coding agent, leaked via npm source maps on 2026/03/31. 510,000 lines of TypeScript. Here's what we found.

### Architecture at a Glance

```
┌──────────────────────────────────────────────────────────┐
│                    CLI Entry (Bun)                        │
│  Cold start 4-5x faster than Node · but ecosystem risk   │
├──────────────────────────────────────────────────────────┤
│              React + Ink Terminal UI                      │
│  Not for beauty — for state management complexity        │
├──────────────────────────────────────────────────────────┤
│           while(true) Agent Loop                         │
│  Simple but created a 1729-line God Object (query.ts)    │
├──────────────────┬───────────────────────────────────────┤
│   40 Tool Plugins │    4-Layer Context Management        │
│   BashTool        │    L1: System prompt (always)        │
│   FileRead/Write  │    L2: Conversation (importance-     │
│   WebFetch        │        weighted, not sliding window) │
│   LSP Integration │    L3: Tool results (summarized)     │
│   SubAgent spawn  │    L4: Compressed (irreversible!)    │
├──────────────────┴───────────────────────────────────────┤
│              3-Layer Memory Architecture                 │
│  MEMORY.md (pointer index, always in context)            │
│  Topic Files (loaded on demand)                          │
│  Raw Transcripts (grep only, never fully loaded)         │
├──────────────────────────────────────────────────────────┤
│          Streaming Parallel Execution                    │
│  RWLock model · subtle race conditions with file changes │
├──────────────────────────────────────────────────────────┤
│            Hidden Systems (Easter Eggs)                  │
│  BUDDY: Tamagotchi pet (18 species, 5 rarities, 1%      │
│         shiny chance, Mulberry32 PRNG)                   │
│  KAIROS: Unreleased autonomous background agent          │
│  Anti-Distillation: Fake tool injection to poison        │
│         competitor training data                         │
└──────────────────────────────────────────────────────────┘
```

### 🏗️ Key Design Decisions (Why, Not Just What)

| Decision | Why They Chose It | The Trade-off |
|----------|-------------------|---------------|
| **Bun over Node** | Cold start 4-5x faster for CLI tool | Ecosystem risk — known bug #28001 caused the source map leak |
| **React+Ink for terminal** | State management complexity, not UI beauty | Heavy dependency for a CLI tool |
| **`while(true)` over state machine** | Simplicity | Created a 1729-line God Object in `query.ts` |
| **Importance-weighted context** | Better than sliding window (keeps relevant, drops noise) | Introduces non-determinism — same conversation can produce different context |
| **Flat worker model (no nesting)** | Avoids complexity explosion | Artificial ceiling for recursive task decomposition |
| **40 hardcoded tools** | Works now, each tool is simple | Won't scale past ~100 tools — needs tool families/factories |

### ⚠️ Problems We Found

1. **`query.ts` God Object** — 1,729 lines, growing. Merge conflicts guaranteed at scale.
2. **Irreversible context compression** — Once compressed, the model literally doesn't know what it forgot. Unauditable.
3. **Dual feature flag systems** — Increases cognitive load for contributors.
4. **No worker nesting** — Can't do recursive task decomposition (e.g., "refactor module A" → spawn sub-tasks per file).
5. **Anti-Distillation raises ethical questions** — Injecting fake tool definitions to poison competitor training data.

### 🆚 How It Compares

| Aspect | Claude Code | Cursor | LangChain |
|--------|------------|--------|-----------|
| **Paradigm** | Terminal agent (bet on agent future) | IDE augmentation (meet users where they are) | Framework (build your own) |
| **Context** | Importance-weighted 4-layer | Sliding window + RAG | User-managed |
| **Tools** | 40 hardcoded, intentionally non-general | IDE-integrated | Generic tool interface |
| **Memory** | 3-layer (MEMORY.md → Topics → Transcripts) | Per-project context | User-implemented |

### 📖 Full Analysis

➡️ **[Read the complete teardown →](claude-code/README.md)**

---

## 🎯 What Makes This Different

| Other "awesome" lists | This repo |
|----------------------|-----------|
| Link to repos | **Dissect** repos |
| "What they use" | **Why they chose it** |
| Stars and badges | **Architecture diagrams + trade-off analysis** |
| Surface-level | **Staff-engineer-level depth** |
| Praise everything | **Find real problems** |

---

## 📐 Each Teardown Includes

```
📄 README.md          — Full architecture analysis + design decisions
📊 *.mmd              — Mermaid diagram source files
🎬 video-script-*.md  — Video scripts (Chinese + English)
🖼️ slides-guide.md    — Slide deck content guide
⚠️ Problems found     — Bugs, code smells, architectural risks
🆚 Comparisons        — How it stacks up against alternatives
```

---

## 🎬 Video Series

Watch the video versions:

| Platform | Language | Link |
|----------|----------|------|
| **B站** | 🇨🇳 Chinese | [Agent零号](https://www.bilibili.com/) |
| **YouTube** | 🇺🇸 English | Coming soon |
| **头条** | 🇨🇳 Chinese | [Agent0](https://www.toutiao.com/) |

---

## 🤝 Contributing

Found an error? Have a better analysis? PRs welcome!

- 🐛 Fix factual errors
- 🔍 Add missing architecture details
- 💡 Suggest projects to teardown → [Open an Issue](https://github.com/NeuZhou/awesome-ai-anatomy/issues)
- 🌍 Help translate teardowns

---

## 📌 Stay Updated

This repo is updated daily. **Star ⭐ and Watch 👁️ to follow along.**

**Next up: ByteDance DeerFlow** — How China's hottest AI company builds their multi-agent research framework.

---

<p align="center">
  <i>Built by <a href="https://github.com/NeuZhou">Agent零号</a> — daily AI architecture insights from a senior engineer's perspective.</i>
</p>
