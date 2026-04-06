# AI Agent Framework Comparison

> A side-by-side comparison of every agent framework we've torn down. Updated as new teardowns are published.

*Last updated: 2026-04-06 • 4 projects analyzed*

---

## Quick Reference

| | Claude Code | DeerFlow 2.0 | Hermes Agent | oh-my-claudecode |
|--|------------|-------------|-------------|-----------------|
| **Creator** | Anthropic | ByteDance | Nous Research | Yeachan Heo |
| **Stars** | ~510K LOC analyzed (closed source, leaked via source map) | 58K ⭐ | 26K ⭐ | 24K ⭐ |
| **Language** | TypeScript | Python + TypeScript | Python | TypeScript |
| **Type** | Standalone CLI | Standalone framework | Standalone CLI | Claude Code plugin |
| **License** | Proprietary | MIT | MIT | MIT |

---

## Architecture

| Dimension | Claude Code | DeerFlow 2.0 | Hermes Agent | oh-my-claudecode |
|-----------|------------|-------------|-------------|-----------------|
| **Agent loop** | Monolithic `while(true)` in a single class | LangGraph + 14-layer middleware chain | Monolithic 9K-line `run_agent.py` | Plugin layer on Claude Code + team pipeline |
| **Extensibility model** | Internal tool registry via `buildTool()` | Middleware (add/remove without touching core) | Monolithic (every change touches one file) | 19 specialized agent definitions |
| **Config format** | CLI flags + `.claude/` directory | YAML (`config.yaml`) | YAML + dotfiles | Plugin settings + `.omc/` |
| **Deployment** | Bun binary (standalone) | Nginx + LangGraph Server + Gateway API | pip install + CLI | npm package (requires Claude Code host) |

---

## Multi-Agent & Orchestration

| Dimension | Claude Code | DeerFlow 2.0 | Hermes Agent | oh-my-claudecode |
|-----------|------------|-------------|-------------|-----------------|
| **Sub-agent support** | No (single agent) | Yes — thread pool, max 3 concurrent | Yes — in-process delegate, max 3, depth 1 | Yes — 19 role-based agents via tmux |
| **Multi-model** | Claude only | Single provider (configurable) | Any provider (OpenRouter, etc.) | Claude + Codex + Gemini |
| **IPC mechanism** | N/A | Thread pool + shared state | In-process function call | File-based inbox/outbox (mkdir locking) |
| **Task pipeline** | None | None | None | plan → exec → verify → fix (with retry) |
| **Model routing** | Single model | Config-based | Config-based, model switching via CLI | Tiered: Haiku (simple) → Sonnet (balanced) → Opus (complex) |

---

## Memory & Persistence

| Dimension | Claude Code | DeerFlow 2.0 | Hermes Agent | oh-my-claudecode |
|-----------|------------|-------------|-------------|-----------------|
| **Memory storage** | `.claude/` rules file (manual) | JSON file (hierarchical schema) | MEMORY.md + USER.md (markdown) | Project + user skill scopes |
| **Memory structure** | Flat rules/instructions | user context / history / facts (3 time horizons) | Flat entries with § delimiter | Skill YAML files with triggers |
| **Update mechanism** | Manual only | LLM-extracted, debounced, async | Agent tool calls (add/replace/remove) | Auto-learning via `/learner` |
| **Confidence scoring** | No | Yes (per-fact 0-1) | No | No |
| **Cross-session recall** | No | No | Yes — FTS5 + LLM summarization | No |
| **Prompt cache optimization** | N/A | No (memory can change mid-session) | Yes — frozen snapshot at session start | N/A |
| **Memory security scanning** | No | No | Yes — threat pattern detection | No |
| **Concurrent write safety** | N/A | No file locking ⚠️ | fcntl file locking | mkdir-based locking |

---

## Safety & Security

| Dimension | Claude Code | DeerFlow 2.0 | Hermes Agent | oh-my-claudecode |
|-----------|------------|-------------|-------------|-----------------|
| **Loop detection** | Built-in (details unknown) | Hash-based, warn@3, kill@5 | Max iterations limit | Not documented |
| **Cost budgets** | N/A (Anthropic-hosted) | ❌ None | ❌ None | Token tracking, no hard limit |
| **Sandbox isolation** | `sandbox-exec` (macOS), none (Linux) | Local filesystem or Docker | 6 backends (Local/Docker/SSH/Daytona/Singularity/Modal) | Inherits Claude Code sandbox |
| **Auth/RBAC** | N/A (local CLI) | ❌ None (advisory only) | DM pairing for messaging | Inherits Claude Code |
| **Skill security** | N/A | N/A | Security scanning on agent-authored skills | N/A |

---

## Communication & Channels

| Dimension | Claude Code | DeerFlow 2.0 | Hermes Agent | oh-my-claudecode |
|-----------|------------|-------------|-------------|-----------------|
| **CLI** | Yes (Bun TUI) | Yes | Yes (full TUI) | Yes (via Claude Code) |
| **IM channels** | No | Feishu, Slack, Telegram | Telegram, Discord, Slack, WhatsApp, Signal, Email | No (plugin only) |
| **Cron/scheduling** | No | No | Yes (built-in with platform delivery) | No |
| **Voice** | No | No | Yes (transcription + TTS) | No |

---

## Learning & Self-Improvement

| Dimension | Claude Code | DeerFlow 2.0 | Hermes Agent | oh-my-claudecode |
|-----------|------------|-------------|-------------|-----------------|
| **Skill auto-creation** | No | No | Yes — creates skills from experience | Yes — `/learner` extracts patterns |
| **Skill self-improvement** | No | No | Yes — patches skills during use | No |
| **Context compression** | 4-layer cascade (pinned→summarize→evict→truncate) | SummarizationMiddleware (configurable) | 5-step (prune tools→protect head/tail→structured summary→iterative update) | Inherits Claude Code |

---

## Patterns We Found

Design patterns identified across the four projects:

| Pattern | Claude Code | DeerFlow | Hermes | OMC |
|---------|:----------:|:--------:|:------:|:---:|
| Middleware chain | ❌ | ✅ | ❌ | ❌ |
| Memory frozen snapshot | ❌ | ❌ | ✅ | ❌ |
| Loop detection (hash-based) | ? | ✅ | ❌ | ❌ |
| Skill self-improvement | ❌ | ❌ | ✅ | Partial |
| File-based IPC | ❌ | ❌ | ❌ | ✅ |
| Model tier routing | ❌ | ❌ | ❌ | ✅ |
| Structured context compression | ✅ | ✅ | ✅ | Inherited |

---

## Anti-Patterns We Found

| Anti-Pattern | Where | Severity |
|-------------|-------|----------|
| God file (9K+ lines) | Hermes `run_agent.py` | High |
| No cost budgets | DeerFlow, Hermes | High |
| No auth/RBAC | DeerFlow | High |
| Single-file JSON memory (no locking) | DeerFlow | Medium |
| Upstream dependency risk | OMC (depends on Claude Code internals) | Medium |

---

## If I Were Building an Agent Today

After analyzing four agent frameworks totaling ~960K lines of code:

- **Architecture:** Take DeerFlow's middleware chain. It's the cleanest extensibility model.
- **Memory:** Take Hermes's frozen snapshot + FTS5 session search. Add DeerFlow's confidence-scored facts.
- **Security:** Take Hermes's memory threat scanning + DeerFlow's loop detection. Add cost budgets (nobody has this yet).
- **Multi-agent:** Take OMC's model tier routing concept but implement it with DeerFlow's thread pool (not file-based IPC).
- **Learning:** Take Hermes's skill self-improvement with security scanning.
- **Context:** Take Claude Code's 4-layer cascade structure with Hermes's structured summary template.

---

*Part of [awesome-ai-anatomy](https://github.com/NeuZhou/awesome-ai-anatomy) — source-level teardowns of how production AI systems actually work.*
