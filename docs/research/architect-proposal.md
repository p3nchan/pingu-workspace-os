# Workspace Architecture Proposal: North Star Design

> **Role**: System Architect
> **Date**: 2026-03-07
> **Status**: Draft for multi-agent debate
> **Scope**: File management, memory systems, project tracking for an AI agent (Claude) as primary reader/writer, with a human co-reader (Penchan)

---

## 0. Design Philosophy

Three axioms drive every decision in this proposal:

1. **The agent's context window is RAM; the filesystem is disk.** Every architectural choice must minimize what loads into RAM while maximizing what can be paged in on demand. Wasted tokens are wasted money and wasted reasoning capacity.

2. **Markdown is the source of truth; everything else is a derived view.** Markdown files are human-readable, version-controllable, auditable, and natively understood by LLMs due to training data distributions. Any index, catalog, or database must be rebuildable from the markdown layer.

3. **Predictability over expressiveness.** An agent that can deterministically find a file in O(1) beats an agent that must explore. Rigid conventions beat flexible ones. Boring, stable paths beat clever, dynamic ones.

---

## 1. Target Directory Structure

### Design Rationale

The structure uses a **Johnny Decimal-inspired numeric prefix system** at the top level to provide deterministic routing, while keeping the tree **flat-and-wide** (max 3 levels deep for any content file). Numeric prefixes create a stable namespace: `00-09` is always system, `10-19` is always projects, etc. This means both the agent and Penchan can predict where things live without consulting an index.

Key departures from current system:
- Root-level clutter (70+ items) collapses into ~10 top-level directories
- `reference/`, `references/`, `workspace/`, `workspace-codex/`, `tmp/` merge or archive
- `browser/`, `canvas/`, `completions/`, `identity/` move into appropriate buckets
- System config files (SOUL.md, AGENTS.md, etc.) move into `00-system/`

### Full Tree

```
.openclaw/
|
|-- 00-system/                          # System identity, rules, boot files
|   |-- BOOT.md                         # Boot manifest (what to load, in what order)
|   |-- SOUL.md                         # Agent identity (immutable-ish)
|   |-- AGENTS.md                       # Operational rules
|   |-- USER.md                         # Human profile
|   |-- ORCHESTRATION.md                # Multi-agent framework
|   |-- SKILL-POLICY.md                 # Skill governance
|   |-- TOOLS.md                        # Tool configuration
|   |-- catalog.json                    # Auto-generated file index (rebuildable)
|   `-- templates/                      # Reusable templates
|       |-- task-envelope.md
|       |-- decision-card.md
|       |-- project-init/               # context.md + status.md starters
|       `-- adr.md
|
|-- 10-projects/                        # Active projects (the work)
|   |-- fiscus/
|   |   |-- context.md                  # Background, strategy, sub-project index
|   |   |-- status.md                   # Current state, phases, blockers
|   |   |-- decisions/                  # Append-only ADRs
|   |   |   `-- 0001-grid-strategy.md
|   |   |-- research/                   # Project-specific research artifacts
|   |   |-- drafts/                     # Work in progress outputs
|   |   `-- archive/                    # Completed phase artifacts
|   |-- penchan-co/
|   |   |-- context.md
|   |   |-- status.md
|   |   |-- decisions/
|   |   |-- codex/                      # Site codebase reference
|   |   `-- seo/
|   |-- p3nchan/
|   |   |-- context.md
|   |   |-- status.md
|   |   `-- content/
|   |-- health-analytics/
|   |-- da-capital/
|   |-- thesis/
|   |-- workspace-redesign/             # This very project
|   `-- _archive/                       # Completed/paused projects (moved here)
|       |-- chart-skill/
|       |-- pingu-avatar/
|       `-- white-cat-planet/
|
|-- 20-areas/                           # Ongoing responsibilities (no end date)
|   |-- investment/                     # Crypto/stock investment research
|   |   |-- context.md
|   |   |-- watchlist.md
|   |   `-- protocols/                  # Per-protocol research files
|   |       |-- sui.md
|   |       |-- cetus.md
|   |       `-- haedal.md
|   |-- content-creation/               # X posts, writing pipeline
|   |   |-- context.md
|   |   |-- sop.md                      # X content SOP
|   |   `-- drafts/
|   |-- community/                      # Discord, Telegram, penguin-group, nekopen
|   |   |-- context.md
|   |   |-- penguin-group.md
|   |   `-- nekopen-group.md
|   `-- openclaw-ops/                   # System maintenance, debug
|       |-- context.md
|       |-- debug-notes.md
|       `-- cron-config.md
|
|-- 30-memory/                          # All memory tiers
|   |-- MEMORY.md                       # Index + stable facts (<1KB, loaded at boot)
|   |-- semantic/                       # Permanent knowledge (rarely changes)
|   |   |-- people.md                   # Penchan profile details, contacts
|   |   |-- preferences.md              # Communication style, format rules
|   |   |-- systems.md                  # Tool configs, model aliases, infra facts
|   |   |-- astrology.md                # Ba-zi reading, reference data
|   |   `-- glossary.md                 # Domain-specific terms
|   |-- episodic/                       # Event-based memories (monthly rollup)
|   |   |-- 2026-02.md                  # Feb 2026 compressed events
|   |   `-- 2026-03.md                  # Mar 2026 compressed events
|   |-- daily/                          # Raw daily notes (ephemeral -> compress)
|   |   |-- 2026-03-07.md
|   |   |-- 2026-03-06.md
|   |   `-- ...
|   `-- experiments.md                  # Experiment log (tool/strategy trials)
|
|-- 40-skills/                          # Agent capabilities (stateless definitions)
|   |-- _index.md                       # Skill registry: name, trigger, path
|   |-- news-toolkit/
|   |   |-- SKILL.md
|   |   `-- references/
|   |-- security-audit/
|   |-- security-github/
|   |-- us-stock/
|   |-- workflow-runner/
|   |   `-- SKILL.md
|   |-- crypto-research/
|   |-- latex/
|   `-- ...                             # Other skills follow same pattern
|
|-- 50-workflows/                       # Reusable process definitions
|   |-- _index.md                       # Workflow registry
|   |-- forum-automation.md
|   |-- session-retro.md
|   |-- content-performance.md
|   |-- x-content-sop.md
|   `-- templates/                      # Instantiable templates
|       |-- ecosystem-weekly.md
|       |-- project-weekly.md
|       `-- research-report.md
|
|-- 60-resources/                       # Reference material (read-only-ish)
|   |-- anthropic-best-practices.md
|   |-- research-tools.md               # Tool inventory + usage notes
|   `-- discord-server-map.md           # Channel IDs, forum structure
|
|-- 70-media/                           # Binary and inbound files
|   |-- inbound/                        # Files received from external sources
|   `-- generated/                      # Charts, images, exports
|
|-- 80-config/                          # Gateway, cron, credentials (machine-read)
|   |-- openclaw.json
|   |-- cron/
|   |-- credentials/                    # Encrypted/protected
|   `-- devices/
|
|-- 90-archive/                         # Cold storage (out of active retrieval)
|   |-- memory/                         # Old daily notes (>90 days)
|   |-- projects/                       # Completed projects
|   `-- logs/                           # Old session logs
|
|-- brain.md                            # Global task board (stays at root for fast access)
`-- .secrets/                           # Sensitive data (gitignored, never indexed)
```

### Why This Tree

| Principle | Implementation |
|-----------|---------------|
| Flat-and-wide | Max 3 levels to any content file. Agent never does >2 directory traversals. |
| Numeric prefix routing | Agent sees `10-projects` and knows it is the project bucket without semantic parsing. |
| Predictable locations | Every file type has exactly one canonical home. No `reference/` vs `references/` ambiguity. |
| Separation of concerns | System rules (`00`), active work (`10-20`), memory (`30`), capabilities (`40-50`), static reference (`60`), media (`70`), config (`80`), cold storage (`90`). |
| brain.md at root | Sole exception to "everything in a numbered folder" -- it is the global entry point read every session, so minimizing path depth matters. |

---

## 2. Memory Architecture

### The Four-Tier Model

Memory flows through four tiers with explicit promotion and demotion rules. This replaces the current two-tier model (daily notes + MEMORY.md) with a system that prevents both knowledge rot (forgetting important things) and context bloat (remembering too much).

```
Tier 0: EPHEMERAL (Context Window)
  |  Lives in: conversation context only
  |  Lifetime: single session
  |  Size: whatever fits in context
  |  Contains: current task state, intermediate reasoning, tool outputs
  |
  | -- [session end: promote or discard] -->
  |
Tier 1: WORKING (Daily Notes)
  |  Lives in: 30-memory/daily/YYYY-MM-DD.md
  |  Lifetime: 7-14 days active, then compress
  |  Size: unconstrained (raw capture)
  |  Contains: session summaries, decisions made, tasks completed,
  |            observations, conversation highlights
  |
  | -- [weekly promotion job] -->
  |
Tier 2: EPISODIC (Monthly Rollups)
  |  Lives in: 30-memory/episodic/YYYY-MM.md
  |  Lifetime: 6-12 months, then archive or merge into semantic
  |  Size: ~2KB per month target
  |  Contains: compressed event timeline, milestone dates,
  |            "what happened and why" narratives
  |
  | -- [quarterly review: extract stable facts] -->
  |
Tier 3: SEMANTIC (Permanent Knowledge)
     Lives in: 30-memory/semantic/*.md + 30-memory/MEMORY.md (index)
     Lifetime: indefinite (until explicitly invalidated)
     Size: each file <1KB, MEMORY.md <1KB
     Contains: stable facts about people, preferences, systems,
               verified domain knowledge, proven patterns
```

### What Goes Where (Decision Matrix)

| Information Type | Tier | Example |
|-----------------|------|---------|
| "Penchan said to try approach X" | T1 Daily | Capture verbatim, may or may not persist |
| "We tried approach X, it failed because Y" | T1 -> T2 | Promote to episodic if pattern-worthy |
| "Approach X never works for us because Y" | T2 -> T3 | Promote to semantic once confirmed 2+ times |
| "Penchan prefers bullet points over prose" | T3 Semantic | Goes directly to `preferences.md` |
| "Model alias: opus = claude-max/claude-opus-4" | T3 Semantic | Goes to `systems.md` |
| "2026-03-05: Launched penchan.co Phase 0" | T2 Episodic | Milestone event |
| "Sub-agent output: 3000 words of research" | T0 Ephemeral | Consumed and discarded, conclusions only persist |
| "ADR: Chose Next.js over Astro for penchan.co" | Project decisions/ | Not memory tier -- lives in project |

### Promotion and Demotion Rules

**Promotion (upward through tiers)**:
- T0 -> T1: At session end, agent writes a brief summary to today's daily note. Only conclusions, decisions, and new facts. No tool outputs, no intermediate reasoning.
- T1 -> T2: Weekly cron job reads past 7 daily notes + existing episodic file, performs **joint rewrite** (not append). Agent compresses into timeline format. Daily notes >14 days old move to `90-archive/memory/`.
- T2 -> T3: Quarterly review. Agent reads all episodic files, extracts facts that have remained stable across 3+ months. These become entries in `semantic/*.md`. Episodic entries that sourced them get a `[promoted]` tag.

**Demotion / Archival**:
- T3 facts that become false: Mark with `[superseded: YYYY-MM-DD]` and move to bottom of file. Do not delete -- audit trail matters.
- T2 episodic entries >12 months: Archive to `90-archive/memory/`.
- T1 daily notes >14 days: Archive to `90-archive/memory/`.

### State Freezing Protocol

When a session runs long (>15 turns or coherence degrades):
1. Agent compiles current state: task progress, open decisions, next steps
2. Writes this to `status.md` of the relevant project
3. Session terminates
4. New session boots fresh, reads `status.md`, continues with clean context

This prevents "context rot" -- the gradual drowning of signal in derivation noise (failed attempts, reversed decisions, stack traces). The compiled state is the truth; the conversation history is disposable.

---

## 3. Indexing / Catalog Strategy

### catalog.json — The Rebuildable Index

A single JSON file at `00-system/catalog.json` that maps every significant markdown file in the workspace. This is the agent's "table of contents" -- it can answer "where is X?" without recursive directory traversal.

```json
{
  "version": "1.0",
  "generated_at": "2026-03-07T00:00:00+08:00",
  "entries": [
    {
      "path": "10-projects/fiscus/context.md",
      "type": "project-context",
      "project": "fiscus",
      "title": "Fiscus - DeFi Yield Optimization",
      "status": "active",
      "updated": "2026-03-06",
      "size_bytes": 2048,
      "tags": ["defi", "yield", "sui"]
    },
    {
      "path": "30-memory/semantic/systems.md",
      "type": "semantic-memory",
      "title": "System & Tool Configuration",
      "updated": "2026-03-06",
      "size_bytes": 1024,
      "tags": ["tools", "models", "config"]
    }
  ]
}
```

### What Gets Indexed

| Indexed | Not Indexed |
|---------|-------------|
| All `context.md`, `status.md` in projects | Binary files in `70-media/` |
| All files in `30-memory/` | Files in `90-archive/` (cold storage) |
| All `SKILL.md` files | Files in `.secrets/` |
| All workflow definitions | Config files in `80-config/` |
| All files in `60-resources/` | `catalog.json` itself |
| ADR files in `decisions/` | Temporary/scratch files |

### When It Rebuilds

- **Daily**: Cron job at 04:00 scans all indexed directories, regenerates `catalog.json`
- **On demand**: Agent can trigger rebuild after significant file operations
- **Cost**: ~0 tokens (filesystem scan, no LLM involved)
- **Integrity**: If `catalog.json` is missing or corrupt, the system still works -- agent falls back to directory listing + glob patterns. The catalog is a performance optimization, not a dependency.

### Discovery Flow

```
Agent needs to find information about topic X:
  1. Check brain.md (is it a current task?)
  2. Check MEMORY.md index (is it a known stable fact?)
  3. Scan catalog.json for matching tags/titles
  4. Read the matching file(s) on demand
  5. If catalog fails: glob for likely filenames, then grep content
  6. Last resort: memory_search semantic tool
```

The agent should almost never reach step 5 or 6. The catalog + predictable naming conventions should resolve 90%+ of lookups in steps 1-4.

### Skill and Workflow Registries

In addition to the global catalog, two domain-specific indexes:

- `40-skills/_index.md`: Human-readable list of all skills with trigger conditions and paths
- `50-workflows/_index.md`: Human-readable list of all workflows with trigger conditions and paths

These are maintained manually (or by cron) as lightweight markdown tables. The agent reads them when it needs to find a capability.

---

## 4. Boot Sequence

### Design Goals
- Load the minimum needed to be operational
- Total boot context < 3,000 tokens
- Additional context loaded lazily based on task

### The Sequence

```
PHASE 1: Identity (always, every session)
  Read: 00-system/BOOT.md .............. ~200 tokens
  Read: 00-system/SOUL.md .............. ~400 tokens
  Read: 00-system/USER.md .............. ~300 tokens
  Subtotal: ~900 tokens

PHASE 2: Operational State (always, every session)
  Read: brain.md ....................... ~300 tokens
  Read: 30-memory/MEMORY.md ........... ~500 tokens
  Read: 30-memory/daily/today.md ...... ~400 tokens
  Read: 30-memory/daily/yesterday.md .. ~400 tokens
  Subtotal: ~1,600 tokens

PHASE 3: Conditional (only if active tasks exist)
  IF brain.md has orchestration tasks:
    Read: 00-system/ORCHESTRATION.md ... ~2,000 tokens (heavy, but conditional)
  IF brain.md references specific projects:
    Read: 10-projects/<name>/status.md . ~300 tokens each
  Subtotal: 0-3,000 tokens

PHASE 4: On-Demand (triggered by conversation)
  Penchan mentions project X:
    Read: 10-projects/X/context.md
  Agent needs a skill:
    Read: 40-skills/_index.md -> specific SKILL.md
  Agent needs a workflow:
    Read: 50-workflows/_index.md -> specific workflow
  Agent needs deep memory:
    Read: 30-memory/semantic/<topic>.md
  Agent needs project history:
    Read: 10-projects/X/decisions/*.md

  Cost: variable, pay-per-use
```

### Estimated Boot Cost

| Scenario | Tokens | Cost (Opus input) |
|----------|--------|--------------------|
| Minimal (chat, no tasks) | ~2,500 | ~$0.04 |
| Normal (1-2 active projects) | ~3,500 | ~$0.05 |
| Heavy (orchestration + 3 projects) | ~6,000 | ~$0.09 |

### BOOT.md Contents

The `BOOT.md` file is the master manifest. It tells the agent exactly what to load and in what order. It replaces the implicit boot logic currently scattered across AGENTS.md.

```markdown
# BOOT.md — Session Initialization Manifest

## Phase 1: Identity
- Read: 00-system/SOUL.md
- Read: 00-system/USER.md

## Phase 2: Operational State
- Read: brain.md
- Read: 30-memory/MEMORY.md
- Read: 30-memory/daily/{today}.md
- Read: 30-memory/daily/{yesterday}.md

## Phase 3: Conditional
- IF brain.md contains orchestration tasks OR any project status is `in_progress`:
  - Read: 00-system/ORCHESTRATION.md
- FOR EACH project referenced in brain.md "In Progress":
  - Read: 10-projects/{name}/status.md

## Phase 4: On-Demand
- Everything else: read when the conversation requires it
- Never preload: archive/, media/, config/, credentials/

## Rules
- AGENTS.md rules are internalized (baked into system prompt by gateway)
- Do NOT read AGENTS.md every session -- it is injected by the gateway
- Total Phase 1+2 target: <2,500 tokens
- Total Phase 1+2+3 target: <6,000 tokens
```

### Sub-Agent Boot (Minimal)

Sub-agents do NOT run the full boot sequence. They receive:
1. Task Envelope (goal, acceptance criteria, inputs, tools, budget)
2. Relevant file paths (not file contents -- they read on demand)
3. Write permissions scoped to specific paths

Sub-agents never read: SOUL.md, USER.md, MEMORY.md, brain.md, ORCHESTRATION.md. They are stateless pure functions.

---

## 5. Naming Conventions

### Directory Names

| Rule | Pattern | Example |
|------|---------|---------|
| Top-level buckets | `NN-lowercase-slug` | `10-projects`, `30-memory` |
| Project folders | `lowercase-slug` (no prefix) | `fiscus`, `penchan-co`, `da-capital` |
| Archived projects | Move to `10-projects/_archive/` | `10-projects/_archive/white-cat-planet/` |
| Skill folders | `lowercase-slug` | `news-toolkit`, `security-audit` |
| Date-organized dirs | `YYYY/MM/` only when >100 files | `30-memory/daily/2026/03/` (future, when needed) |

### File Names

| Rule | Pattern | Example |
|------|---------|---------|
| Daily notes | `YYYY-MM-DD.md` | `2026-03-07.md` |
| Episodic rollups | `YYYY-MM.md` | `2026-03.md` |
| ADR files | `NNNN-slug.md` (zero-padded, sequential) | `0001-grid-strategy.md` |
| Project core files | Fixed names: `context.md`, `status.md` | Always the same, always predictable |
| Skill definition | Fixed name: `SKILL.md` | Always uppercase, always at skill root |
| Semantic memory | `topic-slug.md` | `people.md`, `preferences.md`, `systems.md` |
| Index files | `_index.md` (underscore prefix = meta-file) | `40-skills/_index.md` |
| Templates | Descriptive slug | `task-envelope.md`, `research-report.md` |
| Temporary files | `_tmp-slug.md` or in dedicated `tmp/` | Cleaned up by cron |

### File Name Anti-Patterns (Banned)

- No spaces in filenames (use hyphens)
- No uppercase in directory names (except `SKILL.md` and system files)
- No version suffixes like `-v2`, `-final`, `-backup` (use git or archive/)
- No date-in-filename for non-temporal files (project slugs should not contain dates)
- No duplicate concept folders (`reference/` AND `references/` -- pick one)
- No orphan files at workspace root (everything belongs in a numbered bucket)

### Frontmatter Standard

Every significant markdown file should include YAML frontmatter:

```yaml
---
id: fiscus-context                    # Unique, stable, human-readable
type: project-context                 # Enum: project-context, project-status,
                                      #   adr, semantic-memory, episodic-memory,
                                      #   daily-note, skill-def, workflow,
                                      #   resource, research
project: fiscus                       # Which project this belongs to (if any)
status: active                        # Enum: active, paused, completed, archived
created: 2026-02-15
updated: 2026-03-06
tags: [defi, yield, sui, grid-trading]
---
```

**Minimum required fields**: `type`, `created`, `updated`
**Optional but encouraged**: `id`, `project`, `status`, `tags`
**For ADRs**: Add `decision_status: accepted | superseded | deprecated`

Frontmatter serves three purposes:
1. Catalog generation (machine-readable metadata without parsing body)
2. Agent discovery (filter by type, project, status, tags)
3. Human scanning (Penchan can quickly see what a file is about)

---

## 6. Key Architectural Decisions

### ADR-01: Johnny Decimal-Inspired Top-Level Over PARA

**Decision**: Use numeric-prefixed top-level directories (`00-system`, `10-projects`, etc.) instead of PARA's `Projects/Areas/Resources/Archives`.

**Rationale**:
- Numeric prefixes create a deterministic routing table. The agent does not need to reason about whether something is a "project" vs an "area" -- it just follows the number.
- PARA's strength (actionability-based classification) is a liability for agents. The agent would need to make subjective judgments about file placement that it is not good at.
- Johnny Decimal's strict two-level depth constraint aligns perfectly with the "flat-and-wide" principle.
- Sorting is natural: `ls` always shows directories in the correct conceptual order.

**Tradeoff**: Less intuitive for first-time human readers. Mitigated by using descriptive slugs after the number (`10-projects` not just `10`).

**What we keep from PARA**: The conceptual distinction between "projects" (have end dates), "areas" (ongoing), and "archive" (cold). We just encode it with numbers instead of words.

### ADR-02: MEMORY.md as Index, Not Content Store

**Decision**: MEMORY.md becomes a <1KB index pointing to topic files in `30-memory/semantic/`, rather than an ever-growing knowledge dump.

**Rationale**:
- Current MEMORY.md (7KB, growing) is already approaching the point where loading it costs more tokens than it provides value.
- LangGraph research explicitly warns that single-profile memory stores become error-prone as they grow.
- An index is O(1) to scan; a monolithic file is O(n) where n grows forever.
- Topic files can be loaded individually, paying only for the specific knowledge needed.

**Tradeoff**: Two reads instead of one when deep knowledge is needed (index -> topic file). But the index read is tiny, and the topic file read is targeted.

### ADR-03: Decisions in Dedicated Files, Not Daily Notes

**Decision**: All project decisions get their own file in `decisions/` using append-only ADR format. Daily notes may reference them but are never the source of truth for decisions.

**Rationale**:
- Decisions scattered across daily notes are effectively invisible after 14 days (when daily notes archive).
- ADR files are the established pattern for architectural decisions in software engineering, proven at scale.
- Append-only means decisions are never lost, even if superseded.
- The agent can answer "why did we choose X?" by reading a single file, not searching 30 daily notes.

**Tradeoff**: More files to create. Mitigated by only requiring ADRs for significant decisions (current threshold from AGENTS.md: impacts >1 sub-project, changes direction, involves external terms).

### ADR-04: Sub-Agents as Stateless Pure Functions

**Decision**: Sub-agents receive task envelopes and file paths. They never read system files, never maintain memory across invocations, never communicate with each other directly.

**Rationale**:
- Already established in ORCHESTRATION.md and working well in practice.
- Prevents state corruption from concurrent writes.
- Makes sub-agent failures isolated and retriable.
- Reduces sub-agent boot cost to near zero.

**Tradeoff**: Sub-agents cannot make system-level decisions. This is a feature, not a bug.

### ADR-05: Catalog as Optimization, Not Dependency

**Decision**: `catalog.json` is a rebuildable performance optimization. The system must work without it (falling back to glob + grep).

**Rationale**:
- Any single point of failure is unacceptable for a daily-use system.
- Catalogs can become stale. By making staleness recoverable (just rebuild), we avoid the "stale index worse than no index" problem.
- This follows the "markdown as source of truth" axiom -- the catalog is derived, never authoritative.

**Tradeoff**: Without the catalog, discovery falls back to slower filesystem operations. Acceptable because this should be rare (catalog rebuilds daily).

### ADR-06: Frontmatter as Primary Metadata Layer

**Decision**: Use YAML frontmatter in all significant markdown files for structured metadata.

**Rationale**:
- Frontmatter is parseable by both humans and machines without custom tooling.
- It enables catalog generation without reading file bodies.
- It standardizes what "metadata" means across the entire workspace.
- It is the de facto standard in static site generators, Obsidian, and many LLM-adjacent tools.

**Tradeoff**: Adds maintenance burden -- frontmatter must be kept current. Mitigated by making `updated` a field that the agent refreshes on every write, and by having the catalog builder flag files with stale `updated` dates.

### ADR-07: Chinese (Traditional) as Primary Language for Content

**Decision**: Keep Traditional Chinese as the primary language for daily notes, status files, and user-facing content. Use English for system files, naming conventions, and technical metadata.

**Rationale**:
- Penchan reads Traditional Chinese natively. Human readability is a stated constraint.
- System files (BOOT.md, catalog.json, frontmatter) use English because they are machine-primary, and English is unambiguous in technical contexts.
- This is already the established pattern and works well.

**Tradeoff**: Mixed-language workspace. Acceptable because the language boundary aligns cleanly with the human-readable vs machine-readable boundary.

### ADR-08: No Vector Database (Yet)

**Decision**: Do not introduce a vector database in this architecture iteration. Continue using `memory_search` semantic tool backed by the existing SQLite approach, enhanced by better naming and catalog.

**Rationale**:
- The current bottleneck is organizational, not retrieval. Files are hard to find because they are in wrong places with inconsistent names, not because the search algorithm is weak.
- Adding a vector DB before fixing organization would, as the Perplexity report states, "put complexity on top of chaos."
- The catalog + frontmatter + predictable naming conventions should eliminate 90%+ of discovery failures.
- Vector DB remains a valid future enhancement once the organizational foundation is solid.

**Tradeoff**: Semantic search across the full workspace remains limited to what `memory_search` currently supports. This is acceptable for the current scale (~200 significant files).

---

## 7. What We Keep From the Current System

The current system is not broken -- it is organically evolved and mostly functional. This redesign is a consolidation and formalization, not a replacement. Here is what works and stays:

### Fully Retained (Working Well)

| Current Element | Stays As-Is | Why |
|----------------|-------------|-----|
| `brain.md` as global task board | Yes (at root) | The "what am I doing right now" entry point is excellent. Every research report validated this pattern. |
| `context.md` + `status.md` per project | Yes | Two-file structure is clean, well-documented, and proven in practice. |
| Lazy loading pattern | Yes | "Read when mentioned" is correct. No report suggested eager loading. |
| Sub-agent statelessness | Yes | All research unanimously agrees this is optimal. |
| ADR system in `decisions/` | Yes | Append-only decision records are best practice across all sources. |
| Task Envelope pattern | Yes | Structured sub-agent handoff works well. |
| ORCHESTRATION.md framework | Yes | Comprehensive and recently validated. Move to `00-system/` but content stays. |
| Daily notes in `memory/YYYY-MM-DD.md` | Yes | Move to `30-memory/daily/` but same concept. |
| Write protection for system files | Yes | Sub-agents cannot touch SOUL.md, AGENTS.md, etc. Critical safety. |
| Value priority hierarchy | Yes | Safety > Honesty > Rules > Useful. Proven and correct. |
| Working signals format | Yes | The emoji + timestamp + status format is clear and functional. |

### Retained With Modifications

| Current Element | Change | Why |
|----------------|--------|-----|
| MEMORY.md | Shrink to index only; split content into `semantic/*.md` topic files | Prevents unbounded growth, enables targeted loading |
| Boot sequence (in AGENTS.md) | Extract to explicit BOOT.md manifest | Makes boot deterministic and auditable |
| Root-level system files | Move to `00-system/` | Reduces root clutter from 70+ items to ~3 |
| `projects/` directory | Rename to `10-projects/`, add `_archive/` for dead projects | Numeric prefix + archive prevents zombie projects |
| `skills/` directory | Rename to `40-skills/`, add `_index.md` registry | Enables skill discovery without scanning 60+ folders |
| `workflows/` directory | Rename to `50-workflows/`, add `_index.md` registry | Same rationale as skills |
| Session retro workflow | Add explicit T1->T2 promotion step | Currently retros happen but do not systematically promote to episodic |
| Memory compression SOP | Formalize as three-tier promotion (daily -> episodic -> semantic) | Current "joint rewrite" approach is good but only has two tiers |

### Deprecated (Remove or Merge)

| Current Element | Action | Why |
|----------------|--------|-----|
| `reference/` AND `references/` | Merge into `60-resources/` | Duplicate concept, confusing |
| `workspace/` and `workspace-codex/` | Merge relevant content into `10-projects/` or `60-resources/` | Ambiguous purpose, overlaps with projects |
| `completions/`, `browser/`, `canvas/` | Archive to `90-archive/` if unused; merge into relevant project if active | Root-level clutter with unclear ownership |
| `identity/` | Merge into `00-system/` (if system-level) or `20-areas/` | Overlaps with SOUL.md / USER.md |
| `debug_note.md` at root | Move to `20-areas/openclaw-ops/debug-notes.md` | Belongs in the openclaw operations area |
| `tmp/` at root | Replace with project-level scratch or use OS temp | 25 items accumulating without cleanup |
| Multiple `openclaw.json.bak.*` | Clean up; config backup is a cron/git concern | Root clutter |

---

## 8. Migration Considerations (For the Migration Agent)

While migration strategy is another agent's responsibility, I flag these architectural decisions that affect migration ordering:

1. **Create `00-system/` and `BOOT.md` first** -- everything depends on the boot sequence being right.
2. **Move system files before renaming directories** -- SOUL.md, AGENTS.md, USER.md must be findable at all times. Update the gateway's file references atomically.
3. **Rename `projects/` to `10-projects/` last** -- it has the most cross-references and the highest breakage risk. Consider a symlink transition period.
4. **Frontmatter can be added incrementally** -- do not block on 100% coverage. The catalog builder should handle files with and without frontmatter.
5. **Memory tier split (MEMORY.md -> semantic/) can happen in a single session** -- it is a one-time decomposition, not an ongoing migration.

---

## 9. Success Metrics

How we know this architecture is working:

| Metric | Current (Estimated) | Target |
|--------|-------------------|--------|
| Boot token cost | ~4,000-5,000 tokens | <3,000 tokens (Phase 1+2) |
| Files at workspace root | 70+ | 3 (brain.md, .secrets/, .git/) |
| Time to find a specific file | Agent grep/glob ~3-5 tool calls | Catalog lookup: 1 tool call |
| MEMORY.md size | ~7KB (and growing) | <1KB (index only) |
| Orphan files (no project/area ownership) | ~20+ | 0 |
| Duplicate concept directories | 3+ (`reference/`, `references/`, `workspace/`, `workspace-codex/`) | 0 |
| Sub-agent boot overhead | Variable (sometimes reads MEMORY.md) | Fixed: task envelope only |

---

## 10. Open Questions for Debate

These are areas where I have a position but want the other agents to challenge:

1. **Should `brain.md` stay at root or move to `00-system/`?** I argue root for minimal path depth, but it breaks the "everything in a numbered folder" rule.

2. **How aggressive should daily note archival be?** I propose 14 days. The Pragmatist might argue 7; the Preservationist might argue 30. The right answer depends on how good our T1->T2 promotion is.

3. **Should we use `.json` or `.yaml` for catalog?** JSON is more machine-friendly and the agent parses it natively. YAML is more human-readable. I chose JSON because the catalog is machine-primary.

4. **Is the `20-areas/` bucket necessary, or should areas just be projects without end dates?** The semantic distinction matters for the agent's classification logic, but it adds a routing decision. A simpler alternative: everything is a project, and `status: ongoing` flags areas.

5. **Should frontmatter be mandatory or encouraged?** Mandatory is cleaner but creates friction for quick file creation. I chose "mandatory for indexed files, optional for scratch" but this needs validation.

6. **Should episodic memory be monthly or quarterly?** Monthly gives finer granularity; quarterly produces fewer files. I chose monthly because it aligns with natural review cadence.

---

*End of System Architect proposal. Ready for critique from the Pragmatist and Devil's Advocate.*
