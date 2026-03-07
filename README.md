# pingu-workspace-os

An AI Agent Workspace OS — memory architecture, file organization, and cleanup automation for personal AI assistants.

This repository documents the full redesign of Pingu's (a personal AI assistant) workspace: how it stores memory, organizes files, discovers resources, handles session continuity, and cleans up after itself.

---

## What This Is

Modern personal AI assistants accumulate files at machine speed. Without deliberate architecture, the workspace degrades within weeks: orphaned files, fragmented context, lost decisions, and an agent that can't reliably know what it knows.

This project is a ground-up redesign of that workspace — based on external deep research, multi-agent debate, and empirical testing. It is **not** a generic framework. It is a concrete, implemented system built for a specific Anthropic Claude-based agent named Pingu, operating on macOS, with real files and real constraints.

The result is an **opinionated, minimal, file-based OS** for a personal AI agent: one that loads fast, degrades gracefully, and keeps humans in the loop without requiring them to babysit it.

---

## The 5 Core Problems Solved

| # | Problem | Symptom |
|---|---------|---------|
| 1 | **File Drift** | Files end up in wrong directories or become orphans with no clear owner |
| 2 | **Context Fragmentation** | Related information scattered across daily notes, project files, and chat history — no single source of truth |
| 3 | **Memory Decay** | Important decisions and context age out with daily notes and are silently lost |
| 4 | **Naming Inconsistency** | No enforced naming convention — same type of file named differently each time |
| 5 | **Discovery Problem** | Both agent and human sometimes forget what files and resources exist |

---

## The Debate Methodology

Rather than a single designer making all calls, this redesign used a structured multi-agent debate:

**3 Opus sub-agents, each with a fixed role:**

- **Pragmatist** — What is the minimum viable change that fixes the biggest pain? Argues against anything that requires migration or new tooling.
- **Long-term Architect** — What does the ideal system look like in 18 months? Designs for scale and robustness even if today's needs don't require it.
- **Failure Analyst** — What will go wrong? Stress-tests every proposal. Looks for hidden assumptions and brittleness.

Each agent wrote an independent analysis (see `docs/debates/`). A synthesis pass then produced the final decision matrix.

**Research base**: ChatGPT Deep Research + Perplexity (GPT 5.4) + Academic report on personal knowledge management systems for LLM agents.

---

## Key Design Decisions

### 1. MEMORY.md as Pure Index (<600 bytes) + Topic Files

MEMORY.md had grown to 7,184 bytes — a kitchen sink of facts, rules, and references. The fix: MEMORY.md becomes a pure pointer file, under 600 bytes. All actual knowledge lives in `memory/topics/*.md` files, loaded on demand.

Result: boot token cost reduced by 80%+. The agent always loads the index; it only loads topic files when relevant.

### 2. File Protection Tiers (T0–T3)

Every file in the workspace belongs to a protection tier:

| Tier | Label | Description | Cleanup Rule |
|------|-------|-------------|-------------|
| T0 | System Core | AGENTS.md, ORCHESTRATION.md, MEMORY.md | Never auto-delete |
| T1 | Project Active | Active project files | Never auto-delete |
| T2 | Episodic | Session logs, daily notes | Archive after 30 days, delete after 90 |
| T3 | Ephemeral | Tmp files, one-off outputs | Delete after 7 days |

### 3. Boot Sequence as Strict Numbered Checklist

The agent's startup routine was previously described in natural language ("read relevant files, check for pending tasks..."). This was replaced with a strict three-layer numbered checklist:

1. **Automatic** (always, no exception): load AGENTS.md, MEMORY.md, check today's status
2. **Manual** (when context requires): load project files, topic memory files
3. **On-demand** (user-triggered): deep research, catalog scan

The checklist format is intentional: it is harder to skip steps, and deviations are immediately visible.

### 4. Session-End State Freeze (Done / Incomplete / Next)

Before ending any session, the agent writes exactly 3 lines to the relevant status.md:
- What was completed this session
- What was started but not finished
- What the next concrete step is

This replaces elaborate JSON session logs. It is human-readable on iOS Discord. It survives context window resets. It takes 30 seconds.

### 5. File-Based Memory System (No Vector DB, No SQLite at This Scale)

The Architect sub-agent proposed vector embeddings and SQLite for memory retrieval. This was explicitly deferred. The reasoning:

- At <2000 files, `grep` and structured directory naming outperform embedding retrieval for this use case
- SQLite adds a dependency and an operational surface area with no current payoff
- The agent reads files, not databases — LLM cognition is text-in, text-out
- Trigger for reconsideration: >2000 files OR measurable grep latency problems

### 6. Sub-Agents as Stateless Pure Functions

Sub-agents receive a task envelope (context + instructions + output path). They produce one artifact. They do not read global state. The main agent holds all global state and is responsible for integrating sub-agent outputs.

This mirrors functional programming: sub-agents are functions, not threads.

### 7. Long-Term Thinking Embedded in Core Rules

A principle added to AGENTS.md: **"Design for 2x current scale, not 10x."** This prevents both premature optimization and the opposite failure (building for today only to re-architect in 3 months).

### 8. Lightweight catalog.md (Markdown, Weekly Cron Rebuild)

Instead of a JSON catalog with schema validation, a simple markdown file listing all active projects, their paths, and one-line descriptions. Rebuilt weekly by a cron job (gemini-flash model — fast and cheap). Human-readable. iOS-friendly.

---

## What Was Explicitly NOT Done (And Why)

| Rejected | Reason |
|----------|--------|
| **Johnny Decimal directory numbering** | Migration cost is catastrophic (23 projects + all hardcoded paths). Semantic names like `fiscus` are more meaningful to LLMs than `11.02`. |
| **Vector DB / embeddings** | No current evidence the bottleneck is retrieval speed. Adds infra complexity with no payoff at this scale. |
| **Cryptographic hash drift prevention** | Single-person system. `git diff` is sufficient. |
| **SQLite for memory** | File-based is simpler, more auditable, and adequate for <2000 files. |
| **YAML frontmatter on all files** | Adds parsing overhead. Markdown section headers accomplish 90% of the same goal for free. |
| **Zettelkasten atomic notes** | Knowledge volume is too small for the overhead. Would create more links than content. |
| **status.md → JSON** | Kills human readability. Discord iOS cannot render JSON nicely. |
| **Session init validation script** | A checklist achieves the same result with zero code to maintain. |

---

## Key Quotes from the Debate

> **Failure Analyst**: "The system already cannot follow its own simplest, most clearly stated rule (MEMORY.md <1KB, currently 7KB). Adding more rules will not fix this. It will make it worse."

> **Failure Analyst**: "The status quo is a B-. The proposals aim for an A+ but risk landing at a C."

> **Pragmatist**: "Do the minimum three changes to fix the maximum three pain points, keep the system running, and let the data tell us what to do next."

> **Architect**: "The agent's context window is RAM; the filesystem is disk. Every architectural choice must minimize what loads into RAM while maximizing what can be paged in on demand."

> **Migrator**: "Every phase has a clean rollback path. This is the most important property of the migration plan."

> **Synthesis**: "The problem is not architecture vs. discipline as a binary. It is: improve the architecture so discipline naturally follows."

---

## Directory Structure Overview

```
.openclaw/                    # Workspace root
├── AGENTS.md                 # Core rules, boot sequence, agent behavior
├── ORCHESTRATION.md          # Sub-agent dispatch and coordination rules
├── MEMORY.md                 # Pure index (<600 bytes) — pointers only
├── CONVENTIONS.md            # Naming conventions and file placement rules
├── catalog.md                # Auto-rebuilt weekly: all active projects listed
│
├── memory/
│   ├── topics/               # Named knowledge files (loaded on demand)
│   │   ├── projects.md
│   │   ├── tools.md
│   │   ├── people.md
│   │   └── system.md
│   └── (no session logs here — moved to logs/sessions/)
│
├── projects/                 # One directory per project
│   └── workspace-redesign/   # This project
│       ├── context.md
│       ├── status.md
│       ├── debates/
│       └── research/
│
├── logs/
│   ├── sessions/             # Daily session logs (T2 — archived after 30d)
│   └── cron/                 # Cleanup and cron job logs
│
└── tmp/                      # T3 ephemeral — auto-deleted after 7 days
```

---

## The Cleanup Automation System

Two cron jobs maintain the workspace:

### Daily Cleanup (Sonnet model)
- Runs each morning
- Deletes T3 files older than 7 days
- Archives T2 files older than 30 days
- Reports anomalies (unexpected large files, unknown directory structures)
- Fast, cheap, narrow scope

### Weekly Deep Cleanup (Opus model, dispatched via gemini-flash)
- Runs Sunday nights
- Rebuilds catalog.md from filesystem scan
- Reviews T2 archives for promotion to T1 or deletion
- Checks MEMORY.md size — alerts if approaching 1KB
- Produces a cleanup report for Penchan's review
- Slow, comprehensive, requires judgment

The dispatcher pattern: gemini-flash reads the weekly summary and decides whether to escalate to Opus or handle with Sonnet. This keeps costs down while ensuring high-stakes decisions get a capable model.

See `docs/drafts/daily-cleanup-v2.md` and `docs/drafts/weekly-cleanup-v2.md` for the full automation scripts.

---

## Implementation Status

| Phase | Description | Status |
|-------|-------------|--------|
| Phase 0 | External deep research (3 reports) | Done (2026-03-07) |
| Phase 1 | Cross-comparison review | Done (2026-03-07) |
| Phase 2 | Multi-agent debate + synthesis | Done (2026-03-07) |
| Phase 3 — Tier 1 | MEMORY.md split / Boot sequence / memory cleanup | Done |
| Phase 3 — Tier 2 | catalog.md / session-end freeze / CONVENTIONS.md | Pending Penchan approval |
| Phase 4 | 1-month review | Scheduled |

**Tier 1 results**:
- MEMORY.md: 7,184 bytes → 583 bytes (pure index)
- 4 topic files created (3.3KB total)
- Boot sequence: natural language → strict 3-layer numbered checklist
- memory/ directory: 25 session logs moved to `logs/sessions/` (41 → 16 files)

---

## Files in This Repository

```
README.md                           # This file
context.md                          # Project context and research foundation
status.md                           # Current phase and progress log

docs/
├── debates/
│   ├── 01-pragmatist.md            # Pragmatist sub-agent analysis
│   ├── 02-architect.md             # Long-term Architect sub-agent analysis
│   ├── 03-failure-analyst.md       # Failure Analyst sub-agent analysis
│   └── synthesis.md                # Final synthesis and decision matrix
│
├── research/
│   ├── architect-proposal.md       # System Architect full proposal
│   ├── failure-analysis.md         # Failure mode analysis
│   └── migration-plan.md           # Phased migration strategy
│
└── drafts/
    ├── AGENTS-boot-draft.md        # Revised boot sequence draft
    ├── MEMORY-index-draft.md       # MEMORY.md new format draft
    ├── cleanup-rules-v2.md         # Cleanup rule definitions v2
    ├── cleanup-review-report.md    # Review of cleanup system
    ├── cleanup-validation-report.md# Validation test results
    ├── daily-cleanup-v2.md         # Daily cleanup automation script
    └── weekly-cleanup-v2.md        # Weekly deep cleanup automation script
```

---

*Built by Pingu for Penchan — 2026-03-07*
