# Design Document: LLM-Based Coding Agent (aider/opencode class)

| | |
|---|---|
| **Status** | Draft for review |
| **Owner** | Engineering |
| **Audience** | Principal/Staff engineers, senior ICs, engineering managers |
| **Version** | 2.0 |
| **Date** | 2026-08-18 |
| **Supersedes** | v1.0 (2026-08-18) |

---

## 0. Executive Summary

We are building a terminal-native, LLM-based coding agent in the class of
aider, opencode, and Claude Code. The product is an **agentic pair programmer**:
it reads a repository, understands intent, edits files, runs commands, verifies
its own work, and records everything in git.

This document is the architecture reference. It distills the hard-won
engineering of the field's production systems — aider's repo-map and
edit-format design, Claude Code's tool loop and sandboxing, GitHub Copilot's
agent-mode tooling, OpenAI Codex's containerized feedback loop, Cursor's
layered runtime — plus the academic lineage (ReAct, Reflexion, SWE-agent,
SWE-bench) — into a concrete design for our own implementation.

The central engineering thesis, stated up front:

> **An LLM coding agent is not a model wrapped in a chat loop. It is a
> reliability system built around a non-deterministic model.** The scaffold —
> context selection, edit application, verification, reflection, safety — is
> where the product is won or lost. Research confirms this: a source-code
> taxonomy of 13 open-source coding agents (arXiv:2604.03515) found that
> scaffolds converge wherever external constraints dominate (edit formats,
> tool capabilities, execution isolation) and diverge on open design questions
> (context compaction, state management, multi-model routing). A weaker model
> with a better harness routinely beats a stronger model with a worse one.

The ten highest-leverage design decisions (detailed in §21):

1. **Edit formats as versioned closed protocols** between prompt and parser,
   tested in lockstep with golden transcripts (D1).
2. **Repo map**: tree-sitter symbol graph → PageRank → token budget, hinted
   by the current message (D2, D3).
3. **Reflection over re-prompting**: parse-fail-reflect is the primary
   reliability loop, with bounded attempts and no-progress detection (D8).
4. **Git is the undo log**: commit-before-edit + auto-commit-after-edit (D5).
5. **Model capabilities are data, not code** (D4).
6. **Search, don't index**: agentic grep/glob over embedding retrieval for
   codebase navigation (D9).
7. **Dual edit paths**: text edit protocols (universal) + tool/function
   calling (fast path) behind one interface (D7).
8. **Context budget is an explicit, measured resource**: 1/8 repo map, binary
   search on content, recursive summarization, dynamic fence selection (D2).
9. **Verification loop**: lint and test feedback fed back to the model
   automatically (D8).
10. **Layered permission + sandbox model**: read-only default, tiered
    approvals, OS-level sandbox as the enforcement backstop (D10).

---

## 1. Context and Problem Statement

The last few years produced a new class of developer tooling: LLM-based coding
agents. aider (2023), Cursor (2023), GitHub Copilot agent mode (2025), Claude
Code (2025), and OpenAI Codex (2025) turned the LLM from a chat companion into
an agent that reads a repository, understands intent, edits files, runs
commands, and verifies its own work. By mid-2026 the field has reached
extraordinary capability levels: frontier agents resolve >75% of SWE-bench
Verified tasks (up from ~13% in early 2024), and OpenAI has reported a single
autonomous Codex run lasting over seven hours.

The core engineering challenge is **not** "calling an LLM" — that is a solved
problem. The challenge is building a **reliable, stateful, verifiable
execution loop** around an intrinsically non-deterministic model:

- The model's output must be turned into **precise, safe file edits** — not
  prose. Models that are brilliant at code still emit malformed diffs,
  misquoted context blocks, and truncated output.
- The agent must decide **what context to send** when context windows are
  bounded and expensive. Exploration is the largest and most variable cost in
  agentic coding: a moderate multi-file task consumes 40–80K tokens, of which
  exploration alone is 20–35K.
- Errors are inevitable (malformed output, failed matches, context exhaustion,
  provider failures, agent runs that "fix" in the wrong direction) and the
  agent must **self-correct** rather than crash or loop.
- The agent touches a developer's source code, so it must be **safe,
  reversible, and transparent** — git as the safety net, OS-level sandboxing
  as the enforcement backstop, permissions as the UX.
- The UX must feel like pairing with a senior engineer, not like a brittle
  script that sometimes edits files.

### 1.1 Why this is a systems problem

The field's own evidence says the scaffold is the product:

- **SWE-bench scores are harness scores as much as model scores.** The same
  model, scaffolded differently, moves double digits on the leaderboard.
  Benchmarks that do not control for harness are hard to interpret; tool
  definitions, the system prompt, iteration limits, and error surfacing all
  change the outcome ("AI Agents That Matter", arXiv:2407.01502).
- **SWE-agent** (arXiv:2405.15793) demonstrated that the *agent-computer
  interface* is a first-class variable: redesigning the interface beats
  upgrading the model at equal cost.
- **ContextBench** (arXiv:2602.05892) showed *finding* relevant code and
  *using* it correctly are different skills — an agent can retrieve well and
  still fail to apply the retrieved context.
- **Claude Code** explicitly replaced semantic-search RAG with agentic
  grep/glob after internal benchmarks showed the simpler approach won on both
  quality and operational complexity.
- **"Inside the Scaffold"** (arXiv:2604.03515), a source-code taxonomy of 13
  open-source coding agents, found 11 of 13 compose multiple loop primitives
  (ReAct, generate-test-repair, plan-execute, multi-attempt retry, tree
  search) rather than relying on a single control structure.

This document is the design reference for building our agent. It is written to
be both an implementation blueprint and a review artifact: each component
lists responsibilities, interfaces, trade-offs, and the failure modes it must
handle, grounded in what the field has learned.

---

## 2. Goals and Non-Goals

### 2.1 Goals

- **Edit files reliably.** Given a natural-language request, produce correct,
  reviewable changes to a user's repository with a high success rate on the
  first attempt and high *eventual* success via self-correction.
- **Be model-agnostic.** Support many LLM providers (OpenAI, Anthropic,
  OpenRouter, local models) with per-model capability negotiation.
- **Fit the context budget.** Stay within the model's context window by
  assembling only the most relevant subset of the repository (repo map, files
  in chat, summarized history). Treat context as a budgeted, measured
  resource, not an afterthought.
- **Never lose work.** Every edit is recoverable: dirty-commit before
  changes, auto-commit after, user-facing undo, dry-run modes.
- **Close the loop.** After editing, verify via linting and tests, and feed
  failures back to the model automatically (reflection).
- **Feel fast.** Streaming output, minimal startup latency, background
  warm-up of imports and caches, prompt-cache friendliness.
- **Be safe by default.** Read-only until permission; sandbox as the
  enforcement backstop, not a prompt.

### 2.2 Non-Goals

- Replacing the developer's judgement. The agent is an assistive pair
  programmer; fully unattended, long-horizon autonomous operation (Codex
  cloud-style "fire and forget") is future work (§23).
- Guaranteeing *correct* code; we guarantee *verifiable* changes (lint/tests)
  and *reversible* changes (git).
- Providing a code search engine (we integrate tree-sitter/grep/ripgrep as
  libraries, we do not build one).
- Supporting every possible model capability on day one; capability detection
  is progressive.
- Replacing the user's IDE or editor; we are a terminal-native agent that
  complements editors via file watching and watch-mode markers.

---

## 3. Design Tenets (Principles)

1. **Text in, structured edits out.** The LLM communicates in prose with
   machine-parseable edit blocks. We never let the model write files directly;
   every byte written passes through an application layer that validates and
   falls back gracefully. (This is the distinction between an "AI assistant"
   and an "AI agent": tools turn a model into an execution loop.)
2. **The conversation is the state.** The chat transcript (system prompt +
   history + current turn) is the agent's memory. Compression (summarization)
   and pruning (file add/drop) are first-class operations.
3. **Git is the safety net, not an afterthought.** Commit-before-edit,
   auto-commit-after-edit, and undo are core UX, and `--no-git` must be a
   supported degraded mode.
4. **Feedback loops beat prompting.** When an edit fails to apply, the failing
   block and the actual file content are sent back to the model, not an error
   dialog. When lint or tests fail, the output is reflected. The agent
   *learns from its environment* (Reflexion's "verbal reinforcement").
5. **Every capability is negotiated, not assumed.** Models differ in
   streaming, function calling, reasoning output, context size, edit formats.
   The model registry encodes these facts.
6. **Fast by default.** Lazy imports, background warm-up, prompt caching,
   streaming — latency is a feature.
7. **Search, don't index.** For codebase navigation, prefer agentic grep/glob
   and tree-sitter symbol extraction over embedding indexes: simpler, more
   secure (no code leaves the machine for indexing), no sync to maintain, and
   — per Claude Code's experience — empirically equal or better.
8. **Least privilege, layered enforcement.** Permission prompts are UX; the
   OS-level sandbox is the backstop. A permission dialog that can be
   auto-approved is not a security boundary.
9. **Start simple, scale intelligently** (Anthropic, *Building Effective
   Agents*). Single-agent, text-edit loop first; workflows and multi-agent
   patterns only when measurement justifies them.
10. **Benchmarks are a filter, internal evals are the verdict.** Public
    leaderboards (SWE-bench Verified, Aider Polyglot) measure capability
    ceilings under a protocol; our own task-level evals on real repos decide.

---

## 4. High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                        INTERACTION LAYER                           │
│   CLI (prompt_toolkit) · GUI · scripted/API entry                  │
│   InputOutput: colors, prompts, confirmations, notifications       │
└──────────────────────────────┬─────────────────────────────────────┘
                               │ user message
┌──────────────────────────────▼─────────────────────────────────────┐
│                     ORCHESTRATION LAYER                            │
│   Coder (base) ── factory ──> edit-format subclasses               │
│   run() → run_one() → send_message() → apply_updates() → commit    │
│   reflection loop · summarizer thread · cache warmer thread        │
│   commands (slash) · file watcher · voice · clipboard              │
└──────────┬───────────────────────────────┬─────────────────────────┘
           │                               │
┌──────────▼───────────┐         ┌─────────▼─────────────────────────┐
│  MODEL LAYER         │         │  REPOSITORY LAYER                 │
│  litellm abstraction │         │  GitRepo: tracking, diffs,        │
│  streaming · retry   │         │  auto-commit, undo, dirty commit  │
│  token counting      │         │  .gitignore/.aiderignore handling │
│  cost accounting     │         └───────────────────────────────────┘
└──────────┬───────────┘
           │
┌──────────▼─────────────────────────────────────────────────────────┐
│  CODE ANALYSIS LAYER                                               │
│  RepoMap (tree-sitter tags → PageRank → token-budgeted map)        │
│  Linter (ruff, eslint, etc.) · test runner                         │
│  URL scraping · image handling · MCP client                        │
└────────────────────────────────────────────────────────────────────┘
```

**Component responsibilities at a glance**

| Layer | Responsibility |
|---|---|
| Interaction | Terminal/GUI I/O, input history, colors, confirmations, streaming render |
| Orchestration | The agent loop: prompt assembly, LLM call, response interpretation, edit application, verification, reflection |
| Model | Provider abstraction, streaming, retries, token counting, cost, caching |
| Repository | Git integration: file tracking, commits, undo, repo layout |
| Code Analysis | Repo map, linting, test execution, symbol extraction, tool execution |
| Security | Permission tiers, sandbox (Seatbelt/Docker), secrets hygiene, MCP authorization |

### 4.1 How the reference systems decompose (what we borrow)

| System | Core pattern | What we adopt |
|---|---|---|
| **aider** | Single-process Python loop; text edit formats; repo map; git-native | Repo map, edit-format family, reflection loop, dirty-commit/undo |
| **Claude Code** | `while(tool_call)` loop; 8 core tools; "search don't index"; sandboxing; hooks; subagents | Tool loop shape, sandbox model, permission tiers, MCP with deferred tool definitions |
| **GitHub Copilot agent mode** | Orchestrator over tools (read_file with line ranges, edit_file, run_in_terminal); summarized workspace structure; error-recovery loop | Tool description discipline, workspace-structure prompt, terminal approval UX |
| **OpenAI Codex** | Cloud agent: isolated container, no internet, test loop until green, PR out | Feedback-loop primacy ("run tests until they pass"), container isolation for CI-grade runs |
| **Cursor** | Layered runtime (Tab / Agent / Cloud agents / Bugbot); instructions+tools+model; parallel agents on git worktrees | Rules hierarchy (AGENTS.md), per-model tool tuning, parallel-agent isolation via worktrees |
| **opencode** | Client/server split (TUI ↔ agent core); Go/Bun; MCP integration; permission system | Decoupled I/O from agent core (testable), MCP client design |

---

## 5. Core Runtime: The Agent Loop

### 5.1 The loop, and where it comes from

Every production coding agent runs some variant of the **agentic loop**:
*think → act → observe → repeat*. The lineage is well established:

- **ReAct** (Yao et al., ICLR 2023): interleaving reasoning (Thought) and
  acting (Action/tool calls), with Observations injected by the harness.
  Foundational result: interleaving beats pure chain-of-thought or pure
  acting. Our loop inherits this shape.
- **Reflexion** (Shinn et al., NeurIPS 2023): *Act → Critique → Revise →
  Act* — verbal self-critique from environment feedback (test failures).
  Our reflection mechanism is Reflexion specialized to coding, with a
  bounded critic budget to avoid its characteristic failure mode (looped
  self-flagellation).
- **Plan-Execute**: a planner decomposes long tasks, an executor runs steps.
  This is the aider `architect` mode and Devin's planner/executor split.
- **SWE-agent**: the interface between agent and computer is a first-class
  design variable.

The 2026 taxonomy of open-source scaffolds (arXiv:2604.03515) enumerates five
composable loop primitives:

| Primitive | Description | Used by |
|---|---|---|
| ReAct | Thought/Act/Observe interleaving | Baseline of most agents |
| Generate-test-repair | Write code, run tests, fix failures | Codex cloud, aider reflection |
| Plan-execute | Planner decomposes, executor runs | Architect mode, Devin |
| Multi-attempt retry | Retry on failure with bounded budget | Copilot agent mode |
| Tree search | Explore multiple candidate paths | Research-stage agents |

**Decision (D11):** our default loop is **ReAct-shaped generate-test-repair**
— the model can call tools (read, grep, shell) and emit edits; failures
(lint, tests, parse) are reflected back; attempts are bounded
(`max_reflections = 3` plus a no-progress detector). Plan-execute is layered
on top via architect mode (§11).

### 5.2 The per-turn loop

```
user message
  ├─ preprocess: slash commands? file mentions? URLs? clipboard?
  ├─ assemble prompt (see §6) and send to LLM (streaming)
  ├─ capture reply (text and/or tool/function call)
  ├─ postprocess reply:
  │    ├─ detect file mentions → offer to add files → reflect
  │    ├─ reply_completed hook (subclass behavior)
  │    ├─ parse reply into structured edits (get_edits)
  │    ├─ dry-run edits → permission checks → apply to disk
  │    ├─ auto-commit changed files (git)
  │    ├─ lint → errors? offer to fix → reflect
  │    ├─ run suggested shell commands → paste output → reflect
  │    └─ run tests → failures? offer to fix → reflect
  └─ loop until no reflection is requested (max 3 reflections)
```

Tool-calling agents (Claude Code, Copilot agent mode) run the same shape,
with tool results replacing "text reply" as the observation channel:

```
while True:
    response = call_model(system, messages, tools)
    messages.append(assistant_response)
    if response.stop_reason == end_turn: return extract_text(response)
    if response.stop_reason == tool_use:
        results = execute_tool_calls(response)
        messages.append(user(results))     # observations
        # loop continues; model decides when done
```

Key insight from the field: **the LLM is stateless, the conversation is
stateful.** Everything the agent "remembers" lives in the message stream —
which is why context management (§6) and compaction are load-bearing
machinery, not nice-to-haves.

### 5.3 Key state

| State | Purpose |
|---|---|
| `done_messages` | Summarized conversation history (compressed) |
| `cur_messages` | Current turn's messages (user + assistant replies) |
| `abs_fnames` | Files in chat (editable) |
| `abs_read_only_fnames` | Files in chat (read-only) |
| `aider_edited_files` | Files changed this turn |
| `reflected_message` | Pending follow-up message to re-send to the model |
| `total_cost` / token counters | Session accounting |
| `commit_before_message` | HEAD sha before the turn (undo anchor) |
| `reflection_count` / `malformed_count` | Degenerate-loop detection counters |

### 5.4 Coder subclasses and the factory

`Coder.create()` is a strategy factory keyed on `edit_format`. Each subclass
implements exactly two behaviors:

1. **`get_edits()`** — parse the model's reply into structured edits
   (`(path, original, updated)` tuples or higher-level actions).
2. **`apply_edits()`** — apply those edits to disk.

The edit format is negotiated from the model registry (`main_model.edit_format`)
and can be overridden by the user (`--edit-format`). Formats (aider's history
shows a clear evolution):

| Format | Description | Trade-offs |
|---|---|---|
| `whole` | Model returns full file content | Simple, robust to small files; wasteful for large files, loses context |
| `diff` / SEARCH/REPLACE | `<<<<<<< SEARCH` / `=======` / `>>>>>>> REPLACE` blocks | Precise, cheap; brittle when model invents non-matching context |
| `diff-fenced` | SEARCH/REPLACE inside code fences | Handles models that refuse raw blocks |
| `udiff` | Unified diff hunks | Natural for models trained on diffs; parsing cost |
| `patch` | Structured actions (update/append/insert/delete) | Most machine-like; best for function-calling models |
| `editor-*` | Format for a separate "editor" model | Enables weak/cheap editor models |
| `architect` | Two-model: plan then edit | Best quality; 2× cost, 2× latency |

**Design decision (D1):** the edit format must be a *closed protocol between
system prompt and parser*, versioned and tested in lockstep. Every new format
ships with (a) a system prompt fragment, (b) a parser, (c) an error-path
reflector (see §8), and (d) golden tests with real model transcripts.

**Why formats matter (evidence):** aider's polyglot benchmark folds
edit-application into the grade — a model that solves a problem in prose but
emits a malformed diff fails the task, which is precisely how real agents die
in production. SWE-Edit (2026) showed that changing only the edit *interface*
(separate viewer/editor tools) materially improves SWE-agent performance
without touching the model. The interface is the product.

---

## 6. Context Engine: What to Send to the LLM

The prompt is assembled in **ordered sections** with a token budget:

```
system prompt (instructions + edit format protocol + platform facts)
example conversations (few-shot, format-specific)
done_messages (summarized history)
repo map (ranked symbol graph, token-budgeted)
read-only files (full content, fenced)
chat files (full content, fenced)
current messages
system reminder (only if token budget allows)
```

Budgeting rules (aider-proven defaults, tunable):

- **Repo map**: default 1/8 of the model's context window
  (`get_repo_map_tokens()`), with an 8× multiplier when no files are in chat.
- **Content truncation**: the map is binary-searched on token count to land
  within ~15% of the budget.
- **Files in chat**: full content, always (they are the editable set — the
  contract must be legible to the model).
- **History**: recursive summarization on overflow (§6.3).
- **System reminder**: included only if budget allows; otherwise appended to
  the last user message.

### 6.1 The repository map (RepoMap)

Sending an entire repository is impossible; sending nothing is useless. The
RepoMap solves this with three stages:

1. **Symbol extraction.** Tree-sitter parses every file and extracts
   `name.definition.*` tags (classes, functions, methods) and
   `name.reference.*` tags (symbol usage). Tree-sitter provides
   syntax-robust parsing across 50+ languages without running user code.
   Results are cached in a SQLite-backed disk cache keyed by file mtime
   (incremental rebuilds only). For languages with incomplete tree-sitter
   grammars (e.g., C++), fall back to Pygments lexer tokenization.
2. **Ranking.** Build a graph (files as nodes, references as edges,
   MultiDiGraph) and rank with personalized PageRank (NetworkX). The
   personalization weights encode task relevance:
   - files already in chat: **50×** boost;
   - identifiers mentioned in the conversation: **10×**;
   - named symbols 8+ characters long: **10×**;
   - edge weights are square-rooted reference counts (prevents one hot
     symbol from dominating).
3. **Token budgeting.** Greedily add the highest-ranked (file, symbol) pairs
   until the `map_tokens` budget is exhausted, then render as a compact
   AST-aware tree with lines capped at ~100 characters.

**Design decision (D2):** the map is *hinted by the current message*. The
agent extracts identifiers from the user's text (word splitting + filename
matching) and biases the map toward those symbols. This one mechanism
(cheap, no embeddings, no index) delivers most of the relevance of semantic
search — the same conclusion Claude Code reached after *shipping* embedding
RAG and replacing it with agentic search.

**Design decision (D3):** refresh policy is `auto` — the map is rebuilt when
files change and only when a new turn needs it, with an explicit `force`
flag for `/map` and model switches. Cache the ranked result keyed on file
mtime hashes.

### 6.2 Agentic search: "search, don't index"

For navigation beyond the map, the agent gets read tools — `grep` (ripgrep),
`glob`, `read_file` with line ranges (returning an outline for large files so
the model can re-request the precise slice). This is Claude Code's pattern
and Copilot agent mode's `read_file` design. Rationale:

- No index to build, sync, or trust; zero privacy cost (code never leaves
  the machine for embedding).
- The model drives the search iteratively — a *search loop* rather than a
  one-shot retrieval, which is empirically more effective than top-k
  embedding recall for repository-scale tasks (ContextBench: finding and
  using code are separate skills, and the agentic loop exercises both).
- Embeddings remain an option for repositories too large for grep
  (future work, §23).

### 6.3 File selection (files in chat)

Files in chat are sent in full, fenced. This is a deliberate, simple contract:
*the model can only edit files that are in the chat*. Selection is user-driven
(`/add`, `/drop`) plus automatic:

- **File mentions**: if the model (or user) names a file not in chat, the
  agent offers to add it, then reflects the offer result back to the model
  (`check_for_file_mentions`). This is lightweight RAG: the model tells us
  what it needs next.
- **Guardrails**: warn when >4 files or >20K tokens accumulate ("it's best to
  only add files that need changes").
- **Identifiers**: symbol mentions in the message are resolved to filenames
  via a basename index.

### 6.4 History management (summarization and compaction)

The transcript grows unboundedly; the context window does not. `ChatSummary`
compresses `done_messages` with a cheap "weak" model in a **background thread**
while the main model works. When the next turn starts, the summary is swapped
in only if the history was unchanged during compression (compare snapshots;
discard stale summaries).

The compression algorithm is **recursive head/tail summarization**:

1. Split the transcript at an assistant-message boundary into head and tail.
2. Send the head to the weak model for summarization; keep the tail verbatim
   (recent context stays lossless).
3. If the result still exceeds budget, recurse with increasing depth.

This is the pattern used by aider and by Cursor ("when the context window
fills, trigger a summarization step to give the agent a fresh window with a
summary of its state"), and compaction in Claude Code. Design rules:

- Summaries must preserve *decisions*, *file paths*, and *what was tried and
  failed* — not prose.
- Compaction happens before the model needs the space, never mid-turn
  (except as a last resort, which degrades the current task).
- The full transcript must remain recoverable (persisted session log) — the
  summary is a *view*, not a deletion.

### 6.5 Token budgeting and reminders

- Count tokens with the provider tokenizer when available.
- The system reminder ("make sure you actually make the changes...") is only
  included if the budget allows; otherwise it is appended into the last user
  message (role `user`).
- Pre-flight check: if estimated input tokens exceed the model's
  `max_input_tokens`, warn and offer actionable remediation (`/drop`,
  `/clear`) before sending.
- Long-context discipline: benchmark evidence shows models can externalize
  long-context processing through files and tools (write scratch notes,
  re-read files) — encourage the agent to do work in the filesystem rather
  than forcing everything into the window.

### 6.6 Fence selection

The code fence used to wrap file content is **chosen dynamically** to avoid
colliding with fences already present in the files (triple-backtick,
quadruple-backtick, `<source>` blocks, ...), and the chosen fence is
communicated to the model in the system prompt. This eliminates a whole class
of "the model closed my fence early" failures.

---

## 7. Model Layer

### 7.1 Capability registry

Every model is described by a `ModelSettings` record: context size, pricing,
streaming support, function-call support, reasoning tags, edit format
preference, repo-map token allowance, whether it accepts `reasoning_effort`,
etc. The registry is extensible via YAML/JSON files
(`.aider.model.settings.yml`, `.aider.model.metadata.json`) and an alias
system (`--alias`). Model roles:

| Role | Purpose | Characteristics |
|---|---|---|
| `main_model` | Primary editor; all editing tasks | Strongest model |
| `weak_model` | Commit messages, summarization, simple tasks | Cheap/fast |
| `editor_model` | Executes architect plans | Cheap/fast, strict format adherence |

**Design decision (D4):** model knowledge is *data, not code*. Shipping a new
model must not require a release; it requires a settings file. A `--models`
listing command is part of the CLI surface.

### 7.2 Provider abstraction

Use a battle-tested multi-provider client (litellm). Our code talks to a
narrow interface:

```
send_completion(messages, functions, stream, temperature) → completion
```

Wrapped concerns, in order:

1. **Retry with exponential backoff** for rate limits and transient errors,
   capped (e.g., ~300s), with user-visible "Retrying in Xs…" messages.
2. **Context-window errors**: detect `ContextWindowExceededError` distinctly —
   these are *ours* to fix (drop files, clear history), not retried.
3. **Streaming**: accumulate deltas into `partial_response_content`; render
   incrementally to the terminal (markdown stream) or GUI.
4. **Reasoning content**: models that emit `reasoning_content` (o-series,
   Claude extended thinking) get it wrapped in `<reasoning>…</reasoning>` for
   display and **stripped before parsing** (it must never pollute edit
   parsing). Carve out an explicit thinking-token budget when the model
   supports it (`set_thinking_tokens()`).
5. **Cost accounting**: usage tokens → cost per model settings → per-message
   and session totals, shown via `/cost` and after each turn.
6. **Prompt caching**: cache-control headers on stable prefixes (system
   prompt, history, files); optional background "keepalive" pings (1-token
   completions) to hold the cache warm for long sessions. Codex's agent loop
   runs on the same economics: prompt caching is what makes long agentic
   sessions affordable — design the prompt to be *cache-shaped* (stable
   prefix ordering: system → history → files, with volatile content last).

### 7.3 Long outputs and continuation

When `finish_reason == "length"` (output limit hit), for models that support
assistant-prefill, the accumulated reply is prepended as an assistant message
and the model continues — effectively unlimited output. For others, surface a
token-limit error with remediation guidance. This is a high-leverage feature
for large-file edits.

### 7.4 Per-model prompt tuning

Cursor explicitly tunes instructions and tools per frontier model; Copilot
agent mode sends a shared prompt today but plans per-model tailoring. Our
position: the *system prompt is versioned and model-keyed*. `ModelSettings`
references a prompt profile (edit-format protocol + behavioral rules) so a
model that reliably produces SEARCH/REPLACE blocks gets `diff`, one that
doesn't gets `whole` or `udiff`. This is data, not code (D4).

---

## 8. Edit Application Layer

This is where reliability is won or lost. The pipeline:

```
get_edits()            # parse reply → structured edits (may raise ValueError)
apply_edits_dry_run()  # compute new content without writing
prepare_to_edit()      # permission checks + dirty-commit safety
apply_edits()          # write to disk
```

### 8.1 Matching with graceful degradation

For SEARCH/REPLACE edits, the search text must match exactly — but models
frequently mangle whitespace, add blank lines, or elide code. The matcher
applies a strict-to-fuzzy ladder:

1. exact line match
2. match ignoring uniform leading-whitespace errors (models routinely outdent
   or over-indent whole blocks)
3. tolerate one spurious leading blank line
4. handle `...` elision markers (paired in both SEARCH and REPLACE)
5. (optional) fuzzy match via sequence similarity ≥ 0.8

**Never guess silently.** Every fallback is tried in order; if all fail, the
edit is reported as failed with the actual file content and "did you mean"
suggestions.

### 8.2 The error path is a prompt, not a dialog

On failure, construct a structured error message containing:

- the failing SEARCH/REPLACE block verbatim;
- the actual lines of the file near where the model *probably* meant to edit
  (`find_similar_lines`);
- whether the REPLACE text already exists in the file;
- instructions: "SEARCH must exactly match… Don't re-send successful blocks."

This error text becomes `reflected_message` and is sent back to the model.
Empirically, models fix their own malformed edits on the next reflection in
the large majority of cases. This single loop — *parse, fail, reflect* —
is the biggest reliability lever in the entire system.

The same pattern extends to **verification feedback**: after applying edits,
run lint; if errors, package them with the surrounding code (marked with
block characters via TreeContext) and reflect "Fix any errors below". Run
tests (`--test-cmd`); failures are reflected the same way. Copilot agent
mode's error-recovery loop and Codex's "run tests until they pass" are the
same mechanism at different autonomy levels.

### 8.3 Bounded reflection (avoiding degenerate loops)

Reflexion's failure mode is *looped self-flagellation*: the critic never
accepts a revision. Our countermeasures:

- hard cap on reflections per turn (`max_reflections = 3`);
- malformed-response and context-exhaustion counters surfaced to the user;
- **no-progress detector**: if the model re-emits the same failing block or
  the same broken command twice, stop and hand control to the user with a
  summary of what was tried;
- direction-of-failure detection: if reflections produce *more* lint errors,
  stop. (The field's warning is real: with wrong context, the recovery loop
  can make things worse — the agent "fixes" in wrong directions.)

### 8.4 Safety and permissions

- Only files **in chat** are editable; anything else is refused
  (`allowed_to_edit`).
- Read-only files are never written.
- Before the first edit of a turn, a "dirty commit" snapshots any uncommitted
  user work (with a distinctive commit message) so nothing is lost.
- `--dry-run` computes and shows edits without writing.
- Malformed-response counter and exhausted-context counter are tracked and
  surfaced (models can enter degenerate loops).

### 8.5 Verification

After edits: **auto-lint** (per-language commands; e.g., `ruff`, `eslint`)
and, optionally, **auto-test** (`--test-cmd`). Lint/test output is reflected
back to the model ("Attempt to fix lint errors?"). Linting also runs *before*
editing on a copy when a file is first added, so the model sees existing
problems as part of its context.

Linting stack (aider-proven): tree-sitter AST check for `ERROR` nodes
(language-agnostic), `compile()` for Python, selective flake8, and
user-registered custom lint commands. Design note: the tree-sitter linter
skips TypeScript (grammar incompatibilities) — use custom commands there.

---

## 9. Repository Layer (Git Integration)

| Operation | Purpose |
|---|---|
| Repo discovery | Walk up to the git root; also support `--no-git` degraded mode |
| File tracking | Tracked-file listing (respecting `.gitignore` and `.aiderignore`) |
| Commit-before-edit | `dirty_commit()` — snapshot user's uncommitted work before we touch files |
| Auto-commit | After each successful turn, commit changes with an LLM-generated message (weak model, generated from the diff) |
| Undo | `/undo` — revert to the commit made before the last edit (the commit chain is the undo log) |
| Author attribution | Optional `Co-authored-by` trailer; configurable `GIT_AUTHOR_NAME` / `GIT_COMMITTER_NAME` |
| Sanity checks | Refuse repos with unsupported index formats; warn when repo is unreadable |

**Design decision (D5):** the commit *before* editing and the commit *after*
editing are the undo mechanism. This is simpler and more robust than snapshot
files or in-memory diffs: it reuses git's battle-tested machinery, survives
crashes, and integrates with the user's existing git workflow (`git log`,
`git diff`, `git revert` all work).

**Design decision (D6):** commit messages are LLM-generated but *honest*:
generated from the actual diff (`git diff --stat` + code), never from the
conversation, so a message cannot claim work that wasn't done.

**Worktree isolation (borrowed from Cursor's parallel agents):** for any
parallel or background operation, clone the repo into a git worktree so
concurrent agents never collide on the working tree; changes merge via
branch/PR review. This is our model for future background agents (§23).

---

## 10. Command System, Tool Execution, and MCP

### 10.1 Slash commands

A slash-command system (`/add`, `/drop`, `/commit`, `/undo`, `/model`,
`/clear`, `/web`, `/lint`, `/test`, ...) provides explicit control in an
otherwise conversational interface. Requirements:

- Commands are recognized before LLM interaction; command output can become
  LLM input (e.g., `/web URL` scrapes a page into the chat context).
- Completion support (tab-completion for files, models, commands).
- A `SwitchCoder` exception signals "change model/edit-format and continue
  the session" — the orchestrator re-creates the Coder via the factory,
  preserving chat state. This is how `/model`, `/chat-mode`, and
  `/architect` work at runtime.
- Suggested shell commands: the model can emit `bash` blocks; the user
  approves execution; output is pasted back into the conversation and
  reflected. This gives the agent a *read* capability beyond static files
  (run tests, list directories) without giving it free rein.

### 10.2 Tool calling (function calling)

Beyond text edits, expose tools to the model via function calling:

| Tool | Purpose | Notes |
|---|---|---|
| `read_file` | Read a file (or line range; outline for large files) | Copilot-style line-range contract |
| `grep` | ripgrep search with line context | The "search don't index" workhorse |
| `glob` | Filename pattern search | |
| `bash`/`run_in_terminal` | Execute shell commands | Always approval-gated; sandboxed (see §12) |
| `apply_edits` | Apply structured edits | The text-format parser behind a tool |
| `web_fetch` | Fetch a URL (scraped to markdown) | User-approved; prompt-injection risk accepted deliberately |
| `task` (subagent) | Spawn a child agent | §11.2 |

When tool-call responses arrive (`partial_response_function_call`), parse
arguments (leniently — repair truncated JSON), execute, and return results as
a new message.

**Tool description discipline (from Copilot agent mode's experience):** each
tool ships with detailed, tested instructions for *when and how* to use it.
"Read the contents of a file. You must specify the line range you're
interested in, and if the file is larger, you will be given an outline of the
rest…" — tool descriptions are prompt engineering, and they are evaluated as
such (§19).

**Design decision (D7):** text-edit protocols and tool-calling can coexist in
the same product: the edit-format parser is the fallback that works with any
model; function calling is the fast path when the model supports it. Keep
both behind the same `get_edits`/`apply_edits` interface.

### 10.3 MCP (Model Context Protocol)

MCP is the industry-standard tool interface (adopted across OpenAI, Google,
Microsoft in 2025, now governed under the Linux Foundation; ~9K-star spec
repo). Architecture:

- **Client-host-server**: the host (our agent) runs one client per server;
  servers expose **tools** (actions), **resources** (context data), and
  **prompts** (templates); clients expose elicitation (user interaction).
- **Transport**: JSON-RPC 2.0 over stdio (local processes), HTTP+SSE, or
  streamable HTTP (remote servers).
- **Capability negotiation**: `initialize` exchanges supported features;
  discovery via `tools/list`; execution via `tools/call`.
- **Authorization**: remote MCP servers use OAuth 2.0 (protected-resource
  metadata + authorization-server discovery, `auth.md`); local servers are
  trusted by configuration.

Adoption rules for our agent:

- **Deferred tool definitions**: tool schemas are large; only include tool
  *names* (with search) in the prompt until the model requests a specific
  tool's schema (Claude Code's approach). This keeps MCP's context cost
  proportional to usage, not registration.
- **Scoped capabilities**: connect servers explicitly; never auto-trust
  third-party servers.
- **Security**: MCP servers are attack surface — the mcp-server-git CVEs
  (2025) showed tool-wrapped git commands need argument validation
  (reject `-`-prefixed refs), path-boundary checks, and capability minimalism
  (the `git_init` tool was *removed* rather than patched). Treat each MCP
  tool as a new tool in the permission system.

---

## 11. Multi-Agent Orchestration

### 11.1 Architect/Editor (two-model, aider style)

- A strong "architect" model plans and writes prose with code sketches.
- After approval, a second "editor" model (often cheaper/faster) receives the
  plan as its prompt, with a clean chat history, and applies edits in a
  restricted edit format.
- The editor's repo map is disabled (the plan is the context); costs and
  commit hashes are merged back into the architect's session.

This is the Plan-Execute primitive (§5.1). The editor's job is mechanical
adherence to the plan's edit format — the separation lets us tune the editor
model *only* for edit reliability (SWE-Edit's finding: an editor subagent
trained only on edit-application improves end-to-end results).

### 11.2 Subagents (Claude Code pattern)

A `task` tool spawns a child agent with its **own context window**:

- **Fan-out/aggregate**: one agent decomposes, many agents work in parallel
  on isolated subtasks, the parent merges results.
- **Router/specialists**: a router dispatches to specialist agents
  (e.g., per-language or per-concern).

Design rules:

- Subagent output returns as a tool result (bounded, summarized) — the parent
  never inherits the child's full context.
- Subagents run in isolated sandboxes/worktrees (parallel safety, §12).
- Cap subagent depth and total spawned work; the parent remains the
  accountability boundary.

### 11.3 Cloud-style autonomous mode (Codex pattern, future)

Codex's architecture is instructive for our roadmap, not our v1: a task is
handed to an isolated container (repo + dependencies staged, **no network
access**), the agent edits and runs the test suite in a loop until green, and
returns a PR with a full command log. v1 differences: we are synchronous and
local; the *feedback discipline* (tests are the reward signal, results
reviewable) is adopted immediately; the *isolation model* becomes our
CI-grade sandbox (§12.3).

---

## 12. Security

### 12.1 Threat model

| Threat | Vector | Mitigation |
|---|---|---|
| Secrets exfiltration | Model reads `~/.ssh`, `.env`, config files | Sandbox deny-read rules; secrets hygiene; permission prompts |
| Prompt injection | Web pages, MCP content, repo files with adversarial text | Treat all fetched content as untrusted; explicit user approval for remote content; sandbox containment |
| Destructive commands | `rm -rf`, `git push --force`, `curl | sh` | Permission tiers + command classification + sandbox |
| Writes outside repo | Absolute paths, symlink escapes | Path-boundary enforcement in edit/tool layer AND sandbox |
| Tool argument injection | MCP tools wrapping CLIs (mcp-server-git CVEs) | Argument validation: reject `-`-prefixed refs, verify path resolution before execution |
| Supply chain | Tree-sitter grammars, linters, MCP servers | Pinned versions; explicit opt-in for third-party servers |

### 12.2 Layered permission model

Claude Code's model is the reference: **read-only by default**; auto-allow
safe commands (`echo`, `cat`); explicit approval for edits and commands;
persistent "don't ask again" rules; permission tiers (propose-everything →
fully autonomous). Anthropic reports sandboxing reduced permission prompts by
84% — enforcement replaces nagging.

Command classification tiers:

1. **Safe** (auto-run): `echo`, `cat`, `pwd`, `ls`, `git status/diff/log`.
2. **Read** (auto-run, sandboxed): grep, test runs with no writes.
3. **Edit** (approval): file edits, package installs.
4. **Dangerous** (always approval + warning): `rm -rf`, network mutating
   commands, `sudo`, `git push`.
5. **Blocked** (sandbox-level deny, non-overridable by default): reads of
   secret paths, writes outside the worktree, non-allowlisted network
   destinations.

### 12.3 OS-level sandboxing (the backstop)

Permission dialogs can be auto-approved; the sandbox cannot be clicked
through. Sandbox design (following Claude Code's seatbelt-based native
sandbox and Codex's container model):

- **macOS**: Seatbelt (TrustedBSD MAC) profile — kernel blocks writes outside
  the working directory; network traffic routes through a local proxy with an
  allowlist; kernel blocks non-loopback traffic that bypasses the proxy.
- **Linux**: same profile via landlock/Seccomp or Docker container with
  iptables firewall rules (Codex CLI's approach).
- **Hardcoded denies** (learned from Claude Code's sandbox experiments):
  `**/.git/config` and `**/.git/hooks/**` are denied even inside `allowWrite`
  paths (git hooks are code execution). Secret-path read-denies by default.
- **Escape hatch discipline**: a `dangerouslyDisableSandbox` flag exists but
  is overridable by policy (`allowUnsandboxedCommands: false` + system-prompt
  change: "All commands MUST run in sandbox mode"), because the field
  observed agents talking themselves *out* of the sandbox after a single
  failed command.
- **Known sandbox friction** (documented, tested): `git init`/`cargo init`/
  `uv init` (hardcoded `.git/hooks` deny), `gh api` (TLS via proxy), `docker
  build` (daemon outside sandbox), native `fetch` (ignores HTTPS_PROXY).
  Compatibility tests are part of the test suite (§19).
- **Cloud/CI mode**: Docker container, no network, repo + deps staged
  (Codex model) — used for unattended verification runs.

### 12.4 Application-level safety (independent of sandbox)

- **Secrets**: API keys from environment/`.env` only; never logged; scrub
  sensitive args from logs and analytics; `.env` and `.aider*` added to
  `.gitignore` automatically.
- **Remote content**: URLs offered to users are scraped with a headless
  browser; prompts injected by web pages are an accepted, user-approved risk
  (the user explicitly approves each URL).
- **Shell commands**: model-suggested commands are executed only after
  explicit user confirmation (or sandbox-contained auto-run), and output is
  fed back.
- **File access**: only in-chat files are editable; read-only files are
  enforced; no writes outside the repo root (subtree mode).
- **Supply chain**: tree-sitter grammars, linters, and the LLM provider
  client are pinned versions.

---

## 13. State Management and Concurrency

| Concern | Mechanism |
|---|---|
| Session state | In-memory `Coder` object; messages + files + costs are the state |
| History persistence | `--restore-chat-history` / `--chat-history-file` markdown transcripts |
| Background threads | Summarizer thread; cache-warming thread; both daemonized, both must be joinable before the next turn reads their data |
| Race safety | Summaries are snapshotted and compared before commit; warming pings never affect the real request; file watcher events are debounced |
| Interrupts | Ctrl-C is graceful (first press stops generation, second exits); no partial writes left behind |
| Parallel agents | Git worktrees for isolation; sandbox per agent; parent merges results (§11.2) |

---

## 14. Data Flows (sequence)

```
User: "Refactor validate_email into a helper module"

1. preproc: no command; extract identifiers {validate_email}; mention match → none
2. assemble: system + examples + summarized history + repo map
   (hinted by "validate_email") + chat files + user message
3. send: stream reply; accumulate partial_response_content
4. reply_completed: no-op (default coder)
5. get_edits: parse 2 SEARCH/REPLACE blocks
6. dry-run: block 1 matches; block 2 fails (whitespace mismatch)
7. error path: build structured failure message
8. reflect: send failure message; model replies with corrected block
9. apply edits → auto-commit "Add validation helper module, refactor validate_email"
10. auto-lint: 0 errors → turn ends
```

Tool-loop variant:

```
User: "Why does the build fail on CI but not locally?"

1. assemble prompt with workspace summary + tool descriptions
2. model: read_file(pipeline.yml) → observe
3. model: grep("ci") → observe
4. model: bash("docker compose build 2>&1") → user approves → observe
5. model: answer with root cause + suggested fix; stop_reason=end_turn
```

---

## 15. Failure Modes and Error Handling

| Failure | Detection | Response |
|---|---|---|
| Malformed edit blocks | Parser `ValueError` | Structured error → reflection (max 3) |
| SEARCH block doesn't match | Matcher returns None | `find_similar_lines` suggestion → reflection |
| Context window exceeded | Provider error class | Explain, offer `/drop` `/clear`, never blind-retry |
| Output length exceeded | `finish_reason=length` | Assistant-prefill continuation when supported; else guidance |
| Provider rate limit | Provider error class | Exponential backoff with visible retry messages |
| Empty reply | No content chunks | Warn; suspect provider/account issue |
| Corrupt/unsupported git repo | Repo sanity check | Fail early with actionable message; `--no-git` escape hatch |
| Model in degenerate loop | Reflection counter, malformed-response counter, no-progress detector | Stop after max reflections, show counters and what was tried |
| Recovery loop making things worse | Direction-of-failure (lint errors increasing) | Stop; hand control to user |
| Keyboard interrupt | `KeyboardInterrupt` | Graceful stop; second Ctrl-C exits |
| Unsupported model settings | Capability registry check | Warn and ignore unsupported params, with override flag |
| Sandbox-blocked command | OS-level denial | Surface actionable error; never auto-escalate out of sandbox |
| MCP server failure/crash | Transport error / timeout | Detach server; retry once; surface to user; continue without it |
| Truncated tool-call JSON | Parser | Lenient repair (json repair); reflect if unrepairable |

---

## 16. Observability, Analytics, and Testing

### 16.1 Observability

- `--verbose` dumps the exact messages sent to the model (scrubbed).
- `--llm-history-file` logs every request/response pair for post-mortems.
- Cost and token counters per message and per session (`/tokens`, `/cost`).
- Tool-call traces (stream-json output mode) for debugging agent runs.

### 16.2 Analytics (privacy-first)

- Opt-in anonymous events: launch, message send, exit reason, model
  warnings, edit failures — **never code, chat content, or keys**.
- Permanently disable-able; stored locally when refused.

### 16.3 Evaluation strategy (evals as a product discipline)

**Public benchmarks are a filter; internal evals are the verdict.** The
benchmark landscape:

| Benchmark | What it measures | Notes |
|---|---|---|
| SWE-bench Verified | Real GitHub issue resolution in 12 large Python repos (500 tasks, human-validated) | Gold standard for repo navigation + patching; Python-only; contamination risk high; scores are harness scores (frontier: 74–78%, May 2026) |
| Aider Polyglot | 225 hard Exercism exercises, 6 languages (C++, Go, Java, JS, Python, Rust); required edit format; 2 attempts (2nd with failing-test output) | Measures edit-application + self-correction — the two failure modes that break production agents; self-calibrating (only problems ≤3 models solve) |
| Terminal-Bench | Long-horizon shell-work tasks | Agent-command loop quality |
| SWE-PolyBench / RExBench | Multi-language repo-level tasks | Counter Python-only bias |
| ContextBench | Finding vs. using relevant code as separate skills | Debug retrieval failures specifically |

Principles:

1. **Harness-controlled runs**: evaluate *our scaffold* with fixed prompts and
   tools; treat model-vs-model comparisons as secondary.
2. **Internal eval suite**: tasks drawn from our own repos (the unit of
   measurement is a task with starting state, goal, and verifiable outcome).
   Include a "golden transcript" suite: recorded real model transcripts
   replayed against parsers/matchers in CI — **regression tests must lock the
   behavior of the prompt/parser pair**, since real models are too slow and
   non-deterministic for CI.
3. **Signal beyond pass/fail**: first-attempt vs. second-attempt deltas,
   edit-application success rate, reflection counts, tokens spent — these are
   where reliability lives.
4. **Quality guardrails**: agent commits change tests more often than human
   commits (23% vs 13%) and add mocks more often (36% vs 26%) — our eval
   must flag test-weakening edits, not just "did the tests pass".
5. **Cost discipline**: report cost per resolved task ("AI Agents That
   Matter"); an agent that wins on score but costs 10× is not a win.

### 16.4 Testing strategy

| Level | What | How |
|---|---|---|
| Unit | Parsers, matchers, token budgeting | Golden transcripts captured from real models (fixtures) |
| Unit | Repo map ranking | Synthetic repos with known symbol graphs |
| Integration | Full turn: prompt → edit → commit | Fake LLM that replays recorded responses (deterministic) |
| Integration | Git operations | Temporary repos; assert commit graphs, undo, dirty commits |
| Integration | Sandbox compatibility | Tested tool matrix (git init, package managers, gh, node fetch…) per platform |
| E2E | CLI against real APIs | Smoke suite with cheap models, nightly |
| E2E | Benchmark harness | SWE-bench Verified subset + Polyglot harness wired to CI |
| Property | Matcher robustness | Fuzz: random whitespace perturbations must not change semantics |

---

## 17. Performance and Scaling

- **Startup**: lazy imports + background pre-import of heavy deps
  (httpx, litellm, networkx, numpy); first-run synchronous load only.
- **Token efficiency**: prompt caching headers with cache-shaped prompt
  ordering; map-token budget (1/8); weak-model summarization; cache warming
  for long sessions; deferred MCP tool schemas.
- **Throughput**: streaming rendering; parallelizable pieces (file content
  assembly, repo map) run before the LLM call, not inside it.
- **Model choice**: weak model for summaries and commit messages; strong
  model for edits; editor model for architect mode. Cost/latency is a
  negotiable per-model property.
- **Latency budget**: interactive coding agents are judged on wall-clock;
  Cursor's Composer targets <30s task completion with RL-tuned tool-use
  efficiency. Our budget: first token <2s (with cache), typical turn
  (1–2 reflections) <30s, large-file full rewrites streamed.

---

## 18. Product Surfaces and UX

### 18.1 Terminal-first

The terminal is the primary surface (aider, Claude Code, opencode, Codex CLI
all chose it): zero setup, works in any editor, scriptable. Requirements:

- prompt-toolkit-based input with vi/emacs modes, syntax-highlighted output
  (Rich), context-aware autocompletion (files, commands, model names).
- Streaming markdown output with incremental rendering.
- **Watch mode**: monitor files for AI comment markers (`// ai!` for edits,
  `# ai?` for questions) — the bridge to any IDE without a plugin.
- Voice input (Whisper) and web scraping (`/web`) as input methods.

### 18.2 Rules and memory

The AGENTS.md / CLAUDE.md pattern is now the industry convention (Cursor,
Copilot, Codex all read it; Codex's adoption made it an open format):

- **Hierarchy**: user rules → project rules (AGENTS.md at repo root) →
  nested AGENTS.md (directory-scoped, most specific wins) → per-session
  instructions. Earlier sources take precedence on conflict (Cursor's
  documented order).
- AGENTS.md content: stack, conventions, test/build commands, prohibitions.
- No implicit cross-session memory; memory is *explicit* (rules files,
  `--chat-history-file`). Windsurf's automatic memory is a deliberate
  non-goal: predictable > magical.

### 18.3 Human-in-the-loop

- Approval for edits and commands (with "don't ask again" persistence).
- Undo Last Edit (turn-level) + `/undo` (commit-level) + git itself.
- Transparent tool invocation display — every action the agent takes is
  visible in the transcript.
- The agent must *ask* when the task is ambiguous or the failure is
  architectural (not mechanical) — knowing when to stop is a feature.

---

## 19. Adversarial and Edge-Case Thinking

What the field's incident reports teach us:

1. **Prompt injection via repo content**: a repository can contain
   instructions aimed at the agent (READMEs, tests, `.env.example`,
   MCP-served data). Mitigation: never treat file content as instructions;
   sandbox contains the blast radius; remote content is user-approved.
2. **The agent can be socially engineered by its own environment**: Claude
   Code's sandbox experiments showed agents disabling their own sandbox after
   repeated failures. Policy: sandbox-disable is not model-decisionable.
3. **Tools inherit the dangers of the commands they wrap**: mcp-server-git's
   CVEs (flag injection, path traversal) — validate arguments like a security
   boundary, and prefer removing capabilities over patching them.
4. **The verification loop can be gamed**: an agent that weakens tests to
   make them pass is worse than one that fails. Track test modifications;
   require human review for test edits.
5. **Contamination**: any agent trained on public benchmarks can memorize
   answers; evaluate on held-out private tasks for model selection, and
   prefer *fresh* internal tasks for our own scaffold evals.
6. **Deterministic core**: every LLM-dependent step must be isolatable and
   mockable — the harness runs in CI with recorded transcripts; a broken
   parser must be catchable without a model in the loop.

---

## 20. Performance Targets

| Metric | Target |
|---|---|
| Startup to prompt | < 800ms (lazy imports; warm-up thread) |
| First token (cached) | < 2s |
| Typical edit turn (incl. 1 reflection) | < 30s |
| Repo map build (100K-file repo, warm cache) | < 2s |
| Reflection budget | max 3/turn; no-progress stop after 2 identical failures |
| Polyglot-style edit-application success (v1) | ≥ 90% on golden transcripts |
| Sandbox overhead per command | < 100ms (paid once per process) |

---

## 21. Key Design Decisions (Summary)

| # | Decision | Rationale |
|---|---|---|
| D1 | Edit formats are versioned closed protocols (prompt ↔ parser ↔ tests) | LLM output is non-deterministic; only lockstep testing makes parsing reliable |
| D2 | Repo map is hinted by the current message's identifiers | Highest relevance-per-token for zero embedding cost |
| D3 | Map refresh on file change, lazily at turn time | Budget CPU; freshness matters mostly at edit time |
| D4 | Model capabilities are data files, not code | Add models without releases |
| D5 | Git commit chain is the undo log | Battle-tested, crash-safe, integrates with user's workflow |
| D6 | Commit messages generated from the diff, not the chat | Honesty and correctness of history |
| D7 | Text-edit protocol + function calling are dual paths behind one interface | Works with any model; fast path when capable |
| D8 | Reflection (feed failures back) over re-prompting, bounded and no-progress-detected | Empirically the highest-leverage reliability mechanism |
| D9 | Search, don't index (agentic grep/glob over embeddings) | Simpler, more private, empirically equal-or-better (Claude Code) |
| D10 | Layered permissions + OS sandbox; sandbox is non-negotiable by the model | Permission prompts are UX, not security |
| D11 | Loop = ReAct-shaped generate-test-repair; plan-execute layered on top | Composable primitives (taxonomy of 13 scaffolds) |
| D12 | MCP tools deferred (names in context, schemas on demand); scoped trust | Context cost proportional to usage; minimize attack surface |
| D13 | Evals: public benchmarks as filter, internal task evals as verdict; harness-controlled | Benchmarks are protocol results, not production truth |

---

## 22. Open Questions and Future Work

1. **Embedding-based retrieval**: replace/augment identifier-hinting with
   embeddings when repositories are huge (>100K files) or the user's language
   is vague ("the bug in the payment flow"). Gate on ContextBench-style
   retrieval evals.
2. **Long-horizon autonomy**: multi-step plans with checkpoints, resume on
   failure, persistent task state across sessions; Codex-cloud-style
   containerized background agents with PR-based handoff.
3. **LSP integration**: live diagnostics instead of run-`ruff`-after-edit;
   symbol definitions for better mention resolution.
4. **Parallel sub-agents**: fan out independent subtasks to multiple model
   calls with a coordinator merging results (merge-conflict resolution is the
   hard part); worktree isolation is the enabling primitive.
5. **Edit format evolution**: move from text protocols to structured outputs
   (JSON schema / tool calls) as model reliability improves, while keeping the
   text fallback for weak models.
6. **Privacy modes**: fully local model paths (Ollama) with the same
   abstraction layer.
7. **Memory systems**: explicit memory files (AGENTS.md) today; evaluate
   learned memory only if SWE-ContextBench-style evidence shows retrieval
   discipline makes it net-positive.
8. **Multi-repo and monorepo scoping**: subtree-limited edits today; explicit
   cross-repo operations are a separate product decision.
9. **Model-trained agents**: RL-tuned coding policies (codex-1, Composer,
   mini-SWE-agent) are changing the model side; our scaffold must stay
   interface-compatible as model behavior shifts (interface stability is the
   product).

---

## 23. Implementation Roadmap (sketch)

| Phase | Scope | Exit criteria |
|---|---|---|
| 0 (weeks 1–2) | Skeleton: CLI, IO layer, Coder factory, litellm plumbing, one edit format (`whole`), git auto-commit | End-to-end edit round-trip with a real model |
| 1 (weeks 3–6) | RepoMap (tree-sitter + PageRank), `diff` format + matcher ladder, reflection loop, lint feedback | Polyglot harness ≥ 60% on one strong model |
| 2 (weeks 7–10) | ChatSummary compression, token budgeting, `architect` mode, `/undo`, dirty commits, watch mode | Internal eval suite green; SWE-bench Verified subset wired |
| 3 (weeks 11–14) | Tool calling + MCP client, permission tiers, sandbox (macOS first), command system complete | Sandbox compatibility matrix passing; security review |
| 4 (weeks 15–18) | Subagents, worktree parallelism, evals harness in CI, observability dashboards | Benchmark results + cost-per-task published internally |

---

## 24. Appendix A: Glossary

| Term | Meaning |
|---|---|
| Edit format | The machine-parseable protocol the model uses to request file changes |
| Reflection | Re-sending the model a follow-up message generated by the agent itself (failures, lint output) |
| Repo map | Token-budgeted, ranked summary of repository symbols sent with each prompt |
| Files in chat | The editable file set; the only files the model may change |
| Dirty commit | A safety commit of uncommitted user work made before the agent edits |
| SwitchCoder | Exception that triggers runtime model/format switch while preserving session state |
| Weak model | Cheap model used for compression/summarization/commit messages |
| Editor model | Second model that applies edits in architect mode |
| Assistant prefill | Continuing a truncated reply by prepending the partial output as an assistant message |
| ReAct | Interleaved reasoning + acting loop (Yao et al., 2023) |
| Reflexion | Verbal self-critique from environment feedback (Shinn et al., 2023) |
| Generate-test-repair | Loop primitive: write code, run tests, fix failures |
| Sandbox | OS-level enforcement boundary (Seatbelt/landlock/Docker) |
| MCP | Model Context Protocol: client-host-server tool/resource standard |
| Compaction | Summarizing conversation history to fit the context window |
| No-progress detector | Mechanism that stops the loop when the model repeats failed actions |

## 25. Appendix B: References

**Our reference implementations (source-level):**
- aider: `aider/coders/base_coder.py`, `editblock_coder.py`, `architect_coder.py`,
  `aider/repomap.py`, `history.py`, `models.py`, `repo.py`, `linter.py`,
  `commands.py` — this repository.
- Aider architecture map (ggprompts.com):
  https://ggprompts.com/architecture/aider/index.html

**Production systems:**
- Claude Code docs — overview, agent loop, subagents, sandboxing, memory,
  hooks, MCP: https://code.claude.com/docs/en/overview
- Anthropic engineering — "Making Claude Code more secure and autonomous with
  sandboxing" (Oct 2025): https://www.anthropic.com/engineering/claude-code-sandboxing
- Claude Code internals analyses: https://cc.bruniaux.com/guide/architecture/
- VS Code blog — "Introducing GitHub Copilot agent mode" (Feb 2025):
  https://code.visualstudio.com/blogs/2025/02/24/introducing-copilot-agent-mode
- OpenAI — "Introducing Codex" (May 2025): https://openai.com/index/introducing-codex
- OpenAI — Codex architecture deep-dive (Jan 2026):
  https://www.adwaitx.com/openai-codex-agent-loop-architecture/
- Codex cloud agent internals: https://theaiengineer.substack.com/p/how-openai-codex-works
- Cursor docs — agent overview: https://cursor.com/docs/agent/overview
- Cursor engineering — dynamic context discovery: https://cursor.com/blog/dynamic-context-discovery
- Cursor explained for engineers: https://www.michaeljamieson.dev/blog/cursor-explained-for-engineers-how-the-ai-ide-actually-works
- Agent system architectures of Copilot, Cursor, Windsurf (cuckoo.network):
  https://cuckoo.network/blog/2025/06/03/coding-agent
- opencode deep-dive: https://cefboud.com/posts/coding-agents-internals-opencode-deepdive/

**Academic foundations:**
- ReAct (Yao et al., ICLR 2023): https://arxiv.org/abs/2210.03629
- Reflexion (Shinn et al., NeurIPS 2023): https://arxiv.org/abs/2303.11366
- CodeAct: https://arxiv.org/abs/2402.01030
- SWE-agent (agent-computer interface): https://arxiv.org/abs/2405.15793
- SWE-bench: https://arxiv.org/abs/2310.06770
- AI Agents That Matter (cost, holdout, benchmark shortcuts):
  https://arxiv.org/abs/2407.01502
- Inside the Scaffold — source-code taxonomy of 13 coding agents:
  https://arxiv.org/abs/2604.03515
- SWE-Edit (interface-level editor optimization): https://arxiv.org/abs/2604.26102
- ContextBench (finding vs. using code): https://arxiv.org/abs/2602.05892
- SWE-ContextBench (memory/retrieval discipline): https://arxiv.org/abs/2602.08316

**Industry guides:**
- Anthropic — Building Effective Agents:
  https://www.anthropic.com/engineering/building-effective-agents
- OpenAI — A Practical Guide to Building Agents:
  https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents
- Anthropic — How we built our multi-agent research system:
  https://www.anthropic.com/engineering/multi-agent-research-system

**Standards:**
- Model Context Protocol — spec & docs: https://modelcontextprotocol.io
- MCP GitHub (spec repo): https://github.com/modelcontextprotocol/modelcontextprotocol
- mcp-server-git CVEs (GHSA-5cgr-j3jf-jw3v, CVE-2025-68144/145):
  https://github.com/advisories/GHSA-5cgr-j3jf-jw3v

**Benchmarks & evals:**
- SWE-bench Verified: https://www.swebench.com/verified.html
- Epoch AI — "What skills does SWE-bench Verified evaluate?" (Jun 2025):
  https://epoch.ai/publications/what-skills-does-swe-bench-verified-evaluate
- Aider Polyglot benchmark: https://aider.chat/docs/leaderboards/
- Polyglot methodology write-up: https://aider.chat/2024/12/21/polyglot.html
- AI coding agent evals overview (SWE-bench / Polyglot / Terminal-Bench):
  https://sureprompts.com/blog/ai-coding-agent-evals-swe-bench-aider-polyglot-terminal-bench