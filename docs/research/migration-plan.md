# Migration Plan: Workspace Redesign

> Role: Migration Strategist
> Principle: Don't break what works. This system runs daily for real work — crypto research, content creation, project management, Discord/Telegram ops. Every change must earn its place by being safer than the status quo.
> Date: 2026-03-07

---

## 1. What NOT to Change (and Why It Works)

These components are load-bearing walls. Touching them without a strong reason creates risk with no proportional gain.

### 1.1 Boot Sequence Core Flow
**Current**: SOUL.md → USER.md → memory/YYYY-MM-DD.md (today + yesterday) → MEMORY.md → brain.md → conditional ORCHESTRATION.md

**Why it works**: This sequence is a deliberate cascade from identity → user context → recent events → long-term facts → active tasks → orchestration rules. Each layer adds context that the next layer needs. The conditional ORCHESTRATION.md load (only when orchestration tasks exist) is a smart optimization — it keeps the cold start lean for simple sessions.

**Preserve as-is**: The ordering and conditional logic. Improvements should be additive (adding steps or making existing steps more deterministic), not restructuring the sequence itself.

### 1.2 Two-Tier Orchestration Model
**Current**: Pingu = sole Orchestrator with global state; sub-agents = stateless pure functions that receive context → execute → return results → terminate.

**Why it works**: This eliminates the coordination nightmare of peer-to-peer agent communication. Sub-agents cannot corrupt global state because they do not write to it. All state changes go through one bottleneck (Pingu), which means all state changes are auditable and serialized. The "no recursive spawn" rule prevents runaway agent proliferation.

**Preserve as-is**: The hierarchical model, the write-protection rules (sub-agents cannot modify AGENTS.md, SOUL.md, USER.md, MEMORY.md), and the Task Envelope format. These are battle-tested.

### 1.3 projects/<name>/context.md + status.md Structure
**Current**: Every active project has a relatively static `context.md` (background, strategy, sub-project index) and a frequently updated `status.md` (progress, phases, decisions, sub-agent records).

**Why it works**: It separates "what is this project" from "where are we now" — two fundamentally different rates of change. The lazy-loading pattern (read context.md when project is mentioned, read status.md when progress is needed) keeps token cost low.

**Preserve as-is**: The two-file split, the lazy-loading pattern, the 500-word limit on status.md, and the phase tracking format.

### 1.4 brain.md Global Task View
**Current**: A single file with sections for In Progress, Waiting for Penchan, To-Do, Scheduled Crons, and Blockers.

**Why it works**: It is the one file Pingu can read to orient itself at session start. It answers: "What is happening right now?" This is a critical affordance — without it, Pingu would need to scan every project's status.md to build a global picture, which is expensive and fragile.

**Preserve as-is**: The file, its location at workspace root, and its role as the "first thing to check after identity loading."

### 1.5 Daily Notes (memory/YYYY-MM-DD.md)
**Current**: Raw session records written throughout the day, compressed later.

**Why it works**: They are the append-only ground truth for "what happened." Even if MEMORY.md gets corrupted or pruned too aggressively, the daily notes survive as an audit trail. The date-based naming makes them trivially discoverable.

**Preserve as-is**: The pattern, the directory location, and the compression SOP (read MEMORY.md + recent logs → joint rewrite → confirm <1KB).

### 1.6 Cross-Reference Discipline / Single Source of Truth
**Current**: Each fact has one authoritative file. Other locations use relative-path links, not copies.

**Why it works**: It prevents the divergence problem where the same fact gets updated in one location but not another. Relative links also mean the workspace is self-contained and portable.

**Preserve as-is**: This is a policy, not a technical component. Keep it enforced.

### 1.7 skills/ System
**Current**: Each skill lives in `skills/<name>/` with its own SKILL.md.

**Why it works**: Skills are self-describing and self-contained. The SKILL.md convention means Pingu can discover what a skill does by reading one file.

**Preserve as-is**: The directory structure and the SKILL.md convention.

---

## 2. Phase 1 — Week 1-2: Lowest Risk, Highest Impact

These changes are purely additive. They create new files or augment existing ones. Nothing that works today is moved, renamed, or deleted.

### 2.1 Convert MEMORY.md to a Pure Index

**Problem**: MEMORY.md is currently 7KB and growing. It contains a mix of facts about Penchan, tool configurations, model aliases, workflow notes, and working principles. As it grows, boot loading becomes more expensive and updates risk accidentally dropping facts during compression.

**Change**: Split MEMORY.md into a thin index (~500 bytes) plus topic files under `memory/topics/`.

**Implementation**:
1. Create `memory/topics/` directory.
2. Extract current MEMORY.md sections into topic files:
   - `memory/topics/penchan-profile.md` — personal facts, bazi, preferences
   - `memory/topics/format-conventions.md` — token format, Telegram/Discord formatting
   - `memory/topics/model-aliases.md` — the alias table
   - `memory/topics/tools-and-skills.md` — security skills, research tools, news toolkit, macOS automation, us-stock skill
   - `memory/topics/discord-structure.md` — server layout, forum channels, iOS preview note
   - `memory/topics/working-principles.md` — sub-agent behavior, spawn guidelines, analysis approach
   - `memory/topics/workflow-runner.md` — workflow system overview
3. Rewrite MEMORY.md as a pure index:

```markdown
# MEMORY.md — Long-Term Memory Index

> Boot: always loaded. For details, read the linked topic file.

## Topic Files
- [Penchan Profile](memory/topics/penchan-profile.md) — personal, bazi, preferences
- [Format Conventions](memory/topics/format-conventions.md) — token format, messaging styles
- [Model Aliases](memory/topics/model-aliases.md) — alias → model ID mapping
- [Tools & Skills](memory/topics/tools-and-skills.md) — research tools, security skills, macOS automation
- [Discord Structure](memory/topics/discord-structure.md) — channels, forums, iOS quirks
- [Working Principles](memory/topics/working-principles.md) — sub-agent rules, spawn policy
- [Workflow Runner](memory/topics/workflow-runner.md) — workflow system overview

## Quick Facts (< 5 lines, only ultra-stable facts)
- Penchan: 宅, 奧修哲學, 追求自在, 老婆像貓咪
- Debug notes → projects/openclaw/debug-notes.md
- Anthropic best practices → memory/topics/working-principles.md

---
*Last updated: YYYY-MM-DD*
```

4. Update AGENTS.md boot sequence to say: "Read MEMORY.md (index), then read topic files on-demand when relevant to the conversation."

**Why lowest risk**: The old content is not deleted — it is relocated. If Pingu cannot find a topic file, it can fall back to reading the full archive. The index format is strictly simpler than the current monolith.

**Rollback**: If the index approach causes problems (Pingu fails to load needed context), merge the topic files back into a single MEMORY.md. This is a 5-minute operation.

### 2.2 Create a Workspace Catalog (catalog.md)

**Problem**: Neither Pingu nor Penchan can answer "what files exist in this workspace?" without doing a directory listing. This causes forgotten resources, duplicate file creation, and orphan drift.

**Change**: Create `catalog.md` at the workspace root — a human-readable, machine-parseable registry of all significant files.

**Implementation**:
1. Generate `catalog.md` by scanning the workspace once:

```markdown
# Workspace Catalog
> Auto-generated. Last rebuilt: 2026-03-07

## System Files
| File | Purpose | Updated |
|------|---------|---------|
| SOUL.md | Agent identity & values | 2026-03-04 |
| AGENTS.md | Rules, boot sequence, SOPs | 2026-03-06 |
| USER.md | Penchan profile for agent | ... |
| MEMORY.md | Long-term memory index | ... |
| ORCHESTRATION.md | Orchestrator playbook | 2026-03-06 |
| brain.md | Global task view | daily |

## Projects (23 active)
| Project | context.md | status.md | Last Activity |
|---------|-----------|-----------|---------------|
| fiscus | yes | yes | 2026-03-06 |
| penchan-co | yes | yes | 2026-03-07 |
| workspace-redesign | yes | yes | 2026-03-07 |
| ... | ... | ... | ... |

## Skills (60+ registered)
| Skill | SKILL.md | Category |
|-------|---------|----------|
| crypto-research | yes | research |
| news-toolkit | yes | research |
| ... | ... | ... |

## Memory
- Daily notes: 41 files (2026-02-23 to 2026-03-06)
- Topic files: (list after Phase 2.1)
- Archive: memory/archive/
```

2. Add a weekly cron task or end-of-session step to regenerate catalog.md (scan directories, check modification dates). This can be done by a Sonnet sub-agent in under 30 seconds.

3. Add catalog.md to the boot sequence as an optional reference: "If unsure what files exist, read catalog.md." It should NOT be loaded every session — only when discovery is needed.

**Why lowest risk**: This is a new file that adds capability. It does not modify any existing file or process. If it becomes stale or wrong, the worst outcome is that Pingu ignores it and does a directory scan as it does today.

**Rollback**: Delete catalog.md. Everything reverts to current behavior.

### 2.3 Establish Naming Conventions (as a policy document)

**Problem**: Daily notes already follow YYYY-MM-DD.md, but on 2026-03-06 there are 15+ files with inconsistent slug formats (some have timestamps like `0356`, some have topic slugs like `binance-guide`, some have both). There is no enforced convention for project directory names, decision records, or topic files.

**Change**: Write a `CONVENTIONS.md` file at the workspace root documenting the standard naming patterns.

**Implementation**:

```markdown
# CONVENTIONS.md — Naming & Structure Rules

## Daily Notes
- Format: `memory/YYYY-MM-DD.md` (one canonical daily note per day)
- Sub-session notes: `memory/YYYY-MM-DD-<slug>.md` (lowercase, hyphenated slug)
- Slug should describe the session topic, not the time: prefer `binance-guide` over `0813`

## Project Directories
- Format: `projects/<lowercase-hyphenated-name>/`
- Always contain: `context.md` + `status.md`
- Sub-directories as needed: `research/`, `decisions/`, `drafts/`, `workflows/`, `artifacts/`

## Decision Records
- Format: `projects/<name>/decisions/YYYY-MM-DD-<slug>.md`
- Use the ADR template from AGENTS.md

## Topic Memory Files
- Format: `memory/topics/<lowercase-hyphenated-topic>.md`
- Each file covers one coherent topic

## Skills
- Format: `skills/<lowercase-hyphenated-name>/SKILL.md`
- Auxiliary files alongside SKILL.md in the same directory

## General Rules
- All filenames: lowercase, hyphens for word separation, no spaces, no underscores
- Dates always ISO 8601: YYYY-MM-DD
- English slugs preferred (even for Chinese-language content) for path-level naming
```

**Why lowest risk**: This is a policy document. It does not change any existing file. It establishes expectations for future file creation and gives guidance for gradual cleanup of existing files.

**Rollback**: Delete CONVENTIONS.md. Naming reverts to ad-hoc convention.

### 2.4 Add Frontmatter to Key System Files

**Problem**: When Pingu (or a cron script) needs to understand a file's metadata (when it was last updated, what state it is in, who owns it), it must either read the entire file or guess from the filename.

**Change**: Add a minimal YAML frontmatter block to the top of key files, starting with the ones most frequently read or scanned.

**Implementation** — apply to these files first:
- All `projects/*/status.md`
- All `projects/*/context.md`
- All `memory/topics/*.md` (created in 2.1)

Frontmatter schema:

```yaml
---
type: status | context | topic | decision | daily-note
project: <project-slug>  # omit for system files
status: active | paused | blocked | archived  # for status.md
updated: 2026-03-07
---
```

**Why lowest risk**: Frontmatter is invisible to the document's content. Markdown renderers and readers ignore it. Adding it does not change how any existing process reads the file.

**Rollback**: Strip the frontmatter blocks. A one-line sed command per file.

### 2.5 Formalize the Boot Sequence as a Deterministic List

**Problem**: The current boot sequence in AGENTS.md is described in natural language with conditional logic ("if brain.md has orchestration tasks... read ORCHESTRATION.md"). This is clear to a human reader but leaves room for agent interpretation. Different sessions might load slightly different file sets.

**Change**: Add an explicit, ordered file list to the boot sequence section of AGENTS.md, keeping the existing natural-language description as context.

**Implementation** — append to the existing boot sequence section:

```markdown
### Deterministic Boot Manifest
Always load in this order, every main session:
1. `SOUL.md`
2. `USER.md`
3. `MEMORY.md` (index only — read topic files on-demand)
4. `memory/YYYY-MM-DD.md` (today)
5. `memory/YYYY-MM-DD.md` (yesterday, if exists)
6. `brain.md`
7. **Conditional**: if `brain.md` shows orchestration tasks or any `projects/*/status.md` has `in_progress` → load `ORCHESTRATION.md`

For Discord/Telegram threads: only steps 1, 2, 4, 5 + thread context.
For sub-agents: only the task envelope contents (no system files).
```

**Why lowest risk**: This codifies what is already happening. It does not change behavior — it makes current behavior explicit and verifiable.

**Rollback**: Remove the manifest section. Behavior unchanged since the natural-language description still exists.

---

## 3. Phase 2 — Week 3-4: Medium Risk Improvements

These changes modify existing files or introduce new processes. They build on Phase 1 foundations.

### 3.1 Implement Session-End State Freezing

**Problem**: When a session ends, conclusions, decisions, and new context are scattered across the conversation history. The next session starts cold and must either re-derive everything or hope it was written to a file mid-conversation. Important nuances get lost.

**Change**: Introduce a mandatory "state freeze" step at session end (or before session is expected to end due to context window limits).

**Implementation**:
1. Add to AGENTS.md under a new "Session Lifecycle" section:

```markdown
### Session End Protocol
Before ending a productive session (or when context window is >80% full):
1. Write conclusions to the appropriate file:
   - New facts about Penchan or preferences → `memory/topics/<topic>.md`
   - Project decisions → `projects/<name>/status.md` or `decisions/`
   - System learnings → `projects/openclaw/debug-notes.md`
2. Update `brain.md` if task status changed
3. Write a 2-3 line summary to today's `memory/YYYY-MM-DD.md`:
   `### Session [HH:MM] — [topic]`
   `- [key conclusion 1]`
   `- [key conclusion 2]`
4. If session was cut short, note the unfinished thread in brain.md under ⏳
```

2. Update the `session-retro` workflow to enforce this checklist.

**Dependencies**: Requires Phase 1 topic files (2.1) to exist as write targets.

**Risk**: If the freeze step is too burdensome, Pingu might skip it under token pressure or rush. Mitigation: keep it to 3-4 writes maximum. The freeze is about discipline, not comprehensiveness.

**Rollback**: Remove the session-end protocol from AGENTS.md. Sessions revert to current "write if you remember" behavior.

### 3.2 Implement Append-Only Decision Log per Project

**Problem**: Decisions are currently embedded in `status.md` progress history. Over time, the history section grows, old decisions get trimmed during size management, and the reasoning behind choices is lost.

**Change**: For projects with active decision-making, create a dedicated `decisions/` directory and move to append-only decision records.

**Implementation**:
1. For each active project that currently has decisions in status.md, create `projects/<name>/decisions/` and extract existing decisions into date-prefixed files.
2. Update the AGENTS.md writing rules:

```markdown
### Decision Recording
- Lightweight decisions (reversible, low-impact): stay in `status.md` progress history
- Significant decisions (irreversible, cross-project, strategy changes): → `projects/<name>/decisions/YYYY-MM-DD-<slug>.md`
- Decision files are append-only: never edit a past decision. If superseded, create a new file referencing the old one.
- `status.md` links to decision files: `→ decisions/YYYY-MM-DD-<slug>.md`
```

3. The ADR template already exists in AGENTS.md. No new template needed.

**Dependencies**: None strict, but works better with CONVENTIONS.md (2.3) for naming consistency.

**Risk**: Over-formalization. If every small decision requires a separate file, the `decisions/` directories will bloat with trivial records. Mitigation: the "lightweight vs significant" threshold is explicitly defined. Only strategy/architecture/irreversible decisions go into separate files.

**Rollback**: Merge decision files back into status.md. This is labor-intensive but not destructive.

### 3.3 Automated Memory Promotion (Daily Notes → Permanent Memory)

**Problem**: The "Memory Compression SOP" in AGENTS.md describes a manual joint-rewrite process. In practice, this happens irregularly. Facts that should be in long-term memory stay buried in daily notes.

**Change**: Create a weekly cron job that performs structured memory promotion.

**Implementation**:
1. Create a new workflow: `workflows/memory-promotion.md`

```markdown
# Memory Promotion Workflow
Trigger: weekly cron (Sunday night) or manual

## Steps
1. Read `MEMORY.md` (index)
2. Read all `memory/topics/*.md`
3. Read daily notes from the past 7 days
4. For each daily note entry, classify:
   - **Promote**: Stable fact or reusable knowledge → update relevant topic file
   - **Archive**: One-time event, already reflected in project files → leave in daily note
   - **Discard**: Noise, superseded info → leave (daily notes are append-only, never delete)
5. Update MEMORY.md index if new topic files were created
6. Update topic file frontmatter `updated:` dates
7. Write summary of promotions to `memory/YYYY-MM-DD.md`
```

2. Register this in `cron/` and assign to the existing `system/weekly-cleanup` cron (which already has MEMORY.md write permission per AGENTS.md).

**Dependencies**: Requires topic files from Phase 1 (2.1) to exist. Requires frontmatter (2.4) for the `updated:` tracking.

**Risk**: Aggressive promotion could duplicate information or create topic file sprawl. Mitigation: the workflow explicitly says "update existing topic file" before "create new topic file." The cron also produces a summary that Pingu can audit.

**Rollback**: Disable the cron. Memory stays in daily notes longer but is not lost.

### 3.4 Clean Up Existing File Drift

**Problem**: The workspace has accumulated files in inconsistent locations (e.g., `debug_note.md` at root, `tweet.txt` at root, various backup files like `openclaw.json.bak.*`).

**Change**: Systematically move misplaced files to their correct homes according to the file placement rules in AGENTS.md and the new CONVENTIONS.md.

**Implementation**:
1. Use catalog.md (from 2.2) to identify files that do not belong where they are.
2. For each misplaced file:
   - Determine correct location per AGENTS.md file placement rules
   - Move the file (use `mv`, not copy-delete)
   - If any other file references the old path, update the reference
3. Remove or archive stale files:
   - `openclaw.json.bak.*` → archive or delete (confirm with Penchan)
   - `debug_note.md` at root → merge into `projects/openclaw/debug-notes.md`
   - `tweet.txt` → move to `drafts/` or relevant project
   - `test-formula.png`, `test-integral.png` → move to relevant project or `media/`

**Dependencies**: Requires CONVENTIONS.md (2.3) for target naming. Requires catalog.md (2.2) for inventory.

**Risk**: Moving files can break references in other files, Discord messages, or cron configurations that use absolute paths. Mitigation: before moving, grep the entire workspace for the filename to find references. Update references before moving. Keep a `migration-log.md` documenting every move so broken references can be traced.

**Rollback**: Move files back to their original locations using the migration log.

---

## 4. Phase 3 — Month 2+: Longer-Term Aspirational Changes

These changes are higher complexity and should only be attempted after Phases 1-2 are stable and validated.

### 4.1 Searchable Index (catalog.sqlite or Embeddings)

**Problem**: As the workspace grows, a static catalog.md becomes insufficient for discovery. Pingu needs to answer questions like "what did we decide about rate limiting?" or "where is the Haedal research?"

**Change**: Build a lightweight SQLite index (or extend the existing `memory/main.sqlite`) that indexes all markdown files by path, title, frontmatter fields, and full-text content.

**Implementation sketch**:
- Cron job runs nightly: scan all `.md` files → extract frontmatter + first 500 chars → upsert into SQLite FTS5 table
- Pingu can query: `SELECT path, snippet FROM files_fts WHERE files_fts MATCH 'rate limiting'`
- Optionally add vector embeddings later (sqlite-vec extension) for semantic search

**Dependencies**: Requires frontmatter (2.4) to be widely adopted for structured filtering. Requires the file naming from CONVENTIONS.md (2.3) for consistent path parsing.

**Why defer**: The current workspace has ~23 projects, ~60 skills, and ~41 daily notes. This is still navigable with grep and directory listing. The index becomes essential at 2-3x this scale. Building it now is fine as a background project, but it should not block Phases 1-2.

**Rollback**: Delete the SQLite index. Everything remains discoverable via grep and catalog.md.

### 4.2 Hierarchical Memory Tiers with Automated Decay

**Problem**: Not all memories are equally valuable. Currently, topic files will grow over time without a mechanism to fade obsolete information.

**Change**: Implement a three-tier memory system with explicit lifetimes:

```
Tier 1 — Permanent: Penchan's identity, preferences, relationships, system architecture
Tier 2 — Active: Current project context, recent tool discoveries, active workflows (review quarterly)
Tier 3 — Ephemeral: Session-specific notes, transient tool outputs (auto-archive after 30 days if not promoted)
```

**Implementation sketch**:
- Add a `tier: 1|2|3` field to frontmatter
- Memory promotion workflow (3.3) checks tier-2 files not accessed in 90 days → prompt for review or downgrade
- Tier-3 content (daily notes older than 30 days with no promoted content) → move to `memory/archive/`
- Archive is never deleted, just excluded from search and boot sequence

**Why defer**: This requires the promotion workflow (3.3) to be running reliably first. It also requires enough time to pass that decay becomes relevant. Starting this before having months of data is premature.

### 4.3 Johnny Decimal-Inspired Numbering for Top-Level Directories

**Problem**: The workspace root currently has many directories with no inherent ordering: `agents/`, `brain.md`, `browser/`, `canvas/`, etc. Research suggests Johnny Decimal-style prefixes improve discoverability.

**Change**: Optionally prefix top-level directories with category numbers:

```
00-system/     (SOUL.md, AGENTS.md, USER.md, MEMORY.md, brain.md, ORCHESTRATION.md)
10-projects/   (all project directories)
20-memory/     (daily notes, topics, archive)
30-skills/     (all skills)
40-workflows/  (workflow definitions)
50-media/      (images, documents, inbound files)
90-archive/    (completed projects, old configs)
```

**Why defer (and why it might not happen)**: This is the highest-risk change in the entire plan. Every path reference in every file, every cron job, every sub-agent prompt, and every external tool configuration would need to be updated simultaneously. The benefit (slightly easier directory scanning) does not justify the migration cost given that the current flat structure is well-understood. If the team decides this is worth doing, it should be a dedicated migration sprint with full workspace backup and a comprehensive find-and-replace operation. I recommend NOT doing this unless the flat structure actively causes problems at scale.

**Rollback**: Rename directories back. But this means updating all references twice — forward and back. Very expensive.

### 4.4 Sub-Agent Scoped Memory (Namespaced Topic Files)

**Problem**: Sub-agents currently read only what the orchestrator passes them. They cannot build on knowledge from previous sub-agent runs on the same topic.

**Change**: Allow sub-agents to read (not write) topic files from `memory/topics/` that are tagged with their task's project namespace.

**Implementation sketch**:
- Task Envelope gains a `memory_scope: [list of topic file paths]` field
- Sub-agent reads those files as part of its context bundle
- Sub-agent writes findings to its output file (as today)
- Orchestrator decides whether to promote sub-agent findings to topic files

**Why defer**: The current system works because sub-agents are intentionally stateless. Adding memory scope increases their context window cost and risks leaking irrelevant information. Only adopt this if sub-agents repeatedly fail tasks because they lack context that exists in topic files.

---

## 5. Migration Risks

### Phase 1 Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| MEMORY.md split loses information | Low | High | Verify every line in current MEMORY.md appears in exactly one topic file. Diff check before deleting old content. |
| Pingu fails to read topic files when needed | Medium | Medium | Keep the MEMORY.md index entries descriptive enough that Pingu knows when to follow a link. First 2 weeks: monitor whether Pingu actually loads topic files. |
| catalog.md becomes stale immediately | Medium | Low | It is advisory, not authoritative. Staleness is annoying but not dangerous. Weekly rebuild solves it. |
| CONVENTIONS.md is ignored | Medium | Low | It is a policy. Enforcement requires Pingu to check naming before creating files. Add a reminder in AGENTS.md's "file creation" section. |
| Frontmatter causes parsing issues | Low | Low | YAML frontmatter is standard markdown. All tools handle it. Test with one file first. |

### Phase 2 Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| State freeze step gets skipped under token pressure | High | Medium | Make it short (3-4 writes max). The freeze should take <2 minutes. If Pingu is out of context, at minimum write one line to brain.md. |
| Decision files proliferate, creating noise | Medium | Medium | Clear threshold: only irreversible or cross-project decisions get files. Everything else stays inline in status.md. |
| Memory promotion creates duplicates | Medium | Medium | The promotion workflow must check if information already exists in the target topic file before writing. |
| File moves break references | Medium | High | Grep for the filename in the entire workspace before moving. Update references first, then move. Log everything. |

### Phase 3 Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| SQLite index corruption | Low | Medium | The index is always rebuildable from the source markdown files. Corruption = rebuild, not data loss. |
| Johnny Decimal renaming breaks everything | High | Very High | Do not attempt without full backup, comprehensive path audit, and a dedicated migration session with no other work. Or skip entirely. |
| Automated decay deletes something important | Medium | High | Archive, never delete. Decay moves files to archive where they can always be retrieved. |

---

## 6. Rollback Strategy

Every phase has a clean rollback path. This is the most important property of the migration plan.

### Phase 1 Rollback (per change)

| Change | Rollback Method | Time to Rollback |
|--------|----------------|-----------------|
| 2.1 MEMORY.md → index + topics | Concatenate topic files back into MEMORY.md. One script. | 5 minutes |
| 2.2 catalog.md | Delete `catalog.md`. | 10 seconds |
| 2.3 CONVENTIONS.md | Delete `CONVENTIONS.md`. | 10 seconds |
| 2.4 Frontmatter | Strip frontmatter blocks from affected files. One `sed` command per file. | 2 minutes |
| 2.5 Boot manifest | Remove the manifest section from AGENTS.md. Natural-language boot description still works. | 1 minute |

### Phase 2 Rollback

| Change | Rollback Method | Time to Rollback |
|--------|----------------|-----------------|
| 3.1 State freeze | Remove protocol from AGENTS.md. Sessions revert to current behavior. | 1 minute |
| 3.2 Decision logs | Merge decision files back into status.md. | 10-30 minutes per project |
| 3.3 Memory promotion | Disable cron. Daily notes accumulate but nothing is lost. | 1 minute |
| 3.4 File drift cleanup | Use migration-log.md to move files back to original locations. | 30-60 minutes |

### Phase 3 Rollback

| Change | Rollback Method | Time to Rollback |
|--------|----------------|-----------------|
| 4.1 SQLite index | Delete the index file. Grep + catalog.md still works. | 10 seconds |
| 4.2 Memory tiers | Remove tier field from frontmatter. All files remain accessible. | 5 minutes |
| 4.3 JD numbering | Rename directories back + update all references. | 2-4 hours (dangerous) |
| 4.4 Sub-agent memory scope | Remove `memory_scope` from task envelopes. Sub-agents revert to stateless. | 5 minutes |

**Global rollback**: If the entire migration is failing, the nuclear option is a git reset to the pre-migration commit. This is why we should commit a clean snapshot of the workspace before starting Phase 1.

---

## 7. Testing Criteria

How do we know each change actually helped? Each change needs observable, concrete success criteria — not vague "feels better."

### Phase 1 Success Criteria

| Change | Success Metric | How to Measure | Timeframe |
|--------|---------------|----------------|-----------|
| 2.1 MEMORY.md index | MEMORY.md stays under 500 bytes for 2 weeks. Pingu loads topic files when discussing relevant topics (verify in 5 random sessions). | Read MEMORY.md size. Review session logs for topic file reads. | 2 weeks |
| 2.2 catalog.md | Pingu references catalog.md at least once when asked "do we have a file for X?" or when creating a new file. Zero cases of duplicate file creation in 2 weeks. | Review session logs. Track file creation events. | 2 weeks |
| 2.3 CONVENTIONS.md | All new files created after the convention follows the naming rules. Check weekly: `ls memory/ projects/ skills/` for non-conforming names. | Weekly audit of new file names. | 4 weeks |
| 2.4 Frontmatter | Cron/scripts can parse frontmatter to list "all active projects" or "all files updated this week" without reading full file content. | Run a frontmatter extraction script and verify output. | 1 week |
| 2.5 Boot manifest | Every main session loads the exact same file set (verifiable by checking the first few tool calls). No variance in boot behavior across 10 sessions. | Compare first 5 tool calls across 10 sessions. | 2 weeks |

### Phase 2 Success Criteria

| Change | Success Metric | How to Measure | Timeframe |
|--------|---------------|----------------|-----------|
| 3.1 State freeze | Next-day sessions can resume context without re-reading the full prior conversation. Penchan reports less repetition in follow-up sessions. | Ask Penchan for subjective rating (1-5) on session continuity after 2 weeks. | 2-4 weeks |
| 3.2 Decision logs | When Penchan asks "why did we decide X?", Pingu finds the answer in <30 seconds by reading the decision file, rather than searching through status.md history. | Time-to-answer for decision recall questions. | 4 weeks |
| 3.3 Memory promotion | At least 3 facts per week are promoted from daily notes to topic files. MEMORY.md index grows by 0-2 entries per month (not more — stability is the goal). | Count promotions in weekly cron output. Check MEMORY.md size monthly. | 4-8 weeks |
| 3.4 File cleanup | Root directory has <10 non-system files. No orphan files identified in catalog.md refresh. | Count root files. Run catalog rebuild and check for unregistered files. | 2 weeks |

### Phase 3 Success Criteria

| Change | Success Metric | How to Measure |
|--------|---------------|----------------|
| 4.1 SQLite index | Full-text search returns relevant results for 8/10 test queries. Search completes in <1 second. | Run 10 test queries, measure precision and latency. |
| 4.2 Memory tiers | Topic files do not grow beyond 2KB each after 3 months. Archived content is still retrievable when specifically asked. | Monthly size check. Manual retrieval test. |
| 4.3 JD numbering | All agent sessions correctly resolve file paths. Zero "file not found" errors in 1 week post-migration. | Monitor error logs for path failures. |
| 4.4 Sub-agent memory scope | Sub-agent task success rate improves by >5% on tasks where memory scope is provided vs. baseline. | Compare success rates in weekly retro. |

---

## 8. Dependency Map

```
Phase 1 (all independent of each other, can be done in any order):
  2.1 MEMORY.md → index + topics     ←── no deps
  2.2 catalog.md                      ←── no deps
  2.3 CONVENTIONS.md                  ←── no deps
  2.4 Frontmatter on key files        ←── no deps (but easier after 2.1 creates topic files)
  2.5 Boot manifest                   ←── no deps (but references 2.1's index approach)

Phase 2 (depends on Phase 1 foundations):
  3.1 State freeze protocol           ←── depends on 2.1 (topic files as write targets)
  3.2 Decision logs                   ←── depends on 2.3 (naming conventions for decision files)
  3.3 Memory promotion workflow       ←── depends on 2.1 (topic files) + 2.4 (frontmatter for tracking)
  3.4 File drift cleanup              ←── depends on 2.2 (catalog for inventory) + 2.3 (conventions for target names)

Phase 3 (depends on Phase 2 stability):
  4.1 SQLite index                    ←── depends on 2.4 (frontmatter) + 2.3 (naming) being widely adopted
  4.2 Memory tiers                    ←── depends on 3.3 (promotion workflow running reliably)
  4.3 JD numbering                    ←── depends on ALL of Phase 1 + 2 being stable. Independent of 4.1/4.2.
  4.4 Sub-agent memory scope          ←── depends on 2.1 (topic files exist) + 2.4 (frontmatter for scoping)
```

### Visual Dependency Flow

```
                    ┌─────────────────────────────────────────────┐
                    │              PHASE 1 (Week 1-2)             │
                    │  All items independent, parallel execution  │
                    │                                             │
                    │  [2.1 Memory Index]  [2.2 Catalog]          │
                    │  [2.3 Conventions]   [2.4 Frontmatter]      │
                    │  [2.5 Boot Manifest]                        │
                    └──────────┬──────────────────┬───────────────┘
                               │                  │
                    ┌──────────▼──────────────────▼───────────────┐
                    │              PHASE 2 (Week 3-4)             │
                    │                                             │
                    │  [3.1 State Freeze]  ← needs 2.1            │
                    │  [3.2 Decision Logs] ← needs 2.3            │
                    │  [3.3 Promotion]     ← needs 2.1 + 2.4     │
                    │  [3.4 File Cleanup]  ← needs 2.2 + 2.3     │
                    └──────────┬──────────────────┬───────────────┘
                               │                  │
                    ┌──────────▼──────────────────▼───────────────┐
                    │              PHASE 3 (Month 2+)             │
                    │                                             │
                    │  [4.1 SQLite Index]  ← needs 2.3 + 2.4     │
                    │  [4.2 Memory Tiers]  ← needs 3.3           │
                    │  [4.3 JD Numbering]  ← needs ALL stable    │
                    │  [4.4 Agent Memory]  ← needs 2.1 + 2.4    │
                    └─────────────────────────────────────────────┘
```

### Recommended Execution Order Within Phases

**Phase 1** (do in this order for smoothest flow, though parallel is safe):
1. 2.3 CONVENTIONS.md — establishes the rules everything else follows
2. 2.1 MEMORY.md split — biggest impact, unlocks Phase 2 items
3. 2.4 Frontmatter — easy to add while creating topic files in 2.1
4. 2.5 Boot manifest — codifies the new index-based loading
5. 2.2 catalog.md — requires scanning the workspace, do last when structure is settled

**Phase 2** (do in this order):
1. 3.1 State freeze — immediate daily impact, low effort
2. 3.2 Decision logs — formalize what should already be happening
3. 3.3 Memory promotion — requires topic files to have been in use for 1-2 weeks
4. 3.4 File cleanup — do last, after all conventions and catalog are established

---

## 9. Pre-Migration Checklist

Before starting Phase 1, complete these safety steps:

- [ ] **Git commit**: Commit the entire workspace as-is with message "pre-workspace-redesign snapshot"
- [ ] **Verify backup**: Confirm the commit includes all untracked files (git add all, then commit)
- [ ] **Notify Penchan**: Confirm Phase 1 scope and get explicit approval to proceed
- [ ] **No concurrent major tasks**: Do not start Phase 1 during a high-pressure work period (e.g., content deadline, urgent research)
- [ ] **Test topic file creation**: Create ONE topic file manually (`memory/topics/test.md`), verify Pingu can read it, then delete it

---

## 10. What This Plan Deliberately Avoids

1. **No directory restructuring in Phase 1 or 2**. The current `projects/`, `skills/`, `memory/`, `workflows/` layout is fine. Moving directories is the highest-risk operation possible.

2. **No new tooling requirements**. Everything in Phase 1 and 2 uses existing capabilities: markdown files, cron jobs, sub-agent tasks. No new MCP servers, no SQLite setup, no embedding pipelines.

3. **No changes to SOUL.md or USER.md**. These files are identity-level. They are not part of the workspace management problem.

4. **No changes to the orchestration model**. ORCHESTRATION.md is only 10 days old and already well-designed. Let it stabilize before modifying.

5. **No automation-first approach**. Every process starts as a manual/semi-manual step, proves its value, then gets automated. This avoids building automation for workflows that turn out to be wrong.

---

*Written by: Migration Strategist (Opus sub-agent)*
*Date: 2026-03-07*
*Status: Ready for debate synthesis*
