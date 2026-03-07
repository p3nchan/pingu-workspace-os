# 失敗考古學家報告：OpenClaw Workspace 法醫分析

> 角色：失敗考古學家（Failure Archaeologist）
> 日期：2026-03-07
> 方法論：不理論化，找證據。每個放錯的檔案、每個孤兒文件、每個不一致都在說一個結構性失敗的故事。

---

## A. 現場證據：我找到的問題

### 發現 #1：`reference/` vs `references/` — 同義目錄共存

**證據：**
- `/Users/pingu/.openclaw/reference/` — 4 個檔案（`ai-strategy.md`, `local-deep-research-analysis.md`, `penchan-x-posts-archive.txt`, `pricing.md`）
- `/Users/pingu/.openclaw/references/` — 3 個檔案（`seo-architecture.md`, `translate-methodology.md`, `web-components.md`）
- 兩個目錄在 `skills/` 內也有不一致：24 個 skill 用 `references/`（複數），1 個（`mcp-builder`）用 `reference/`（單數）

**症狀：** Agent 需要找參考資料時無法預測路徑。必須同時檢查兩個目錄，或依賴 grep 搜尋，浪費 token。

**根因分析：** 沒有命名規範。`reference/` 可能是早期建立的，後來的檔案被放到 `references/`（英文慣例偏複數），但沒人回頭統一。`mcp-builder/reference/` 再次證明即使有多數派（`references/`），單一例外就能造成不一致。

**嚴重度：** 🟡 中 — 每次找資料會多花數秒 + token，但不會造成邏輯錯誤

---

### 發現 #2：X 內容草稿散落三個地方

**證據：**
- `/Users/pingu/.openclaw/drafts/x-threads/` — 2 個 X thread 草稿（2026-03-03）
- `/Users/pingu/.openclaw/projects/x-content/drafts/` — 1 個 AI-BTC thread 草稿 + 2 個 PNG 圖表（2026-03-06）
- `/Users/pingu/.openclaw/projects/p3nchan/pipeline/` — 4 個 post 草稿（2026-03-04）
- `/Users/pingu/.openclaw/pipeline/` — 1 個 final post（`20260304-post2-tokenized-gold-final.md`，與 `p3nchan/pipeline/20260304-post2-tokenized-gold.md` 是同一篇的不同版本）

**症狀：** 同類內容（X/social 內容草稿）分散在 4 個路徑。Agent 無法一眼找到「所有待發的草稿」。root `pipeline/` 存了一篇 final 版本，但 p3nchan 目錄裡有其草稿版，來源追溯斷裂。

**根因分析：** 三個結構性問題疊加：
1. `drafts/`（root）和 `projects/x-content/drafts/` 語義重疊——誰是 X 內容的正式家？
2. `pipeline/`（root）本應是「通用內容管線」，但實際只放了一篇文章的 final 版，而草稿卻在 `p3nchan/pipeline/`
3. `projects/x-content/` 本身沒有 `context.md` 或 `status.md`，是個無骨架的專案

**嚴重度：** 🔴 高 — 內容生產是高頻任務，每次都要猜「草稿在哪」直接浪費 token 和造成遺漏

---

### 發現 #3：`workspace/` 和 `workspace-codex/` — 殭屍複本

**證據：**
- `/Users/pingu/.openclaw/workspace/` 包含：
  - 過時的系統檔複本：`AGENTS.md`、`SOUL.md`、`USER.md`、`IDENTITY.md`、`HEARTBEAT.md`、`TOOLS.md`、`BOOTSTRAP.md`（**內容與 root 完全不同，是舊版**）
  - `x-archive-analysis/` — 34 個檔案（分析腳本 + JSON 資料），同時 `projects/p3nchan/x-archive-analysis/` 也有 11 個（是子集）
  - `memory/agent-sociology/2026-02-28-observation.md` — 一個孤立的 observation 複本
  - 有自己的 `.git/` 和 `.openclaw/workspace-state.json`

- `/Users/pingu/.openclaw/workspace-codex/` 包含：
  - 又一份系統檔複本：`AGENTS.md`、`SOUL.md`、`USER.md`、`IDENTITY.md`、`HEARTBEAT.md`、`TOOLS.md`
  - `REVIEW-REPORT-2026-02-28.md` — 一份孤立的 review 報告
  - 有自己的 `.git/` 和 `.openclaw/workspace-state.json`

**症狀：**
- `workspace/AGENTS.md` 第一行就不同：root 版是完整的 Session 啟動規範，workspace 版是 `"This folder is home. Treat it that way."` 的極簡版本——**完全不同的內容**
- `workspace/x-archive-analysis/` 有 34 個檔案但 `projects/p3nchan/x-archive-analysis/` 只有 11 個——哪個才是 source of truth？遺失了 23 個檔案還是故意精簡？
- Agent 如果不小心讀到 `workspace/AGENTS.md` 而非 root `AGENTS.md`，會得到完全錯誤的指令

**根因分析：** 這兩個目錄是歷史遺留——可能是 OpenClaw 不同版本的 workspace 結構。遷移到目前架構時沒有清理舊的 workspace 目錄。它們各自帶著 `.git/`，代表曾經是獨立的 git repo。

**嚴重度：** 🔴 高 — 過時的系統檔複本是 context pollution 的地雷；資料重複讓 source of truth 模糊

---

### 發現 #4：`projects/` 結構一致性崩壞

**證據（23 個專案審計結果）：**

| 專案 | context.md | status.md | decisions/ | README.md | 問題 |
|------|:---:|:---:|:---:|:---:|------|
| chart-skill | NO | NO | NO | NO | 只有 1 張 PNG + 1 個 research-notes.md，不像專案 |
| reading | NO | NO | NO | NO | 只有 2 個 naval 哲學筆記，不像專案 |
| openclaw | NO | NO | NO | NO | 有 debug-notes.md 和 research/，但沒有標準骨架 |
| x-content | NO | NO | NO | NO | 只有 drafts/ 子目錄，無任何專案骨架 |
| fiscus | YES | YES | NO | YES | 有 README 但沒 decisions/，最大專案之一卻缺 ADR 目錄 |
| nekopen-group | YES | YES | NO | YES | 用 README 而非標準 context.md 做入口 |
| health-analytics | YES | YES | NO | YES | 同上，README + context 雙軌 |

**標準遵循統計：**
- 有 `context.md` + `status.md`：18/23（78%）
- 完全無骨架（無 context 也無 status）：4/23（17%）—— `chart-skill`, `reading`, `openclaw`, `x-content`
- 有 `decisions/`：只有 3/23（13%）—— `da-capital`, `p3nchan`, `pendaily`
- 有 `README.md`（非標準）：5/23（22%）

**症狀：** AGENTS.md 明確規定「每個活躍專案的 `projects/<name>/` 有兩個核心文件：`context.md` + `status.md`」，但實際只有 78% 遵守。`openclaw` 是系統自身的專案，卻連自己的標準都沒遵守。

**根因分析：**
1. 沒有 `projects/` 建立時的 pre-flight validation——agent 可以建一個空目錄就算是「專案」
2. `README.md` 混入是因為某些專案可能先有 README 再有 context.md，但沒統一
3. `decisions/` 目錄的 ADR 規範寫得很好但幾乎沒人用——規範存在但沒有執行力

**嚴重度：** 🔴 高 — boot sequence 掃描 `projects/*/status.md` 找 `in_progress`，沒有 status.md 的專案會被忽略

---

### 發現 #5：`memory/` 命名混亂 + 三重 agent-sociology

**證據：**

**Daily notes 命名不一致：**
- 標準格式 `YYYY-MM-DD.md`：12 個（2026-02-23 到 2026-03-06）
- Sub-agent 格式 `YYYY-MM-DD-<slug>.md`：22 個（全部是 2026-03-06 的）
- 2026-03-06 一天就產生了 **22 個 sub-agent notes + 1 個主 daily note = 23 個檔案**

**agent-sociology 三份拷貝：**
- `/Users/pingu/.openclaw/memory/agent-sociology/` — 18 個 observation（2026-02-04 到 2026-03-06，目前使用中）
- `/Users/pingu/.openclaw/memory/archive/agent-sociology/` — 17 個 observation（2026-02-03 到 2026-02-22，含 README.md）
- `/Users/pingu/.openclaw/workspace/memory/agent-sociology/` — 1 個 observation（2026-02-28，孤兒）

**archive/ 命名不一致：**
- `archive/2026-01` — 月份格式
- `archive/2026-02-early`、`archive/2026-02-mid`、`archive/2026-02-late` — 月份加時段格式
- `archive/2026-03-early` — 同上
- 沒有統一的歸檔粒度：一月整月一個資料夾（4 個檔案），二月拆三段（共 100+ 個檔案）

**其他孤兒：**
- `memory/experiments.md` — 放在 memory/ 根目錄，不像 daily note 也不在 archive
- `memory/main.sqlite` — 5.5MB SQLite 檔案在 memory/ 根目錄，最後修改 2026-02-24，似乎已被棄用
- `memory/state/` — 空目錄

**症狀：** Agent 啟動時讀「今天 + 昨天」的 daily note，但 2026-03-06 一天有 23 個檔案——讀哪些？讀全部浪費 token，不讀可能漏掉重要資訊。archive 的歸檔規則沒有標準化，無法自動化。

**根因分析：**
1. Sub-agent notes 爆炸是因為每個 sub-agent session 都寫獨立檔案，沒有合併機制
2. Archive 分段命名是人工操作，「手感」決定粒度
3. agent-sociology 沒有統一管理，搬遷時留了碎片
4. `main.sqlite` 是早期嘗試的 RAG 索引，停用但沒清理

**嚴重度：** 🔴 高 — boot sequence 讀取 daily notes 的規則無法處理一天 23 個檔案的情況

---

### 發現 #6：Root level 檔案散亂

**證據：**
- `debug_note.md`（root，109 行）vs `projects/openclaw/debug-notes.md`（187 行）— **命名不同（underscore vs hyphen），內容重疊**。AGENTS.md 只引用 `projects/openclaw/debug-notes.md`，root 的 `debug_note.md` 是孤兒
- `tweet.txt`（root，803 bytes）— 一個推文草稿散落在根目錄
- `test-formula.png` + `test-integral.png`（root）— 測試用的圖片遺留
- `pipeline-flowchart.png`（root，76KB）— 應該在某個專案目錄或 media/ 裡
- `healthcheck.log` + `healthcheck_output.txt`（root，都是 0 bytes）— 空檔案遺留
- `openclaw.db`（root，0 bytes）— 空的 SQLite 資料庫
- `openclaw.json.bak` + `.bak.1` ~ `.bak.4` + `.save`（root，共 6 個備份）— 備份沒有清理機制
- `exec-approvals.json`、`update-check.json`、`secrets.json` — 系統運行檔散落在根目錄
- `TODO.md`（root）vs `brain.md`（root）— 功能重疊，TODO.md 只有 1 個項目，brain.md 才是實際使用的待辦系統
- `IDENTITY.md`（root，5 行）vs `identity/`（目錄，含 device-auth.json 和 device.json）— 完全不同的東西用同一個名字
- `HEARTBEAT.md`（root）— 內容只有一行注釋，實際 heartbeat 邏輯在 cron 系統
- `MANIFEST.md` + `SKILL-POLICY.md` — 在啟動序列中未被提及，不清楚何時被讀取

**症狀：** 根目錄有 13 個 `.md` 檔案 + 12 個其他檔案 = 25 個散落的檔案。Agent 啟動時讀 `SOUL.md`→`USER.md`→`MEMORY.md`→`brain.md`→`ORCHESTRATION.md`，但 `MANIFEST.md`、`SKILL-POLICY.md`、`TODO.md`、`IDENTITY.md`、`HEARTBEAT.md` 不在啟動序列中——它們的角色是什麼？

**根因分析：** 根目錄被當成「不知道放哪就放這」的 inbox。沒有「根目錄只放啟動時必讀的入口檔」的規則。

**嚴重度：** 🟡 中 — 不直接造成執行錯誤，但每個多餘的檔案都是 context pollution 風險

---

### 發現 #7：`tools/` vs `skills/` 邊界模糊

**證據：**
- `tools/` — 14 個工具（shell scripts, JS, Python），多數是系統操作腳本（`health-import/`, `claude-max-api-proxy/`, `oura-sync/`, `cloudflare-analytics/` 等）
- `tools/ticktick-cli.sh` + `tools/ticktick-oauth.sh` + `tools/ticktick-oauth.py` — 但 `skills/ticktick/` 也存在
- `tools/claude-status-monitor/` — 有自己的 `.git/` repo，是個獨立專案混在 tools 裡
- `skills/` — 60 個 skill 目錄，全部有 `SKILL.md`（一致性好）
- 但 `tools/` 沒有任何標準化的 metadata 檔案
- `tools/` 裡有些腳本放在根層（`daily-health-report.sh`, `fb-oauth.js`, `meta-token-exchange.sh`, `ticktick-cli.sh`, `ticktick-oauth.py`, `ticktick-oauth.sh`），有些在子目錄裡——不一致

**症狀：** TickTick 既在 `tools/ticktick-*.sh` 又在 `skills/ticktick/`——要用哪個？`tools/` 的定位不清：是「agent 可用的技能」還是「系統基礎設施腳本」？

**根因分析：** `skills/` 是 OpenClaw 的 skill 引擎，有標準化的 `SKILL.md` 入口。`tools/` 則是有機生長出來的腳本集合，沒有定義、沒有分類、沒有入口檔。兩者語義重疊但生命週期不同。

**嚴重度：** 🟡 中 — skills 內部一致性良好（全部有 SKILL.md），tools 是低頻使用的基礎設施，影響較小

---

### 發現 #8：`orchestration/` 目錄和 `ORCHESTRATION.md` 的關係

**證據：**
- `ORCHESTRATION.md`（root，17KB，395 行）— 完整的 Orchestrator Playbook
- `orchestration/templates/task-envelope.md` — Task Envelope 模板
- `orchestration/templates/decision-card.md` — Decision Card 模板
- `orchestration/active_projects.md` — 活躍專案清單
- `orchestration/kb/` — 目錄存在但未檢查內容

**症狀：** `ORCHESTRATION.md` 是主文件，`orchestration/` 是配套資源。這個組織方式其實是合理的（入口檔在根目錄，細節在子目錄），但命名關係沒有在 `ORCHESTRATION.md` 中明確記載。

**根因分析：** 相較於其他問題，這是比較健康的結構。但 `orchestration/active_projects.md` 和 `brain.md` 功能可能重疊。

**嚴重度：** 🟢 低 — 結構尚可，只需明確化

---

### 發現 #9：`config/`、`credentials/`、`cron/` 系統目錄

**證據：**
- `config/` — 只有 1 個檔案（`mcporter.json`）
- `credentials/` — 4 個 pairing/auth JSON
- `cron/` — `jobs.json` + 3 個 `.bak` 檔案（`jobs.json.bak`, `jobs.json.bak.pre-forum-migration`, `jobs.json.bak.pre-analytics-move`）+ `runs/`（21 個 run 記錄）
- `.secrets/` — 43 個項目
- `secrets.json` — 在 root 而非 `.secrets/` 內

**症狀：** `secrets.json` 應該在 `.secrets/` 內而非根目錄。cron 的 3 個 .bak 檔案是歷史遺留，沒有清理。config 目錄只有一個檔案，存在感很低。

**根因分析：** 系統配置分散在 `config/`、`credentials/`、`.secrets/`、根目錄（`openclaw.json`, `secrets.json`）四個地方。

**嚴重度：** 🟡 中 — `secrets.json` 在根目錄是安全風險（雖然 .gitignore 應該擋住了）

---

### 發現 #10：`enterprise-ai-framework` vs `enterprise-ai-guide` 重複專案

**證據：**
- `projects/enterprise-ai-framework/` — 有 `context.md`, `status.md`, `content-drafts.md`
- `projects/enterprise-ai-guide/` — 有 `context.md`, `status.md`
- 兩者名稱極度相似，都與「企業 AI」相關

**症狀：** 不確定是同一專案的重複還是兩個真正不同的專案。Agent 看到相似名稱可能會混淆。

**根因分析：** 可能是專案重新命名時沒有清理舊的。

**嚴重度：** 🟢 低 — 需要確認是否合併

---

### 發現 #11：`writing/` 目錄與專案 writing 資源重疊

**證據：**
- `writing/style-article.md` — 文章寫作風格指南
- `writing/style-post.md` — 社群貼文寫作風格指南
- `writing/pendaily-brand.md` — PenDaily 品牌風格

**症狀：** `pendaily-brand.md` 明顯屬於 `projects/pendaily/`。style guides 是跨專案的參考資料，放 `writing/` 合理，但 `writing/` 只有 3 個檔案——是否值得獨立目錄？還是應該放 `reference/` 或 `skills/` 下？

**根因分析：** 缺乏「跨專案參考資料要放哪」的明確規則。

**嚴重度：** 🟢 低 — 檔案數少，影響小

---

### 發現 #12：`maintenance/` 與 `workflows/` 和 `cron/` 邊界模糊

**證據：**
- `maintenance/` — 7 個 `.md` 檔案：`cleanup-rules.md`, `cleanup-suggestions.md`, `daily-cleanup.md`, `daily-pipeline.md`, `hourly-healthcheck.md`, `website-weekly.md`, `weekly-cleanup.md`
- `workflows/` — 8 個 `.md` 檔案 + `templates/`（3 個模板）+ `output/`
- `cron/` — `jobs.json` 定義排程

**症狀：** `maintenance/daily-cleanup.md` vs `workflows/daily-content-pipeline.md` — 都是「每天要做的事」的定義。哪個才是 agent 該參考的？`maintenance/` 和 `workflows/` 的邊界在哪？

**根因分析：** `maintenance/` 似乎是「系統維護流程」，`workflows/` 是「業務流程」，但沒有在任何地方明確定義這個區分。

**嚴重度：** 🟡 中 — agent 不確定該讀哪個目錄

---

### 發現 #13：`MEMORY.md` 已超過 1KB 限制

**證據：**
- `MEMORY.md` 實際大小：7,184 bytes（7KB），超過 AGENTS.md 規定的「目標 <1KB」近 7 倍
- 內容包含大量工具說明、Discord 頻道結構、安全 skills 描述等

**症狀：** 每次主 session 啟動都要載入 7KB 的 MEMORY.md，違反自己定的規則。隨著內容持續增加，會越來越浪費啟動 token。

**根因分析：** 「聯合重寫 SOP」雖然寫得很好，但實際執行時沒有嚴格控制大小。工具說明和格式規範不應該放在 MEMORY.md——它們更像是 procedural memory / skill reference。

**嚴重度：** 🔴 高 — 直接違反 AGENTS.md 的規範，每個 session 都多花 ~6KB token

---

## B. 痛點追溯表

| 痛點 | 具體證據 | 結構根因 | 建議修復 |
|------|----------|----------|----------|
| **Files drift（檔案漂移）** | `pipeline/20260304-post2-tokenized-gold-final.md` 是 `p3nchan/pipeline/20260304-post2-tokenized-gold.md` 的 final 版但放在根 pipeline/；`debug_note.md`（root）vs `debug-notes.md`（projects/openclaw/）重複存在；`tweet.txt` 散落根目錄 | 沒有「檔案歸位驗證」機制。Agent 完成任務後可以把產出放在任何地方，沒有 post-write validation | 建立 pre-commit hook 或 session-end check：根目錄白名單以外的新檔案 = 告警 |
| **Context fragmentation（上下文碎片化）** | X 內容草稿散落在 `drafts/x-threads/`、`projects/x-content/drafts/`、`projects/p3nchan/pipeline/`、`pipeline/` 四個地方；`workspace/x-archive-analysis/`（34 檔）vs `projects/p3nchan/x-archive-analysis/`（11 檔）資料不同步 | 缺乏「一個專案一個家」的嚴格執行。跨專案內容沒有明確的 routing rule | 內容類型統一到專案目錄；消滅根目錄的 `drafts/`、`pipeline/`、`writing/`，歸入對應專案 |
| **Memory decay（記憶衰退）** | `MEMORY.md` 7KB 超過 1KB 限制；`memory/main.sqlite` 5.5MB 已棄用但沒清理；`memory/state/` 空目錄；sub-agent 一天產生 22 個碎片 note 沒有合併 | 沒有自動化的 memory promotion/compression job 實際運作。SOP 寫了但執行不到位 | 拆分 MEMORY.md 為索引（<1KB）+ topic files；sub-agent notes 自動合併到當日主 note；定期清理已棄用的索引 |
| **Naming inconsistency（命名不一致）** | `reference/` vs `references/`；`debug_note.md` vs `debug-notes.md`（underscore vs hyphen）；archive 分段命名不一致（`2026-01` vs `2026-02-early`）；`skills/mcp-builder/reference/` vs 其他 24 個 skill 的 `references/` | 沒有全域命名規範文件。每次建檔靠 agent 當下判斷 | 建立 `naming-convention.md`：目錄一律用 kebab-case、reference 目錄一律用 `references/`（複數）、archive 用 `YYYY-MM` 格式 |
| **Discovery problem（發現問題）** | 23 個專案中 4 個無骨架（chart-skill, reading, openclaw, x-content）；5 個有 README 而非 context.md；根目錄 25 個檔案混雜；`MANIFEST.md` 和 `SKILL-POLICY.md` 不在啟動序列中 | 沒有 catalog/index 機制；boot sequence 無法自動發現所有專案；根目錄沒有白名單 | 新增 `catalog.json` 自動生成索引；專案強制包含 context.md + status.md；根目錄白名單化 |

---

## C. 風險地圖

### 不修會惡化的問題（按緊急度排序）

1. **🔴 MEMORY.md 膨脹**（現在 7KB，只會越來越大）
   - 路徑：`/Users/pingu/.openclaw/MEMORY.md`
   - 每天新增的工具/偏好/規範都往這塞，沒有 topic file 分流
   - 預估：再過 2 週可能到 10KB+

2. **🔴 Sub-agent notes 爆炸**（2026-03-06 一天 22 個）
   - 路徑：`/Users/pingu/.openclaw/memory/2026-03-06-*.md`
   - 隨著 Autonomous Deep Mode 更頻繁使用，每天的碎片只會更多
   - Boot sequence 讀「今天 + 昨天」無法處理這個量

3. **🔴 內容草稿四散**
   - 路徑：`drafts/`、`pipeline/`、`projects/x-content/drafts/`、`projects/p3nchan/pipeline/`
   - 內容生產越頻繁，散落的檔案越多，遺漏風險越高

4. **🟡 workspace/ 和 workspace-codex/ 殭屍目錄**
   - 路徑：`/Users/pingu/.openclaw/workspace/`、`/Users/pingu/.openclaw/workspace-codex/`
   - 不會惡化但過時的 AGENTS.md 一直是隱患

5. **🟡 projects/ 骨架不一致**
   - 隨著新專案建立，如果沒有 template enforcement，不一致只會加劇

### 已穩定不會惡化的問題（可後處理）

1. **🟢 `reference/` vs `references/`** — 檔案數少且穩定，不急
2. **🟢 `enterprise-ai-framework` vs `enterprise-ai-guide`** — 可能已非活躍專案
3. **🟢 `writing/` 目錄** — 只有 3 個檔案，不會自然增長
4. **🟢 `tools/` 結構** — 使用頻率低，不會自然惡化
5. **🟢 `identity/` vs `IDENTITY.md`** — 語義不同但不衝突
6. **🟢 `memory/main.sqlite`** — 棄用的 5.5MB 檔案，只佔空間
7. **🟢 `TODO.md`** — 和 `brain.md` 重疊但 TODO 幾乎不用

---

## D. 我的處方：針對性修復清單

### P0（本週立即做，一次修好不復發）

**修復 1：拆分 MEMORY.md 為索引 + topic files**
- 對應證據：發現 #13（MEMORY.md 7KB 超標 7 倍）
- 動作：
  - 把 `MEMORY.md` 內的「格式補充」「Model Alias」「安全 Skills」「研究工具」「macOS UI」「Discord Server」「Workflow Runner」「Anthropic Best Practices」移到 `memory/topics/` 下的獨立檔案
  - `MEMORY.md` 只保留：關於 Penchan 的核心事實 + topic file 索引
  - 目標：MEMORY.md < 800 bytes
- 結構性保障：在 AGENTS.md 加入「MEMORY.md 超過 1KB 時自動觸發拆分」規則

**修復 2：建立 sub-agent notes 合併機制**
- 對應證據：發現 #5（一天 22 個碎片 note）
- 動作：
  - 主 session 結束前（或 daily cleanup cron）將 `memory/YYYY-MM-DD-*.md` 的重要內容壓縮合併到 `memory/YYYY-MM-DD.md`
  - 合併後的碎片 note 移到 `memory/archive/` 或直接刪除
  - 在 AGENTS.md 的記憶壓縮 SOP 加入「sub-agent notes 合併」步驟
- 結構性保障：daily cleanup cron 自動執行

**修復 3：統一內容草稿路徑**
- 對應證據：發現 #2（X 內容四散）
- 動作：
  - 刪除根目錄 `drafts/`，搬移內容到 `projects/x-content/drafts/`（或 `projects/p3nchan/drafts/`，依實際歸屬）
  - 刪除根目錄 `pipeline/`，搬移到對應專案
  - 為 `projects/x-content/` 補建 `context.md` + `status.md`
  - 在 AGENTS.md 的「檔案歸位原則」加入：**根目錄不放 `drafts/` 和 `pipeline/`**
- 結構性保障：根目錄白名單

### P1（下週做，防止惡化）

**修復 4：清理殭屍目錄**
- 對應證據：發現 #3（workspace/ 和 workspace-codex/）
- 動作：
  - 確認 `workspace/x-archive-analysis/` 的 34 個檔案是否比 `projects/p3nchan/x-archive-analysis/` 更完整——如果是，合併
  - 將 `workspace-codex/REVIEW-REPORT-2026-02-28.md` 移到 `reports/` 或 `projects/openclaw/`
  - 刪除 `workspace/` 和 `workspace-codex/`（它們的 AGENTS.md 等系統檔已完全過時）

**修復 5：修復 projects/ 骨架一致性**
- 對應證據：發現 #4
- 動作：
  - `chart-skill`：非專案，移到 `skills/chart/` 或 `reference/`
  - `reading`：非專案，移到 `reference/reading/` 或 `memory/topics/reading.md`
  - `openclaw`：補建 `context.md` + `status.md`
  - `x-content`：補建 `context.md` + `status.md`
  - 有 README.md 的專案：確認 README 內容是否已在 context.md 中，是則刪除 README
- 結構性保障：新專案建立 SOP 強制產生骨架

**修復 6：統一命名規範**
- 對應證據：發現 #1、#5
- 動作：
  - `reference/` → 合併到 `references/`（複數統一）
  - `skills/mcp-builder/reference/` → 改名為 `references/`
  - `debug_note.md`（root）→ 刪除（已有 `projects/openclaw/debug-notes.md`）
  - `archive/` 子目錄統一為 `YYYY-MM` 格式

### P2（本月內，改善品質）

**修復 7：根目錄白名單化**
- 對應證據：發現 #6
- 動作：
  - 定義根目錄允許的檔案白名單：`AGENTS.md`, `SOUL.md`, `USER.md`, `MEMORY.md`, `ORCHESTRATION.md`, `brain.md`, `TOOLS.md`, `openclaw.json`, `openclaw.db`
  - 移走：`tweet.txt` → 刪除或移到 `tmp/`；`test-*.png` → 刪除；`pipeline-flowchart.png` → 移到 `projects/` 或 `media/`；`TODO.md` → 合併到 `brain.md`
  - 清理：`healthcheck.log`、`healthcheck_output.txt`（空檔刪除）；`openclaw.json.bak.*` 只保留最新一個
  - `IDENTITY.md`、`HEARTBEAT.md`、`MANIFEST.md`、`SKILL-POLICY.md` → 決定是否納入啟動序列，否則移到 `config/` 或 `profile/`

**修復 8：建立 catalog 索引**
- 對應證據：發現 #4、痛點追溯表 Discovery problem
- 動作：
  - 建立 `catalog.json` 或 `catalog.md`，自動記錄所有專案路徑、狀態、最後更新時間
  - 每日 cron 重建（掃描 `projects/*/status.md` 提取狀態）
  - boot sequence 讀 catalog 而非掃描整個 projects/

---

## E. 給其他兩位的建議

### 給實用派

你最關心「能立刻改善日常體驗」的修復。以下是我找到的「不改會越來越痛」的證據，供你排優先級：

1. **MEMORY.md 膨脹是定時炸彈**：7KB 已經是標準的 7 倍，而且只會增長。每個 session 都在為此多付 token。這是最值得立刻修的。

2. **Sub-agent notes 一天 22 個**：如果 Autonomous Deep Mode 更頻繁使用（ORCHESTRATION.md 鼓勵這麼做），這個問題會指數級惡化。需要合併機制。

3. **內容草稿四散**是日常高頻遇到的摩擦。每次寫 X thread 都要猜「放哪」，寫完又要猜「在哪」。統一到專案目錄是低成本高收益的修復。

4. **`workspace/` 殭屍目錄**裡的過時 `AGENTS.md` 是隱性風險——如果某天 agent 被路徑指到那個版本的 AGENTS.md，會執行完全不同的指令。建議立刻歸檔或刪除。

### 給架構師

你追求「理想架構能持續運作不需人工維護」。以下是我在實際 workspace 中找到的、對你的提案有特別強需求的證據：

1. **Johnny Decimal 的「穩定 ID」在這裡有強需求**：`enterprise-ai-framework` vs `enterprise-ai-guide` 這種差一個詞的專案名，在數字 ID 系統下就不會混淆。`reference/` vs `references/` 的問題也會被固定編號解決。

2. **PARA 的 Archive 概念在這裡執行得不好**：`memory/archive/` 的歸檔粒度不一致（月份 vs 月份分段）、`workspace/` 和 `workspace-codex/` 是該歸檔但沒歸檔的殭屍。如果要用 PARA，Archive 的自動化規則必須是一等公民。

3. **Topic file 拆分有最強需求**：MEMORY.md 的膨脹直接證明了「單一大檔案」的失敗。你提議的 `permanent/people.md`、`permanent/preferences.md` 等 topic file 結構，正好對應到 MEMORY.md 裡應該被拆出去的各個區塊。

4. **Catalog/索引層的需求是真實的**：4 個無骨架專案 + 散落的檔案 = 沒有自動化的 discovery。`catalog.json` 的提案在這裡有直接的落地場景。

5. **但要注意「規範存在但沒執行力」的教訓**：AGENTS.md 已經規定了很好的 ADR 機制，但 23 個專案只有 3 個用了 `decisions/`。MEMORY.md <1KB 的規則也被無視。**任何新架構都必須有自動化的 enforcement，不能靠 agent 自律。**

---

*分析完成。所有路徑均為實際存在的檔案系統路徑，所有數字均為實際計數結果。*
