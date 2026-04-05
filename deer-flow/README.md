# DeerFlow 2.0: Bytedance's Open-Source SuperAgent Harness

> A Staff Engineer's deep-dive into how ByteDance built a production-grade agent orchestration system that hit #1 on GitHub Trending.

## At a Glance

| Metric | Value |
|--------|-------|
| Stars | 58,080 |
| Forks | 7,241 |
| Language | Python (backend) + TypeScript (frontend) |
| Framework | LangGraph + FastAPI + Next.js |
| License | MIT |
| First commit | May 2025 |
| v2.0 | Feb 2026 (complete ground-up rewrite) |

DeerFlow is not another ChatGPT wrapper. It's a **SuperAgent harness** — an orchestration layer that manages sub-agents, sandboxed execution, persistent memory, and multi-channel messaging. Think of it as the OS that makes a single LLM behave like a team of engineers.

---

## Architecture Overview

```
┌──────────────────────────────────────┐
│         Nginx (Port 2026)            │
│         Unified reverse proxy        │
└───────┬──────────────────┬───────────┘
        │                  │
        ▼                  ▼
┌────────────────────┐  ┌──────────────────────┐
│  LangGraph Server  │  │   Gateway API (8001)  │
│   (Port 2024)      │  │   FastAPI REST        │
│                    │  │                      │
│  ┌──────────────┐  │  │  Models, MCP, Skills, │
│  │  Lead Agent  │  │  │  Memory, Uploads,     │
│  │  ┌────────┐  │  │  │  Artifacts            │
│  │  │Middlew.│  │  │  └──────────────────────┘
│  │  │ Chain  │  │  │
│  │  └────────┘  │  │
│  │  ┌────────┐  │  │
│  │  │ Tools  │  │  │
│  │  └────────┘  │  │
│  │  ┌────────┐  │  │
│  │  │SubAgent│  │  │
│  │  └────────┘  │  │
│  └──────────────┘  │
└────────────────────┘
```

The system splits into two backend services behind Nginx:

1. **LangGraph Server** — runs the actual agent loop. Every user message enters here, passes through the middleware chain, hits the LLM, and tools get called.
2. **Gateway API** — a FastAPI REST service that handles everything besides inference: model config, MCP server management, skill CRUD, memory endpoints, file uploads, and artifact serving.

This separation is deliberate. The agent loop is stateful and long-running (a single task can run for hours). The Gateway is stateless REST. Deploying them separately means you can scale them independently.

---

## The Middleware Chain: DeerFlow's Secret Weapon

This is the most interesting piece of engineering in the whole codebase.

Every message passes through a chain of middlewares before reaching the LLM. Each middleware handles exactly one cross-cutting concern. The ordering is strict and documented in code comments:

```python
# ThreadDataMiddleware must be before SandboxMiddleware
#   to ensure thread_id is available
# DanglingToolCallMiddleware patches missing ToolMessages
#   before model sees the history
# SummarizationMiddleware should be early to reduce context
#   before other processing
# ClarificationMiddleware should be last to intercept
#   clarification requests after model calls
```

### The Full Chain (14 middlewares in v2.0):

| # | Middleware | What it does | Why it matters |
|---|-----------|-------------|----------------|
| 1 | ThreadDataMiddleware | Creates per-thread isolated directories (workspace, uploads, outputs) | Every conversation gets its own filesystem. No cross-contamination. |
| 2 | UploadsMiddleware | Injects newly uploaded files into conversation context | PDFs, Excel, PPTs auto-converted to Markdown before the LLM sees them |
| 3 | SandboxMiddleware | Acquires sandbox environment for code execution | Can be local filesystem or Docker-based isolation |
| 4 | SandboxAuditMiddleware | Logs sandbox file operations for security auditing | Production safety net — every file read/write gets logged |
| 5 | DanglingToolCallMiddleware | Patches orphaned tool calls that lack responses | Prevents LLM confusion from interrupted tool executions |
| 6 | LLMErrorHandlingMiddleware | Catches LLM provider errors, retries with backoff | Your agent doesn't crash because OpenAI returned a 500 |
| 7 | ToolErrorHandlingMiddleware | Converts tool exceptions into proper ToolMessages | The LLM sees "tool X failed because Y" instead of a raw traceback |
| 8 | SummarizationMiddleware | Compresses conversation history when approaching token limits | Configurable trigger thresholds, uses a separate cheap model |
| 9 | TodoMiddleware | Tracks multi-step task progress in plan mode | Real-time task status updates streamed to the UI |
| 10 | TokenUsageMiddleware | Tracks per-turn and cumulative token usage | Cost monitoring baked into the runtime |
| 11 | TitleMiddleware | Auto-generates conversation titles after first exchange | UX polish — no "Untitled" threads |
| 12 | MemoryMiddleware | Queues conversations for async memory extraction | Memory updates are debounced to minimize LLM calls |
| 13 | ViewImageMiddleware | Injects image data for vision-capable models | Only activates when the model config has `supports_vision: true` |
| 14 | LoopDetectionMiddleware | Detects and breaks repetitive tool call loops | P0 safety: warn at 3 repeats, force-stop at 5 |
| 15 | SubagentLimitMiddleware | Truncates excess parallel task calls | Hard cap of N per turn — excess calls silently discarded |
| 16 | DeferredToolFilterMiddleware | Hides deferred tool schemas from model binding | Dynamic tool loading — not all tools visible at all times |
| LAST | ClarificationMiddleware | Intercepts clarification requests and interrupts execution | Must be last to catch any tool calls that need user input |

### Why This Matters

Most agent frameworks treat middleware as an afterthought. DeerFlow makes it the primary extension point. Want to add rate limiting? Write a middleware. Want to inject custom context? Middleware. Want to audit every LLM call? Middleware.

The ordering constraints are the real engineering here. Getting them wrong causes subtle bugs:
- If SummarizationMiddleware runs *after* MemoryMiddleware, you might summarize away information that memory extraction hasn't processed yet
- If ClarificationMiddleware isn't last, a downstream middleware might act on a message that should have been intercepted for user clarification
- If ThreadDataMiddleware doesn't run first, nothing else has a thread_id to work with

---

## SubAgent System: Parallel Task Execution

DeerFlow's subagent system is essentially a work-stealing thread pool for AI tasks.

```python
# Two thread pools: schedulers and executors
_scheduler_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-scheduler-")
_execution_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-exec-")
```

**How it works:**

1. The lead agent calls `task(description="...", prompt="...", subagent_type="general-purpose")`
2. The executor spins up a new agent in a background thread
3. The subagent gets its own sandbox, tools, and context — but shares the parent's thread data
4. Max 3 concurrent subagents per turn, 15-minute timeout
5. Results stream back via SSE events

**The Hard Concurrency Limit**

This is enforced at two levels:
- **SubagentLimitMiddleware** silently truncates excess `task()` calls in the model's response
- **The system prompt** explicitly instructs the model to count sub-tasks and batch them: "I have 6 sub-tasks, limit is 3, I'll launch 3 now and 3 next turn"

The prompt engineering here is remarkably detailed — over 200 lines dedicated to teaching the model how to decompose, batch, and synthesize parallel work. This is not "just call task()" — it's a full orchestration protocol.

---

## Memory System: LLM-Powered Persistent Context

DeerFlow's memory is more structured than most implementations I've seen.

### Memory Schema

```json
{
  "version": "1.0",
  "lastUpdated": "2026-04-05T12:00:00Z",
  "user": {
    "workContext": {"summary": "...", "updatedAt": "..."},
    "personalContext": {"summary": "...", "updatedAt": "..."},
    "topOfMind": {"summary": "...", "updatedAt": "..."}
  },
  "history": {
    "recentMonths": {"summary": "...", "updatedAt": "..."},
    "earlierContext": {"summary": "...", "updatedAt": "..."},
    "longTermBackground": {"summary": "...", "updatedAt": "..."}
  },
  "facts": [
    {"id": "...", "content": "...", "category": "...", "confidence": 0.9, "createdAt": "..."}
  ]
}
```

Key design decisions:

1. **Structured memory over flat text** — separate slots for work context, personal context, top-of-mind, and historical context at three time horizons
2. **Confidence-scored facts** — each fact has a 0-1 confidence score. Low-confidence facts can be pruned or verified.
3. **Debounced updates** — memory extraction doesn't happen after every message. It batches conversation chunks and processes them asynchronously.
4. **File-based storage with mtime caching** — simple JSON files, but with modification-time based cache invalidation to avoid unnecessary reads.

### Memory vs. OpenClaw vs. Claude Code

| Feature | DeerFlow | OpenClaw | Claude Code |
|---------|----------|----------|-------------|
| Storage | JSON files | MEMORY.md (markdown) | CLAUDE.md |
| Structure | Hierarchical (user/history/facts) | Flat markdown with sections | Flat rules |
| Updates | LLM-extracted, debounced, async | Manual + agent-written | Manual only |
| Confidence | Per-fact confidence scores | No | No |
| Multi-agent | Per-agent isolated memory | Per-workspace | Per-project |

---

## Loop Detection: A P0 Safety Feature

This is the kind of thing that separates production systems from demos.

```python
_DEFAULT_WARN_THRESHOLD = 3  # inject warning after 3 identical calls
_DEFAULT_HARD_LIMIT = 5      # force-stop after 5 identical calls
_DEFAULT_WINDOW_SIZE = 20    # track last N tool calls
```

The algorithm:
1. After each model response, hash all tool calls (name + args, order-independent)
2. Track hashes in a sliding window per thread (LRU-evicted at 100 threads)
3. At 3 identical calls: inject a system message "you are repeating yourself — wrap up"
4. At 5 identical calls: strip all tool_calls from the response, forcing a text answer

The hash function normalizes tool call order so `[search("A"), read("B")]` and `[read("B"), search("A")]` produce the same hash. This prevents the model from "evading" detection by reordering its calls.

---

## Sandbox System: Virtual Path Translation

DeerFlow's sandbox isolates each thread's file operations through virtual path translation:

```
Virtual Path                    →  Physical Path
/mnt/user-data/workspace/       →  ~/.deerflow/threads/{thread_id}/workspace/
/mnt/user-data/uploads/         →  ~/.deerflow/threads/{thread_id}/uploads/
/mnt/user-data/outputs/         →  ~/.deerflow/threads/{thread_id}/outputs/
/mnt/skills/                    →  deer-flow/skills/
```

Two providers:
- **LocalSandboxProvider** — filesystem isolation only (bash disabled by default for safety)
- **AioSandboxProvider** — full Docker container isolation with bash access

The `str_replace` tool has a clever per-path mutex:

> "File-write safety: str_replace serializes read-modify-write per (sandbox.id, path) so isolated sandboxes keep concurrency even when virtual paths match"

This prevents race conditions when multiple subagents try to edit the same file simultaneously.

---

## IM Bridge: Native Chat Platform Integration

DeerFlow natively supports Feishu, Slack, and Telegram as first-class messaging channels.

The Feishu integration is the most sophisticated — it streams responses via `runs.stream()` and updates a single in-thread card in place (patching the same `message_id` until the run finishes). Slack and Telegram still use the simpler `runs.wait()` response path.

This makes DeerFlow usable as a "deploy once, message anywhere" system — same agent, same memory, accessible from your team's existing chat tools.

---

## Clarification System: Ask Before Acting

The clarification system is a full interruption mechanism, not just a prompt hint.

The ClarificationMiddleware sits at the end of the chain (this is critical). When the agent calls `ask_clarification()`:
1. The middleware intercepts the tool call
2. Execution halts — no further tools run
3. The question is surfaced to the user
4. On user response, execution resumes from where it paused

The system prompt dedicates ~150 lines to teaching the model **when** to ask for clarification:
- Missing information ("create a web scraper" without specifying the target)
- Ambiguous requests (multiple valid interpretations)
- Destructive operations (delete, overwrite)

The workflow priority is explicitly stated: **CLARIFY → PLAN → ACT**. Never start working and clarify mid-execution.

---

## What DeerFlow Gets Right

1. **Middleware-first architecture** — makes the system extensible without touching core agent logic
2. **Loop detection as P0** — production agents will loop. Detecting and breaking loops prevents cost blowup and poor UX.
3. **Structured memory with confidence** — most agent memory is "append text to a file." DeerFlow's hierarchical schema with confidence scores is meaningfully better.
4. **Subagent batching protocol** — the 200+ line system prompt section teaching the model to batch parallel work is serious prompt engineering
5. **Virtual path isolation** — clean abstraction over local vs. Docker sandboxes

## What Could Be Better

1. **No cost budgets** — token tracking exists but there's no per-thread or per-user spending limit
2. **Memory is single-file JSON** — works for personal use, won't scale to multi-tenant deployments
3. **No built-in eval framework** — no way to benchmark agent performance on repeatable tasks
4. **Subagent depth is 1** — subagents can't spawn their own subagents. For deeply recursive tasks, you're limited to a flat decomposition.
5. **Security model is advisory** — the security notice says "improper deployment may introduce security risks" but there's no mandatory auth or sandboxing

---

## Key Takeaways for Agent Builders

1. **Invest in middleware.** If you're building an agent system, a middleware chain is the highest-leverage architectural decision you'll make. Every cross-cutting concern becomes a composable, testable unit.

2. **Loop detection is not optional.** Every production agent will eventually get stuck. Build the circuit breaker before it costs you $500 in API calls.

3. **Memory needs structure.** "Append everything to a text file" works for prototypes. For anything real, you need categories, confidence scores, and retention policies.

4. **Teach the model to orchestrate.** DeerFlow's subagent prompt is 200+ lines of very specific instructions. Parallel task decomposition doesn't emerge from "be helpful" — it requires explicit protocol design.

5. **Separate your control plane from your data plane.** LangGraph Server (stateful, long-running) vs. Gateway API (stateless REST) is a textbook microservice split that makes scaling tractable.

---

*This teardown is part of [awesome-ai-anatomy](https://github.com/NeuZhou/awesome-ai-anatomy) — a series of Staff Engineer-level deep dives into how production AI systems are actually built.*
