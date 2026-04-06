# AI Agent Framework Comparison

> A side-by-side comparison of every project we've torn down. Updated as new teardowns are published.

*Last updated: 2026-04-06 • 10 projects analyzed*

---

## Quick Reference

With 10 projects, we split the table into two groups for readability.

### Group A — Agent Frameworks & Coding Agents

| | Claude Code | DeerFlow 2.0 | Hermes Agent | oh-my-claudecode | Goose |
|--|------------|-------------|-------------|-----------------|-------|
| **Creator** | Anthropic | ByteDance | Nous Research | Yeachan Heo | Block Inc |
| **Stars** | N/A (proprietary, leaked) | 58K ⭐ | 26K ⭐ | 24K ⭐ | 37K ⭐ |
| **Language** | TypeScript | Python + TypeScript | Python | TypeScript | Rust + TypeScript |
| **Lines of Code** | ~510K | — | ~260K | ~194K | ~198K |
| **Type** | Standalone CLI | Standalone framework | Standalone CLI | Claude Code plugin | Standalone CLI + Desktop |
| **License** | Proprietary | MIT | MIT | MIT | Apache-2.0 |

### Group B — Platforms, Tools & Specialized

| | Dify | Pi Mono | Lightpanda | MiroFish | Guardrails AI |
|--|------|---------|------------|----------|---------------|
| **Creator** | Dify.AI | Mario Zechner | Lightpanda GmbH | MiroFish team | Guardrails AI Inc |
| **Stars** | 136K ⭐ | 32K ⭐ | 27K ⭐ | 50K ⭐ | 6.6K ⭐ |
| **Language** | Python + TypeScript | TypeScript | Zig + Rust | Python + Vue.js | Python |
| **Lines of Code** | ~1,240K | ~147K | ~91K | ~39K | ~18K |
| **Type** | Visual AI platform | Multi-package coding agent | Headless browser | Prediction simulation | LLM output validator |
| **License** | Modified Apache 2.0 | MIT | AGPL-3.0 | AGPL-3.0 | Apache-2.0 |

---

## Architecture

### Group A — Agent Frameworks & Coding Agents

| Dimension | Claude Code | DeerFlow 2.0 | Hermes Agent | oh-my-claudecode | Goose |
|-----------|------------|-------------|-------------|-----------------|-------|
| **Agent loop** | Monolithic `while(true)` in a single class | LangGraph + 14-layer middleware chain | Monolithic 9K-line `run_agent.py` | Plugin layer on Claude Code + team pipeline | `Agent.reply()` dispatcher loop (900-line struct) |
| **Extensibility** | Internal tool registry via `buildTool()` | Middleware (add/remove without touching core) | Monolithic (every change touches one file) | 19 specialized agent definitions | MCP-first: 6 extension types (stdio/builtin/platform/HTTP/frontend/inline_python) |
| **Config format** | CLI flags + `.claude/` directory | YAML (`config.yaml`) | YAML + dotfiles | Plugin settings + `.omc/` | YAML + profiles |
| **Deployment** | Bun binary (standalone) | Nginx + LangGraph Server + Gateway API | pip install + CLI | npm package (requires Claude Code host) | Single Rust binary (CLI) or Electron (desktop) |

### Group B — Platforms, Tools & Specialized

| Dimension | Dify | Pi Mono | Lightpanda | MiroFish | Guardrails AI |
|-----------|------|---------|------------|----------|---------------|
| **Core architecture** | Visual workflow builder (ReactFlow front, Flask+graphon back) | 7-package monorepo with game-engine layering | Pipeline: libcurl → html5ever → Zig DOM → V8 → CDP | Flask API + OASIS social simulation engine | Guard → Runner → ValidatorService → Validators (onion pattern) |
| **Extensibility** | Plugin daemon (separate process), node types | Extension API with 30+ event types, custom tools/providers | Web API stub additions via Zig comptime bridge | Limited (OASIS engine is the core) | Validator Hub (`guardrails install`) — npm for validators |
| **Config format** | 400+ env vars in docker-compose + web admin | TypeScript config + extension manifest | Build flags, compile-time options | Flask config + YAML | Python API + RAIL XML (deprecated) |
| **Deployment** | 7+ Docker containers + vector DB | npm packages (`npx @mariozechner/pi-coding-agent`) | Single binary (Zig compiled) | Docker (Flask + Vue + Zep Cloud) | pip install (library, not standalone) |

---

## Multi-Agent & Orchestration

| Dimension | Claude Code | DeerFlow | Hermes | OMC | Goose | Pi Mono | Dify |
|-----------|:----------:|:--------:|:------:|:---:|:-----:|:-------:|:----:|
| **Sub-agent support** | ❌ | ✅ (thread pool, max 3) | ✅ (in-process, max 3, depth 1) | ✅ (19 role-based via tmux) | ❌ (single agent) | ❌ (single agent) | ✅ (child workflow engines) |
| **Multi-model** | Claude only | Single provider (configurable) | Any provider (OpenRouter) | Claude + Codex + Gemini | 30+ providers | All major providers (lazy-loaded) | All major providers |
| **IPC mechanism** | N/A | Thread pool + shared state | In-process function call | File-based inbox/outbox (mkdir locking) | N/A | N/A | GraphEngine child spawning |
| **Task pipeline** | None | None | None | plan → exec → verify → fix (with retry) | None | Steering + follow-up queues | Visual DAG (user-designed) |
| **Model routing** | Single | Config-based | Config-based | Tiered: Haiku → Sonnet → Opus | Config-based | Config-based + stealth mode | Per-node in workflow |

> **Lightpanda**, **MiroFish**, and **Guardrails AI** are not agent frameworks and don't have agent orchestration. MiroFish uses OASIS for multi-agent social simulation (LLM role-play), but that's a different paradigm — simulating human social behavior, not task orchestration.

---

## Memory & Persistence

| Dimension | Claude Code | DeerFlow | Hermes | OMC | Goose |
|-----------|------------|----------|--------|-----|-------|
| **Memory storage** | `.claude/` rules file (manual) | JSON file (hierarchical schema) | MEMORY.md + USER.md (markdown) | Project + user skill scopes | MCP memory extension (built-in) |
| **Memory structure** | Flat rules/instructions | 3 time horizons (user context / history / facts) | Flat entries with § delimiter | Skill YAML files with triggers | Key-value via MCP |
| **Update mechanism** | Manual only | LLM-extracted, debounced, async | Agent tool calls (add/replace/remove) | Auto-learning via `/learner` | MCP tool calls |
| **Confidence scoring** | ❌ | ✅ (per-fact 0-1) | ❌ | ❌ | ❌ |
| **Cross-session recall** | ❌ | ❌ | ✅ (FTS5 + LLM summarization) | ❌ | ✅ (SQLite sessions) |
| **Prompt cache optimization** | N/A | ❌ (memory can change mid-session) | ✅ (frozen snapshot at session start) | N/A | ❌ |
| **Concurrent write safety** | N/A | ❌ (no file locking ⚠️) | ✅ (fcntl file locking) | ✅ (mkdir-based locking) | N/A |

| Dimension | Dify | Pi Mono | Lightpanda | MiroFish | Guardrails AI |
|-----------|------|---------|------------|----------|---------------|
| **Memory storage** | PostgreSQL + Redis (multi-tenant) | Extension-provided | N/A (browser, no agent memory) | Zep Cloud (knowledge graph + memory) | Execution history (in-memory) |
| **Memory structure** | Conversation variables + knowledge base | Extension-defined | N/A | Graph nodes + edges (entities + relations) | Call logs per guard invocation |
| **Cross-session** | ✅ (database-backed) | Extension-dependent | N/A | ✅ (Zep persistent graph) | ❌ |

---

## Safety & Security

| Dimension | Claude Code | DeerFlow | Hermes | OMC | Goose |
|-----------|------------|----------|--------|-----|-------|
| **Loop detection** | Built-in (details unknown) | ✅ Hash-based, warn@3, kill@5 | Max iterations limit | Not documented | ✅ RepetitionInspector |
| **Cost budgets** | N/A (Anthropic-hosted) | ❌ None | ❌ None | Token tracking, no hard limit | ❌ None |
| **Sandbox isolation** | `sandbox-exec` (macOS), none (Linux) | Local filesystem or Docker | 6 backends (Local/Docker/SSH/Daytona/Singularity/Modal) | Inherits Claude Code sandbox | Docker + platform-extension scoping |
| **Tool inspection** | N/A | N/A | N/A | N/A | ✅ 5-inspector pipeline (Security → Egress → Adversary → Permission → Repetition) |
| **Auth/RBAC** | N/A (local CLI) | ❌ None (advisory only) | DM pairing for messaging | Inherits Claude Code | N/A (local CLI) |
| **Anti-distillation** | ✅ (fake tool injection) | ❌ | ❌ | ❌ | ❌ |

| Dimension | Dify | Pi Mono | Lightpanda | MiroFish | Guardrails AI |
|-----------|------|---------|------------|----------|---------------|
| **Sandbox isolation** | ✅ Go-based code sandbox + SSRF proxy (Squid) | None | N/A (browser sandbox by design) | None | N/A (library) |
| **Auth/RBAC** | ✅ Multi-tenant with workspace isolation | None | N/A | None | N/A |
| **LLM output validation** | ❌ | ❌ | N/A | ❌ | ✅ Core purpose (schema + validators + reask loop) |
| **Cost controls** | ✅ LLM quota layer in workflow engine | Auto-retry with backoff (no budget) | N/A | ❌ (multi-LLM-call pipeline, no tracking) | ❌ |

---

## Context Management

| Dimension | Claude Code | DeerFlow | Hermes | OMC | Goose | Pi Mono |
|-----------|------------|----------|--------|-----|-------|---------|
| **Compaction strategy** | 4-layer cascade (pinned → summarize → evict → truncate) | SummarizationMiddleware (configurable) | 5-step (prune tools → protect head/tail → structured summary → iterative update) | Inherits Claude Code | Auto-compact + tool-pair summarization | LLM-based summary, extension-overridable |
| **Irreversible?** | ✅ (once compressed, gone) | Configurable | Iterative fallback | Inherited | Summarized (replaceable) | Summarized |
| **Overflow recovery** | N/A | N/A | N/A | N/A | N/A | ✅ Auto-remove error, compact, retry |

---

## Communication & Channels

| Capability | Claude Code | DeerFlow | Hermes | OMC | Goose | Pi Mono | Dify |
|------------|:----------:|:--------:|:------:|:---:|:-----:|:-------:|:----:|
| **CLI/TUI** | ✅ (Bun TUI) | ✅ | ✅ (full TUI) | ✅ (via Claude Code) | ✅ (Rust CLI) | ✅ (custom TUI library) | ❌ |
| **Desktop app** | ❌ | ❌ | ❌ | ❌ | ✅ (Electron) | ❌ | ❌ |
| **Web UI** | ❌ | ✅ (Next.js) | ❌ | ❌ | ❌ | ✅ (Lit web components) | ✅ (Next.js) |
| **IM channels** | ❌ | Feishu, Slack, Telegram | Telegram, Discord, Slack, WhatsApp, Signal, Email | ❌ | ❌ | Slack (pi-mom) | ❌ |
| **Cron/scheduling** | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Voice** | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ (TTS node) |

---

## Learning & Self-Improvement

| Dimension | Claude Code | DeerFlow | Hermes | OMC | Goose | Pi Mono |
|-----------|:----------:|:--------:|:------:|:---:|:-----:|:-------:|
| **Skill auto-creation** | ❌ | ❌ | ✅ | ✅ (`/learner`) | ❌ | ❌ |
| **Skill self-improvement** | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Skill security scanning** | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |

---

## Patterns We Found

Design patterns identified across all 10 projects:

| Pattern | CC | DF | HM | OMC | Goose | Pi | Dify | LP | MF | GA |
|---------|:--:|:--:|:--:|:---:|:-----:|:--:|:----:|:--:|:--:|:--:|
| Middleware chain | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ (layers) | ❌ | ❌ | ❌ |
| Memory frozen snapshot | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | — | ❌ | ❌ |
| Loop detection (hash-based) | ? | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | — | ❌ | ❌ |
| Skill self-improvement | ❌ | ❌ | ✅ | partial | ❌ | ❌ | ❌ | — | ❌ | ❌ |
| File-based IPC | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | — | ❌ | ❌ |
| Model tier routing | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ (stealth) | ✅ (per-node) | — | ❌ | ❌ |
| Structured context compression | ✅ | ✅ | ✅ | inherited | ✅ | ✅ | ✅ | — | ❌ | ❌ |
| MCP as extension protocol | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ (server) | ❌ | ❌ |
| Tool inspection pipeline | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | — | ❌ | ❌ |
| Visual workflow builder | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | — | ❌ | ❌ |
| Validator ecosystem (Hub) | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | — | ❌ | ✅ |
| Comptime code generation | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |

> **Legend:** CC=Claude Code, DF=DeerFlow, HM=Hermes, OMC=oh-my-claudecode, Pi=Pi Mono, LP=Lightpanda, MF=MiroFish, GA=Guardrails AI. "—" means not applicable (not an agent framework).

---

## Anti-Patterns We Found

| Anti-Pattern | Where | Severity |
|-------------|-------|----------|
| God file (9K+ lines) | Hermes `run_agent.py` | High |
| God file (1,729 lines, growing) | Claude Code `query.ts` | High |
| God file (1,500+ lines) | Pi Mono `agent-session.ts` | Medium |
| God file (1,400+ lines) | MiroFish `report_agent.py` | Medium |
| No cost budgets | DeerFlow, Hermes, Goose, MiroFish | High |
| No auth/RBAC | DeerFlow | High |
| Single-file JSON memory (no locking) | DeerFlow | Medium |
| Upstream dependency risk | OMC (depends on Claude Code internals) | Medium |
| Upstream dependency risk | MiroFish (depends on OASIS + Zep Cloud) | Medium |
| Stealth mode / API impersonation | Pi Mono (impersonates Claude Code tool names) | Medium |
| Irreversible context compression | Claude Code (4th layer can't be audited) | Medium |
| Anti-distillation (fake tool injection) | Claude Code | Ethical concern |
| 400+ env vars in deployment | Dify docker-compose | Operational complexity |
| RAIL XML dead weight | Guardrails AI (deprecated but still in codebase) | Low |

---

## Key Insight: What 10 Projects Tell Us About the AI Agent Landscape

After analyzing 10 projects totaling over **3 million lines of code**, several macro-trends emerge:

### 1. The God Object Problem Is Universal

Every substantial agent framework we examined has at least one oversized core file: Claude Code's `query.ts` (1,729 lines), Hermes's `run_agent.py` (9,000+ lines), Pi Mono's `agent-session.ts` (1,500+ lines), Goose's `extension_manager.rs` (2,300 lines). The agent loop naturally attracts concerns like a gravitational well — context management, tool dispatch, error handling, and state tracking all want to live close to the main loop. Only DeerFlow (middleware chain) and Dify (graph engine extraction) have found structural solutions to this problem.

### 2. The Extensibility Spectrum

Projects fall on a clear spectrum from monolithic to modular:
- **Most modular:** Goose (everything is an MCP extension), Dify (plugin daemon as separate process), Pi Mono (7 standalone packages)
- **Middleware-based:** DeerFlow (14-layer chain)
- **Monolithic with hooks:** Hermes, Claude Code, OMC
- **Library pattern:** Guardrails AI (validators as installable packages)
- **Purpose-built:** Lightpanda (single binary, no plugin system needed)

### 3. Security Is Still an Afterthought

Goose is the only project with a proper tool inspection pipeline (5 inspectors including LLM-based adversarial review). Everyone else bolts on permission checks or skips them entirely. Dify gets credit for its SSRF proxy and sandbox isolation. Nobody has implemented cost budgets that actually enforce spending limits — this is the single biggest gap across all 10 projects.

### 4. Memory Is the Unsolved Problem

Every project handles memory differently, and none have it fully figured out:
- **Best cross-session recall:** Hermes (FTS5 + LLM summarization), Dify (database-backed)
- **Best prompt cache efficiency:** Hermes (frozen snapshots)
- **Best structured memory:** DeerFlow (confidence-scored facts)
- **Most creative:** MiroFish (knowledge graph via Zep for social simulation)
- The dream combination — confidence-scored facts with frozen snapshots and full-text session search — doesn't exist yet.

### 5. The MCP Bet

Goose and Lightpanda have made MCP a first-class citizen. Goose's entire extension system is built on MCP, and Lightpanda ships an MCP server mode alongside CDP. As MCP becomes the standard protocol for tool-use, frameworks that adopted it early will have an advantage. Most others (Claude Code, Hermes, DeerFlow) still use internal tool registries.

### 6. Not Everything Is an Agent

The 10 projects span four distinct categories:
- **Coding agents** (Claude Code, Hermes, OMC, Pi Mono, Goose) — interactive terminal tools
- **Platforms** (Dify, DeerFlow) — build-your-own-agent infrastructure
- **Infrastructure** (Lightpanda) — specialized tooling for agents
- **Specialized** (MiroFish, Guardrails AI) — domain-specific applications

The healthiest projects have a clear identity. The ones trying to be everything (agent + platform + marketplace) accumulate the most complexity.

---

## If I Were Building an Agent Today

After analyzing 10 projects totaling ~3M lines of code:

- **Architecture:** Take DeerFlow's middleware chain or Goose's MCP-extension model. Both solve extensibility cleanly.
- **Memory:** Take Hermes's frozen snapshot + FTS5 session search. Add DeerFlow's confidence-scored facts. Use Dify's approach for multi-tenant persistence.
- **Security:** Take Goose's 5-inspector tool inspection pipeline. Add Dify's SSRF proxy and sandbox isolation. **Actually implement cost budgets** — nobody has this yet.
- **Multi-agent:** Take OMC's model tier routing concept but implement it with proper IPC (not file-based). Dify's child engine spawning is cleanest for DAG workflows.
- **Learning:** Take Hermes's skill self-improvement with security scanning.
- **Context:** Take Claude Code's 4-layer cascade structure with Hermes's structured summary template. Add Pi Mono's overflow recovery (auto-compact on error + retry).
- **Provider abstraction:** Take Pi Mono's lazy-loaded provider system (but skip the stealth mode 🙃).
- **TUI:** Take Pi Mono's differential rendering library — it's the best standalone TUI in the group.
- **Browser tooling:** Use Lightpanda for headless agent tasks — 9x speed gain matters at scale.
- **Output validation:** Wrap critical LLM calls in Guardrails-style validation loops with schema enforcement.

---

*Part of [awesome-ai-anatomy](https://github.com/NeuZhou/awesome-ai-anatomy) — source-level teardowns of how production AI systems actually work.*
