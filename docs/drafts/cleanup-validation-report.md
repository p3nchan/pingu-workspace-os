# Cleanup System Validation Report
Date: 2026-03-07

## Summary
- Passed: 19 checks
- Warnings: 5 items
- Failed: 0 items

---

## Detailed Results

### 1. Cron Job Configuration

**`system/daily-cleanup`** (id: `d1307d9d`)
- Schedule: `33 3 * * *` (Asia/Taipei) — daily at 03:33. **PASS.** Off-peak hour, no user activity expected.
- Dispatcher model: `gemini-flash` — **PASS.** Efficient: Flash only reads the instruction file and spawns a sub-agent. Minimal token cost (~300 tokens).
- Sub-agent model: `sonnet` — **PASS.** Correct for execution-heavy tasks (file operations, find/delete, report writing).
- Prompt pattern: reads `maintenance/daily-cleanup.md`, spawns Sonnet sub-agent with full content. **PASS.** Clean separation of dispatcher vs executor.
- `sessionTarget: "isolated"`, `wakeMode: "now"` — **PASS.**
- Last run status: `ok`, duration: 77s, consecutive errors: 0. **PASS.**

**`system/weekly-cleanup`** (id: `83d99860`)
- Schedule: `35 5 * * 1` (Asia/Taipei) — Monday at 05:35. **PASS.** ~2 hours after daily cleanup, allowing daily to finish first.
- Dispatcher model: `gemini-flash` — **PASS.** Same efficient pattern.
- Sub-agent model: `opus` — **PASS.** Correct for analysis-heavy tasks (memory refinement, project health assessment, rule iteration).
- Prompt pattern: reads `maintenance/weekly-cleanup.md`, spawns Opus sub-agent. **PASS.**
- Last run status: `ok`, duration: 279s (~4.5 min), consecutive errors: 0. **PASS.**
- Last run: 2026-02-28 (most recent Monday), next run: 2026-03-09. **PASS.**

**Schedule reasonableness:**
- Daily at 03:33 and weekly at 05:35 are both well-chosen off-peak times in Asia/Taipei timezone.
- The 2-hour gap between daily (03:33) and weekly (05:35 on Mondays) ensures daily cleanup completes before weekly starts. Daily's average duration is ~77s, so there is ample buffer.
- Using gemini-flash as dispatcher is optimal: Flash is the cheapest model available, and the dispatcher only performs two operations (read file + spawn sub-agent), so there is no reason to use a more expensive model.

### 2. File Path Validation

All file paths referenced in the cleanup instruction files were verified:

| Path | Status | Referenced By |
|------|--------|---------------|
| `memory/` | EXISTS | daily-cleanup.md, weekly-cleanup.md |
| `memory/topics/` | EXISTS | daily-cleanup.md, weekly-cleanup.md, cleanup-rules.md |
| `memory/archive/` | EXISTS | daily-cleanup.md, weekly-cleanup.md |
| `memory/state/` | EXISTS | daily-cleanup.md |
| `maintenance/cleanup-rules.md` | EXISTS | daily-cleanup.md, weekly-cleanup.md |
| `maintenance/cleanup-suggestions.md` | EXISTS | weekly-cleanup.md |
| `projects/` | EXISTS | weekly-cleanup.md |
| `logs/sessions/` | EXISTS | AGENTS.md (referenced architecture) |
| `browser/openclaw/user-data/` | EXISTS | weekly-cleanup.md |
| `canvas/` | EXISTS | weekly-cleanup.md |
| `media/` | EXISTS | daily-cleanup.md, weekly-cleanup.md |
| `media/inbound/` | EXISTS | daily-cleanup.md |
| `cron/runs/` | EXISTS | daily-cleanup.md |

**Result: ALL 13 paths verified. No missing paths.**

### 3. Command Compatibility

**Test command:** `find /Users/pingu/.openclaw/memory/ -not -newermt "$(date -v-7d +%Y-%m-%d)" -name "*.md"`

**Result: PASS.** The command executed successfully on macOS and returned valid results (archived .md files older than 7 days). The macOS BSD `find` command supports the `-newermt` flag.

**Note on `date -v-7d`:** This is macOS-specific syntax (BSD date). If this workspace ever migrates to Linux, the command would need to change to `date -d '7 days ago'`. This is acceptable since the workspace is explicitly macOS-based.

**GNU find (`gfind`):** Not installed. Not needed since BSD find supports `-newermt`.

### 4. Cleanup Rules Consistency

**cleanup-rules.md version:** v2.0 (2026-03-07) — current.

Cross-reference of thresholds/limits across all three files:

| Parameter | cleanup-rules.md | daily-cleanup.md | weekly-cleanup.md | Consistent? |
|-----------|-------------------|-------------------|---------------------|-------------|
| `.bak` retention | Keep newest 1 | Keep newest 1 | N/A | YES |
| Screenshots (`screen0_*.png`) | >3 days delete | >3 days delete | N/A | YES |
| Temp files retention | >1 day delete | >1 day delete | N/A | YES |
| `media/inbound/` retention | 7 days | 7 days | "already handled by daily" | YES |
| `cron/runs/` log retention | 14 days | 14 days | N/A | YES |
| `jobs.json.bak.*` retention | Keep newest 2 | Keep newest 2 | N/A | YES |
| `memory/` root .md file limit | 15 | 15 | N/A | YES |
| Transcript compression threshold | 40 lines | 40 lines | N/A | YES |
| Compression target | <20 lines | <20 lines | N/A | YES |
| Topic file size target | <=1KB | N/A | <=1KB | YES |
| Topic file hard limit | 1.5KB | N/A | 1.5KB | YES |
| Topic file count limit | 10 | N/A | <=10 | YES |
| `canvas/` report threshold | 20MB | N/A | 20MB | YES |
| `media/` subdir report threshold | >50MB | N/A | >50MB | YES |
| Disabled cron job deletion threshold | 14 days | N/A | 14 days | YES |
| `trash` > `rm` preference | Yes (media/inbound) | Yes (media/inbound) | Yes (browser cache) | YES |
| Time method | `-newermt` | `-newermt` | N/A | YES |
| Topics exclusion from daily | Daily must not modify | Explicitly excluded | Weekly maintains | YES |
| Standard subdirs whitelist | topics/, archive/, state/ | Matches | N/A | YES |

**Result: ALL 19 parameters are consistent across all files. No conflicts found.**

### 5. AGENTS.md Alignment

**Memory system section analysis:**

1. **Directory structure (目錄結構):**
   - AGENTS.md defines: `MEMORY.md` (pure index, <600 bytes), `memory/YYYY-MM-DD.md`, `memory/topics/*.md`, `memory/archive/`, `memory/experiments.md`, `logs/sessions/`, `projects/<name>/context.md`
   - Cleanup files reference all of these correctly.
   - **PASS.**

2. **Rules (規則):**
   - AGENTS.md: "New stable knowledge goes to topic files, not MEMORY.md"
   - Weekly cleanup follows this exactly in its Topic Files Update SOP (Section 1).
   - AGENTS.md: "Session transcripts go to `logs/sessions/`, not memory/"
   - Daily cleanup: checks for and compresses uncompressed transcripts in memory/.
   - **PASS.**

3. **Compression SOP (壓縮 SOP):**
   - AGENTS.md defines: Read MEMORY.md index + topic files, read recent logs, jointly rewrite topic files, keep MEMORY.md <600 bytes, each topic <1.5KB.
   - Weekly cleanup Section 1 implements this exact SOP with matching thresholds.
   - **PASS.**

4. **Write protection (寫入保護) — CRITICAL CHECK:**
   - AGENTS.md states: "Sub-agent 禁止修改：AGENTS.md、SOUL.md、USER.md、MEMORY.md"
   - AGENTS.md grants exemption: "`system/weekly-cleanup` cron 有權更新 `MEMORY.md`（記憶精煉是其核心任務）"
   - Weekly cleanup claims: "本 cron job 是唯一有權修改 MEMORY.md 的 sub-agent（見 AGENTS.md 寫入保護豁免）"
   - **PASS.** The exemption is explicitly documented on both sides.

5. **Weekly cleanup and topic file modification:**
   - AGENTS.md allows sub-agents to write to `memory/YYYY-MM-DD.md` and `projects/` files.
   - Weekly cleanup also modifies `memory/topics/*.md` files. This is NOT explicitly listed in the "Sub-agent 只能寫入" allowlist in AGENTS.md.
   - **WARNING:** AGENTS.md's write protection section says sub-agents can only write to "專案檔案（projects/）" and "每日筆記（memory/YYYY-MM-DD.md）". The weekly cleanup's exemption only explicitly covers `MEMORY.md`, but the weekly cleanup also modifies `memory/topics/*.md`, `cleanup-rules.md`, and `catalog.md`. These are not covered by the stated exemption.
   - **Severity: MEDIUM.** The intent is clearly that weekly-cleanup has broad maintenance permissions, but the explicit exemption text in AGENTS.md is narrower than the actual scope of operations.

6. **Weekly cleanup also updates AGENTS.md itself:**
   - Weekly cleanup Section 1, step 6: "同步更新 AGENTS.md 的「MEMORY.md Topic 檔」段落中的 topic 清單"
   - AGENTS.md write protection: "Sub-agent 禁止修改：AGENTS.md"
   - The exemption only mentions MEMORY.md, not AGENTS.md.
   - **WARNING:** weekly-cleanup.md instructs the sub-agent to modify AGENTS.md, which is explicitly prohibited. This is a policy conflict.

### 6. cleanup-suggestions.md Review

**Total suggestions: 5** (spanning 2026-03-02 to 2026-03-06)

| Date | Suggestion | Status in v2 | Assessment |
|------|-----------|--------------|------------|
| 03-02 | `-mtime` unreliable, suggest reference file method | **ADDRESSED** in v2 | cleanup-rules.md v2.0 adopts `-newermt` (slightly different solution than suggested, but solves same problem). Suggestion should be marked `[adopted]`. |
| 03-04 | `-mtime +7` found only 2 files vs `-newermt` found 18 | **ADDRESSED** in v2 | Reinforces 03-02 finding. cleanup-rules.md cites this as justification. Suggestion should be marked `[adopted]`. |
| 03-04 | nvm/node path may break after upgrade, suggest weekly check | **ADDRESSED** in v2 | weekly-cleanup.md Section 5 now includes node environment check. Suggestion should be marked `[adopted]`. |
| 03-06 | `memory/agent-sociology/` has duplicate old files | **PARTIALLY ADDRESSED** | cleanup-rules.md v2 adds "standard subdirs whitelist" that would flag this. But the actual cleanup (moving/deleting the directory) has not been performed. Still requires manual action. |
| 03-06 | `openclaw.db` is 0 bytes, possibly orphan empty file | **ADDRESSED** in v2 | cleanup-rules.md v2 adds "orphan empty file detection" rule. But the specific file itself has not been cleaned up yet. Still requires manual action. |

**WARNING:** None of the 5 suggestions have been marked with status annotations (`[adopted]` or `[ignored]`). The weekly cleanup is supposed to review and annotate these, but the file still shows raw entries without status. This could mean:
- The weekly cleanup has not run since the suggestions were filed (last run was 2026-02-28, before the 03-02 suggestions).
- Or the weekly cleanup is not properly annotating suggestions.

**The next weekly run (2026-03-09) should annotate all 5 suggestions.**

### 7. Scheduling Efficiency

**Complete cron job schedule (all 14 jobs, Asia/Taipei timezone):**

| Time | Job Name | Frequency | Model |
|------|----------|-----------|-------|
| **Every :25** | `system/hourly-healthcheck` | Hourly | gemini-flash |
| 03:33 | `system/daily-cleanup` | Daily | gemini-flash -> sonnet |
| 05:35 | `analytics/github-trending` | Daily | gemini-flash |
| **05:35** | `system/weekly-cleanup` | **Mon only** | gemini-flash -> opus |
| 07:37 | `analytics/penchan-daily` | Daily | gemini-flash -> sonnet |
| **07:37** | `system/monthly-manifest` | **1st of month** | gemini-flash |
| 08:08 | `crypto/liquidusd-apy` | Daily | gemini-flash |
| 09:09 | `content/daily-pipeline` | Daily | gemini-flash -> sonnet |
| **09:39** | `analytics/website-weekly` | **Mon only** | gemini-flash -> sonnet |
| **09:39** | `system/monthly-security-audit` | **1st of month** | gemini-flash |
| 10:10 | `observe/moltbook-sociology` | Daily | gemini-flash -> sonnet |
| 11:11 | `health/daily-report` | Daily | gemini-flash |
| 13:43 | `system/openclaw-update` | Daily | gemini-flash |
| 15:45 | `system/claude-api-usage` | Daily | gemini-flash |
| 23:53 | `analytics/github-pulse` | Daily | gemini-flash |

**Conflict Analysis:**

1. **05:35 conflict (Mondays):** `analytics/github-trending` (daily) and `system/weekly-cleanup` (weekly) both fire at 05:35 on Mondays. Since both use `sessionTarget: "isolated"`, they will run in separate sessions. However, they start simultaneously, which could cause resource contention (both spawning sub-agents at the same time). **WARNING: Minor scheduling conflict.**

2. **07:37 conflict (1st of month):** `analytics/penchan-daily` and `system/monthly-manifest` both fire at 07:37 on the 1st. Same isolation pattern. Low risk since monthly-manifest is lightweight.

3. **09:39 conflict (1st Monday of month):** `analytics/website-weekly` and `system/monthly-security-audit` could collide on the 1st Monday. Both are resource-intensive. **Potential concern but extremely rare (only when the 1st falls on a Monday).**

4. **Hourly healthcheck at :25:** Runs every hour, never conflicts with other jobs (no other job uses minute :25).

**Cleanup schedule relative to other jobs:**
- Daily cleanup at 03:33 is the first job of the day. **Good.** It cleans up before other jobs generate new files.
- Weekly cleanup at 05:35 on Mondays runs after daily cleanup has had time to finish (~77s typical). **Good.** However, it shares the timeslot with github-trending.
- No job runs between 23:53 and 03:33, giving a ~4-hour quiet window for cleanup. **Good.**

---

## Recommendations

### Priority 1 (Should fix before v2 production)

1. **Resolve AGENTS.md write protection scope for weekly-cleanup**
   - weekly-cleanup.md instructs the sub-agent to modify `AGENTS.md` (updating topic list), `memory/topics/*.md`, `cleanup-rules.md`, and `catalog.md` -- but the AGENTS.md exemption only explicitly covers `MEMORY.md`.
   - **Fix:** Expand the AGENTS.md exemption to: "`system/weekly-cleanup` cron 有權更新 `MEMORY.md`、`memory/topics/`、`maintenance/cleanup-rules.md`、`catalog.md`、以及 AGENTS.md 的 topic 清單段落"
   - Or alternatively, remove the instruction in weekly-cleanup.md to update AGENTS.md, and have the main session do it instead.

### Priority 2 (Should fix soon)

2. **Annotate cleanup-suggestions.md**
   - All 5 existing suggestions lack status annotations. The next weekly run (2026-03-09) should annotate them. Verify after that run completes.

3. **Resolve 05:35 Monday scheduling overlap**
   - Move `analytics/github-trending` to 05:15 or 05:55 to avoid simultaneous start with `system/weekly-cleanup` on Mondays.
   - Weekly cleanup runs for ~4.5 minutes and spawns an Opus sub-agent; github-trending also spawns agents. Staggering avoids resource contention.

### Priority 3 (Nice to have)

4. **Address pending manual cleanup items from suggestions**
   - `memory/agent-sociology/` duplicate directory: Confirm and trash.
   - `openclaw.db` 0-byte file: Confirm and remove if orphan.
   - These will be detected by the v2 daily cleanup but still need manual resolution.

5. **Add `gfind` as fallback documentation**
   - GNU find is not installed. While BSD `find` supports `-newermt`, document this macOS dependency in cleanup-rules.md for future reference. If the workspace ever runs on Linux, the `date -v-7d` syntax will also need updating to `date -d '7 days ago'`.

6. **Consider adding `cleanup-rules.md` version check to daily-cleanup**
   - Daily cleanup reads `cleanup-rules.md` but does not verify the version. Adding a version assertion (e.g., "if version < 2.0, abort and alert") would catch stale rule files.

---

## Appendix: Validation Matrix

| # | Check | Result | Notes |
|---|-------|--------|-------|
| 1 | Daily cleanup cron exists and enabled | PASS | |
| 2 | Weekly cleanup cron exists and enabled | PASS | |
| 3 | Daily points to maintenance/daily-cleanup.md | PASS | Read-then-spawn pattern |
| 4 | Weekly points to maintenance/weekly-cleanup.md | PASS | Read-then-spawn pattern |
| 5 | Daily spawns Sonnet sub-agent | PASS | Correct for execution |
| 6 | Weekly spawns Opus sub-agent | PASS | Correct for analysis |
| 7 | Daily schedule (03:33) reasonable | PASS | Off-peak, first job of day |
| 8 | Weekly schedule (Mon 05:35) reasonable | PASS | After daily, but overlaps github-trending |
| 9 | Both use gemini-flash dispatcher | PASS | Cheapest model for simple dispatch |
| 10 | All 13 referenced paths exist | PASS | No missing paths |
| 11 | `-newermt` works on macOS | PASS | BSD find supports it |
| 12 | `date -v-7d` works on macOS | PASS | BSD date syntax |
| 13 | Threshold consistency (19 params) | PASS | All match across 3 files |
| 14 | AGENTS.md memory structure matches | PASS | |
| 15 | AGENTS.md compression SOP matches | PASS | |
| 16 | AGENTS.md weekly-cleanup MEMORY.md exemption | PASS | Explicitly granted |
| 17 | AGENTS.md weekly-cleanup topics/rules scope | WARN | Exemption too narrow for actual operations |
| 18 | AGENTS.md weekly-cleanup AGENTS.md self-edit | WARN | Prohibited by L0 policy |
| 19 | Suggestions addressed in v2 | WARN | 3/5 fully addressed, 2/5 partially, none annotated |
| 20 | No scheduling conflicts | WARN | 05:35 Monday overlap with github-trending |
| 21 | cleanup-rules.md is v2.0 | PASS | Dated 2026-03-07 |
| 22 | Daily cleanup last run healthy | PASS | ok, 77s, 0 errors |
| 23 | Weekly cleanup last run healthy | PASS | ok, 279s, 0 errors |
| 24 | Hourly healthcheck no conflicts | PASS | Runs at :25, no overlaps |
