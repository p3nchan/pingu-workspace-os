# 長期架構師分析：Workspace Redesign 藍圖

> **角色**: 長期架構師 (Long-term Systems Architect)
> **日期**: 2026-03-07
> **核心信念**: 結構債的複利比金融債更兇猛。現在不修地基，一年後付十倍代價。
> **立場**: 我承認 Failure Analyst 說的每一個風險都是真的。但他的結論是「所以不要動」，我的結論是「所以要用對的方式動」。不動的代價他沒算。

---

## A. 規模推演：1 年後的樣貌

### 現況快照（2026-03-07）

| 指標 | 當前數值 | 資料來源 |
|------|---------|---------|
| 專案數量 | 23 個 (`projects/` 目錄) | `ls projects/` |
| 活躍 daily notes | ~39 個（含 23 個 2026-03-06 的 session 記錄）| `ls memory/` |
| 已歸檔 daily notes | ~210 個（`archive/` 下各子目錄加總）| `2026-01` + `2026-02-*` + `2026-03-early` |
| `memory/` 總大小 | 8.3 MB（含 5.7 MB SQLite） | `du -sh memory/` |
| Skills 數量 | 61 個 | `ls skills/` |
| Workflows 數量 | ~10 個 | `ls workflows/` |
| MEMORY.md 大小 | 7,184 bytes（目標 <1KB，超標 7x） | 實測 |
| AGENTS.md 大小 | 16,437 bytes / ~350 行 | 實測 |
| Workspace 總大小 | 978 MB | `du -sh .openclaw/` |
| 根目錄項目數 | 70+ 個檔案/目錄 | `ls -la` |
| `decisions/` 有內容的專案 | 0 個（3 個有目錄但只有 `.gitkeep`） | 實測 |
| `reference/` vs `references/` | 兩個目錄並存，各放不同東西 | 實測 |

### 增長率估算

基於 2026-02-22 到 2026-03-07（13 天）的觀測：

- **專案**：13 天內建了 23 個專案。即使速度放緩到每週 1-2 個新專案，1 年後 = **75-130 個專案**
- **Daily notes**：平均每日 10-15 個 session 記錄（2026-03-06 一天就 23 個檔案）。1 年 = **3,650-5,475 個 daily note 檔案**
- **Memory 目錄大小**：目前 8.3 MB（13 天）。線性推算 1 年 = **233 MB**，但 agent-sociology 這類研究目錄會加速膨脹，實際可能 **300-500 MB**
- **Decisions**：目前 0 筆實際 ADR。這本身就是問題——重大決策散落在 `status.md` 和 daily notes 裡沒有被正式沉澱
- **Skills**：61 個，其中 `latex/` 系列佔 215 MB（含二進制）。1 年後預估 **80-100 個 skills**

### 現有結構下，哪些東西會先崩潰？

**第 1 個崩潰點：`memory/` 目錄（3 個月內）**

2026-03-06 一天產生 23 個 session 記錄檔案，總計 ~200 KB。如果維持這個產出速度，3 個月後 `memory/` 會有 **~2,000 個檔案**在活躍層（歸檔前）。即使用 `YYYY-MM-DD-slug.md` 前綴可以排序，但一個扁平目錄裡塞 2,000 個檔案時：

- `ls memory/` 變成噪音牆，無法掃描
- Agent 的 boot sequence 讀「今天 + 昨天」的 daily notes 需要先列目錄找出哪些是今天的
- 歸檔 cron 的分類規則是用日期切半月（`2026-02-early`, `2026-02-late`），命名不穩定（`early` vs `mid` vs `late`），且歸檔後的目錄結構沒有標準化

**第 2 個崩潰點：根目錄混亂（已經崩潰）**

70+ 個頂層項目，包括：

- 系統設定檔（`AGENTS.md`, `SOUL.md`, `MEMORY.md`, `ORCHESTRATION.md` 等 12 個 `.md`）
- 運行時產物（`openclaw.db`, `openclaw.json`, 5 個 `.bak` 檔案, `healthcheck.log`, `healthcheck_output.txt`）
- 功能目錄（`projects/`, `skills/`, `workflows/`, `memory/`）
- 重複/模糊目錄（`reference/` vs `references/`, `workspace/` vs `workspace-codex/`）
- 臨時產物（`test-formula.png`, `test-integral.png`, `tweet.txt`, `debug_note.md` vs `projects/openclaw/debug-notes.md`）
- 平台整合目錄（`discord/`, `telegram/`, `browser/`, `canvas/`）

這已經不是「整潔度」的問題，是 **Agent 定位效率**的問題。Agent 讀根目錄需要消化 70 個條目才能理解 workspace 結構，每一個不相關的條目都是 attention 的噪音。

**第 3 個崩潰點：MEMORY.md 過載（已經崩潰）**

MEMORY.md 已經 7,184 bytes，是自訂目標 1KB 的 7 倍。這不只是「超標了」的問題：

- 每個 session 啟動都載入整個 MEMORY.md（7KB = ~1,750 tokens），隨著資訊繼續堆疊，這個數字只會繼續長大
- boot sequence 載入的固定成本：`SOUL.md`(2.7KB) + `AGENTS.md`(16.4KB) + `MEMORY.md`(7.2KB) + `brain.md`(1.3KB) + 今天/昨天 daily notes(~15KB) = **~42KB** = **~10,500 tokens**，占 200K context window 的 5.25%。看似不多，但 1 年後如果 MEMORY.md 漲到 20KB，boot 成本就超過 15,000 tokens
- 更關鍵的是，MEMORY.md 目前混雜了：用戶偏好、模型 alias、安全 skills 紀錄、研究工具清單、Discord server 結構、Debug 筆記索引、Workflow runner 說明、Anthropic best practices、工作原則。至少 5 種不同的知識類型塞在一個檔案裡

**第 4 個崩潰點：決策流失（已經在發生）**

- 3 個專案有 `decisions/` 目錄，但全部只有 `.gitkeep`——0 筆 ADR
- 決策實際記在 `status.md` 的「進度歷史」和「關鍵決策記錄」中
- 但 `status.md` 有 500 字上限（AGENTS.md line 100），舊決策會被清除
- Daily notes 是原始記錄，但已經在歸檔，且歸檔後的目錄結構混亂（`2026-02-early`, `2026-02-mid`, `2026-02-late`）
- 結果：**重要決策在系統裡沒有穩定的永久住所**。3 個月前的決策，既不在 status.md（被清掉了），也不在 decisions/（沒人寫），只在某個已歸檔的 daily note 裡——如果你記得是哪一天的話

---

## B. 理想終態架構設計

### 設計原則

1. **Context Window 是 RAM，Filesystem 是 Disk**。架構的核心目標是最小化 boot 載入量，最大化按需讀取的效率。
2. **Markdown 是 Source of Truth，其他一切都是衍生視圖**。任何索引、目錄、資料庫都必須可從 markdown 層重建。
3. **可預測性 > 表達力**。死板但穩定的路徑 > 靈活但需要搜尋的路徑。

### 完整目錄樹

```
.openclaw/
|
|-- BOOT.md                              # 唯一入口。啟動時第一個讀的檔案
|                                         # 內容：載入清單 + 載入順序 + 版本
|
|-- system/                               # 系統身份與規則
|   |-- SOUL.md                           # Agent 身份（幾乎不變）
|   |-- AGENTS.md                         # 操作規則
|   |-- USER.md                           # 人類 profile
|   |-- ORCHESTRATION.md                  # 多 agent 框架
|   |-- SKILL-POLICY.md                   # Skill 治理
|   |-- TOOLS.md                          # 工具設定
|   |-- templates/                        # 可重複使用的模板
|   |   |-- task-envelope.md
|   |   |-- decision-card.md
|   |   |-- project-init.md               # 新專案的 context.md + status.md 模板
|   |   `-- adr.md                        # ADR 模板
|   `-- cron/                             # Cron job 定義（從根目錄移入）
|       |-- healthcheck.md
|       |-- weekly-cleanup.md
|       `-- content-pipeline.md
|
|-- projects/                             # 所有專案（活躍 + 休眠）
|   |-- fiscus/
|   |   |-- context.md                    # 背景、策略、子專案索引
|   |   |-- status.md                     # 執行進度、Phase、blockers
|   |   |-- decisions/                    # Append-only ADR
|   |   |   `-- 0001-independent-from-easyfund.md
|   |   |-- research/                     # 專案內研究產物
|   |   `-- archive/                      # 完成的 Phase 產物
|   |-- penchan-co/
|   |   |-- context.md
|   |   |-- status.md
|   |   |-- decisions/
|   |   |-- drafts/
|   |   `-- references/
|   |-- p3nchan/
|   |-- health-analytics/
|   |-- da-capital/
|   |-- thesis/
|   |-- workspace-redesign/
|   |-- openclaw/
|   |   |-- context.md
|   |   |-- status.md
|   |   `-- debug-notes.md               # 系統除錯知識庫（KB）
|   |-- _dormant/                         # 休眠專案（>30 天無更新）
|   |   |-- chart-skill/
|   |   |-- pingu-avatar/
|   |   `-- monitoring-system/
|   `-- _archived/                        # 正式歸檔（明確結束的專案）
|       `-- white-cat-planet/
|
|-- memory/                               # 記憶系統
|   |-- MEMORY.md                         # 索引檔：指向 topics/，自身 <1KB
|   |-- topics/                           # 語意記憶（permanent facts，原子化）
|   |   |-- user-profile.md               # Penchan 的個人資料、偏好
|   |   |-- model-aliases.md              # 模型別名對照表
|   |   |-- format-conventions.md         # 格式約定（Telegram、X、Token 等）
|   |   |-- tools-and-skills.md           # 工具與 skill 索引
|   |   |-- discord-structure.md          # Discord server 結構
|   |   |-- work-principles.md            # 工作原則與最佳實踐
|   |   `-- bazi-profile.md               # 八字命盤（不常變）
|   |-- daily/                            # 每日筆記（情景記憶）
|   |   |-- 2026/
|   |   |   |-- 03/
|   |   |   |   |-- 2026-03-07.md         # 當日主筆記
|   |   |   |   |-- 2026-03-07--fiscus-research.md
|   |   |   |   `-- 2026-03-07--website-seo.md
|   |   |   `-- 02/
|   |   |       |-- 2026-02-28.md
|   |   |       `-- ...
|   |   `-- _index.md                     # 月份索引（自動生成，每月重建）
|   |-- experiments.md                    # 實驗記錄（跨專案）
|   `-- agent-sociology/                  # 特殊研究系列（保留）
|       `-- ...
|
|-- skills/                               # Skill 定義（保持現有結構）
|   |-- crypto-research/
|   |-- security-audit/
|   |-- ...
|   `-- _index.md                         # Skill 索引（名稱 → 路徑 → 用途）
|
|-- workflows/                            # 跨專案通用流程（保持現有結構）
|   |-- README.md                         # 工作流索引
|   |-- content-pipeline.md
|   |-- session-retro.md
|   `-- templates/
|
|-- media/                                # 媒體檔案（保持現有結構）
|   |-- inbound/
|   `-- outbound/
|
|-- runtime/                              # 運行時產物（不需版控）
|   |-- openclaw.db
|   |-- openclaw.json
|   |-- openclaw.json.bak*
|   |-- healthcheck.log
|   |-- healthcheck_output.txt
|   |-- secrets.json
|   |-- exec-approvals.json
|   |-- delivery-queue/
|   |-- devices/
|   `-- logs/
|
|-- integrations/                         # 平台整合設定
|   |-- discord/
|   |-- telegram/
|   `-- browser/
|
`-- brain.md                              # 全局即時視圖（保持根目錄，每次 boot 讀）
```

### 每個設計決策的理由

#### 入口檔與職責邊界

| 入口檔 | 職責 | 載入時機 | 大小上限 |
|--------|------|---------|---------|
| `BOOT.md` | 載入清單與順序的唯一權威。不含任何規則，只列「讀什麼、什麼順序、什麼條件」 | 每次 session 第一個讀 | 500 bytes |
| `brain.md` | 全局即時狀態：進行中的事、等確認的事、待辦、排程、異常 | 每次 session 第二個讀 | 2KB |
| `system/AGENTS.md` | 操作規則與行為準則 | 每次 session 讀 | 保持現有 |
| `system/SOUL.md` | 不可變身份核心 | 每次 session 讀 | 保持現有 |
| `system/USER.md` | 人類 profile | 每次 session 讀 | 保持現有 |
| `memory/MEMORY.md` | 長期記憶**索引**，指向 `topics/*.md` | 主 session 讀 | <1KB（這次真的做到） |
| `system/ORCHESTRATION.md` | 多 agent 框架 | 有 orchestration 任務時才讀 | 保持現有 |

**關鍵改變**：`BOOT.md` 取代了 AGENTS.md 裡散落的啟動規則。啟動序列目前分布在 `AGENTS.md` lines 3-9 + `ORCHESTRATION.md` Section 3 Cold Start，沒有一個明確的 canonical source。`BOOT.md` 解決這個問題。

#### 記憶系統分層

| 層級 | 目錄 | 保留期限 | 寫入者 | 範例 |
|------|------|---------|--------|------|
| **Ephemeral（瞬時）** | Context window 內 | 單一 session | Agent | 當前對話、中間推理 |
| **Episodic（情景）** | `memory/daily/YYYY/MM/*.md` | 永久歸檔 | Agent + Cron | 每日筆記、session 記錄 |
| **Semantic（語意）** | `memory/topics/*.md` | 永久，定期 review | 主 session + weekly cleanup cron | 用戶偏好、工具清單、格式約定 |
| **Procedural（程序）** | `system/AGENTS.md`, `workflows/*.md`, `skills/*/SKILL.md` | 永久 | Penchan 授權的主 session | 操作規則、工作流程、技能定義 |

**為什麼 `memory/topics/` 用原子化檔案而非一個大 MEMORY.md**：

目前 MEMORY.md 有 9 個 section 混在一起。當 Agent 需要查「模型 alias」時，它必須載入整個 7KB 的檔案然後定位到特定 section。改成原子化後：

- `memory/topics/model-aliases.md`（~500 bytes）只在需要時讀取
- Boot 時只讀 `memory/MEMORY.md` 索引（<1KB），知道 topics 的路徑
- 修改某個 topic 不需要重寫整個 MEMORY.md，減少「聯合重寫」時漏改的風險

Failure Analyst 會說「2 次讀取 vs 1 次」。我的回答：MEMORY.md 現在 7KB，1 年後可能 20KB。讀 20KB 只為了找 500 bytes 的 model alias 是浪費。原子化讀取的固定成本是 1KB（索引）+ 500 bytes（目標 topic）= 1.5KB，永遠不會隨 topic 數量增長。

#### 專案生命週期管理

```
projects/
|-- fiscus/              # 活躍（最近 30 天有更新）
|-- penchan-co/          # 活躍
|-- _dormant/            # 休眠（>30 天無更新，但未結束）
|   `-- chart-skill/
`-- _archived/           # 歸檔（專案已正式結束）
    `-- white-cat-planet/
```

**三態分類標準**：

| 狀態 | 條件 | 可見性 | 動作 |
|------|------|--------|------|
| Active | 最近 30 天 `status.md` 有更新 | `projects/` 根層 | 正常讀取 |
| Dormant | >30 天無更新、但未明確結束 | `projects/_dormant/` | `brain.md` 不列出，但路徑穩定可讀 |
| Archived | Penchan 明確標記「結束」或「放棄」 | `projects/_archived/` | 只讀，不修改 |

**為什麼用 `_` 前綴**：確保 `_dormant/` 和 `_archived/` 在字母排序中出現在最後，不干擾活躍專案的視覺掃描。Agent 做 `ls projects/` 時，活躍專案排前面。

#### 命名規範

這裡不是「要有規範」，而是具體的規則：

**目錄命名**：
```
projects/<slug>/              # kebab-case，全小寫，無空格
skills/<slug>/                # kebab-case，全小寫
memory/daily/YYYY/MM/         # 數字，2 位零填充
```

**Daily note 檔案命名**：
```
YYYY-MM-DD.md                 # 當日主筆記（唯一）
YYYY-MM-DD--<slug>.md         # Session 記錄（雙破折號分隔日期與 slug）
```

為什麼用雙破折號 `--`：目前用的是單破折號 `YYYY-MM-DD-slug.md`，但這讓 `2026-03-06-cron-review.md` 的結構模糊——`cron-review` 裡的破折號和日期的破折號混在一起。用 `2026-03-06--cron-review.md` 可以機器解析出 `date = 2026-03-06`, `slug = cron-review`。

**ADR 命名**：
```
decisions/NNNN-<slug>.md      # 4 位零填充序號 + kebab-case
```
例如：`decisions/0001-independent-from-easyfund.md`

**Topic 記憶命名**：
```
memory/topics/<topic-slug>.md  # kebab-case，名稱即內容描述
```
例如：`model-aliases.md`, `discord-structure.md`, `user-profile.md`

#### 索引/Catalog 機制

我**不**建議建一個 `catalog.json` 然後靠 cron 每天重建。Failure Analyst 說得對——stale catalog 比 no catalog 更危險。

我建議的替代方案是**分散式微索引**：

| 索引檔 | 位置 | 覆蓋範圍 | 維護者 |
|--------|------|---------|--------|
| `brain.md` | 根目錄 | 活躍任務、待辦、排程 | 每次 session 更新 |
| `memory/MEMORY.md` | `memory/` | topics 清單 + 路徑 | weekly cleanup cron |
| `skills/_index.md` | `skills/` | Skill 名稱 → 路徑 → 用途 | 新增/刪除 skill 時更新 |
| `workflows/README.md` | `workflows/` | 工作流清單 | 已存在，保持 |
| 各專案 `context.md` | `projects/<name>/` | 子專案索引 | 已存在，保持 |

每個索引只負責自己的 scope，不嘗試建一個「全局 catalog」。這樣單一索引壞了不會影響其他 scope。

#### Boot Sequence

`BOOT.md` 的具體內容：

```markdown
# BOOT.md — 啟動載入清單
# 版本：1.0 | 最後更新：YYYY-MM-DD

## 必載（每次 session）
1. brain.md                              → 全局狀態
2. system/SOUL.md                        → 身份
3. system/USER.md                        → 用戶 profile
4. system/AGENTS.md                      → 操作規則

## 條件載入
5. memory/MEMORY.md                      → 長期記憶索引（主 session only）
6. memory/daily/YYYY/MM/YYYY-MM-DD.md    → 今天 + 昨天 daily notes
7. system/ORCHESTRATION.md               → 如 brain.md 有進行中的 orchestration 任務

## 按需載入（不在 boot 時讀）
- projects/<name>/context.md             → 提到專案時才讀
- projects/<name>/status.md              → 需要看進度時才讀
- memory/topics/<topic>.md               → 需要特定記憶時才讀
- skills/<name>/SKILL.md                 → 使用 skill 時才讀
```

**為什麼 BOOT.md 要獨立存在**：

目前啟動規則分布在 `AGENTS.md` lines 3-9 和 `ORCHESTRATION.md` Section 3。當 Agent 啟動時，它必須先讀完 16KB 的 AGENTS.md 才能知道要讀什麼。這是循環依賴——你需要讀 AGENTS.md 來知道怎麼啟動，但 AGENTS.md 本身就是啟動要讀的東西之一。`BOOT.md` 打破這個循環：極小（<500 bytes）、無規則只有清單、第一個讀。

---

## C. 遷移路徑

### 核心約束：不能停機

系統每天都在用。Penchan 每天有 session。Cron 每小時跑。Sub-agent 隨時可能被 spawn。所以每一個 phase 都必須保持向後相容——舊路徑能用、新路徑也能用——直到確認全面切換。

### Phase 0：止血（第 1-2 天）

**目標**：修復已經崩潰的東西，不新增任何架構

| 動作 | 具體操作 | 自動化程度 |
|------|---------|-----------|
| MEMORY.md 拆分 | 把 MEMORY.md 的 9 個 section 拆成 `memory/topics/*.md`（7 個檔案）。MEMORY.md 改為純索引，只列每個 topic 的路徑和一句話描述 | **手動**（需要判斷分類邊界） |
| 根目錄清理 | 把運行時產物移入 `runtime/`（`openclaw.db`, `*.bak`, `healthcheck.*`, `exec-approvals.json`, `secrets.json`） | **可自動化**（`mkdir runtime && mv ...`） |
| 合併重複 | `reference/` + `references/` 合併成 `references/`，內容歸位到對應專案 | **手動**（需判斷歸屬） |
| 刪除臨時產物 | `test-formula.png`, `test-integral.png`, `tweet.txt`, 多個 `openclaw.json.bak.*` | **可自動化** |
| 建立 BOOT.md | 把 AGENTS.md 的啟動序列抽出寫成 BOOT.md，AGENTS.md 改為引用 BOOT.md | **手動** |

**預計耗時**：2-4 小時（一個 sub-agent session）
**風險等級**：低。`runtime/` 移動對功能無影響（OpenClaw runtime 用絕對路徑或相對 `./` 路徑）。MEMORY.md 拆分後舊路徑仍有效（索引檔還在原位）。

### Phase 1：記憶重組（第 3-5 天）

**目標**：建立穩定的 daily note 歸檔結構

| 動作 | 具體操作 | 自動化程度 |
|------|---------|-----------|
| Daily notes 目錄結構 | 在 `memory/daily/` 下建 `YYYY/MM/` 結構。新的 daily notes 開始寫入新結構 | **可自動化** |
| 歸檔遷移 | 把 `memory/archive/` 的內容搬到 `memory/daily/YYYY/MM/` 結構下 | **可自動化**（腳本，按日期解析） |
| 命名規範切換 | 新 session 記錄改用 `YYYY-MM-DD--slug.md`（雙破折號） | **手動**（修改 Agent 的寫入行為） |
| AGENTS.md 更新 | 更新 boot sequence 引用路徑 | **手動** |

**預計耗時**：2-3 小時
**風險等級**：低-中。需要更新 `system/weekly-cleanup` cron 的歸檔路徑。但 cron 是讀 markdown 流程定義的，改定義即可。

### Phase 2：專案生命週期（第 6-10 天）

**目標**：分類現有 23 個專案，建立休眠/歸檔機制

| 動作 | 具體操作 | 自動化程度 |
|------|---------|-----------|
| 專案分類 | 掃描所有 `projects/*/status.md` 的最後更新時間，標記 active / dormant / archived | **可自動化** |
| 建立 `_dormant/` 和 `_archived/` | 把非活躍專案移入對應目錄 | **手動**（需 Penchan 確認分類） |
| ADR 補建 | 從 `status.md` 歷史記錄中提取重大決策，補建 ADR 到 `decisions/` | **半自動**（Agent 提取，Penchan 確認） |
| Skills 索引 | 建立 `skills/_index.md`，列出所有 skill 的名稱、路徑、觸發條件 | **可自動化** |

**預計耗時**：3-5 小時（分 2-3 個 session）
**風險等級**：中。移動專案目錄會影響 hardcoded 路徑。解法：每個被移動的專案在原位留一個 `_MOVED.md`，內容是 `This project has been moved to projects/_dormant/<name>/`。Agent 讀到 `_MOVED.md` 就知道去哪找。6 個月後刪除 `_MOVED.md`。

### Phase 3：系統檔案重組（第 11-14 天）

**目標**：把系統檔案從根目錄移入 `system/`，平台整合移入 `integrations/`

| 動作 | 具體操作 | 自動化程度 |
|------|---------|-----------|
| 建立 `system/` 目錄 | 移入 `SOUL.md`, `AGENTS.md`, `USER.md`, `ORCHESTRATION.md`, `SKILL-POLICY.md`, `TOOLS.md` | **手動**（需同步更新所有引用） |
| 建立 `integrations/` | 移入 `discord/`, `telegram/`, `browser/` | **手動** |
| 全域路徑更新 | grep 所有檔案，更新路徑引用 | **半自動**（grep 找，手動改） |
| BOOT.md 更新 | 更新所有路徑 | **手動** |

**預計耗時**：4-6 小時
**風險等級**：**高**。這是影響最廣的 phase。每個引用 `AGENTS.md` 路徑的地方都要改。建議在一個低活動窗口（週末早上）執行，且 Penchan 在線可以快速 hotfix。

**降低風險的策略**：
1. 在 Phase 3 之前，先跑一個 grep 掃描，列出所有 hardcoded 路徑引用，建成 checklist
2. 用 symlink 過渡：根目錄留 symlink 指向 `system/`，確保舊路徑 6 個月內仍然有效
3. Cron 和 sub-agent prompt 中的路徑用 symlink 期間逐步更新

### Phase 4：穩定化（第 15-30 天）

**目標**：驗證新結構在日常運作中穩定

| 動作 | 具體操作 |
|------|---------|
| 每日驗證 | 每天 boot 時確認所有 BOOT.md 列的檔案都存在且可讀 |
| Cron 穩定性 | 觀察 2 週的 cron 執行記錄，確認無路徑錯誤 |
| Sub-agent 相容性 | 觀察 sub-agent spawn 的成功率 |
| 清理 symlink | 確認所有引用已更新後，移除根目錄的 symlink |
| 回顧 BOOT.md | 確認 boot 載入量沒有回升 |

**預計耗時**：持續 2 週的觀察，不需要集中工時

### 遷移總結

| Phase | 內容 | 耗時 | 風險 | 可自動化 |
|-------|------|------|------|---------|
| 0 | 止血：MEMORY.md 拆分 + 根目錄清理 | 2-4h | 低 | 60% |
| 1 | 記憶重組：daily note 結構化 | 2-3h | 低-中 | 70% |
| 2 | 專案生命週期分類 | 3-5h | 中 | 50% |
| 3 | 系統檔案重組 | 4-6h | 高 | 30% |
| 4 | 穩定化觀察 | 2 週 | 低 | N/A |

**總計**：~11-18 小時的主動工作 + 2 週觀察期。可在 1 個月內完成。

---

## D. 從研究中借鑑的 5 個具體機制

### 機制 1：State Freezing and Injection（狀態凍結與注入）

**原理**：來自學術研究報告。Session 結束時，不是保留對話歷史，而是將當前決策狀態、目標、專案地圖編譯成密集的結構化區塊。下次 session 起始時，只注入這個編譯後的狀態區塊，完全丟棄對話殘渣。核心洞察：transformer 架構優先處理最近的 token，如果最近的 token 是推理噪音（失敗嘗試、錯誤修正、反覆討論），Agent 的準確度就會崩潰。

**如何適配我們的系統**：

這個機制其實已經部分存在——`status.md` 和 `brain.md` 就是某種形式的 state snapshot。但目前的問題是：

1. `status.md` 混合了 state（進行中什麼）和 history（過去做了什麼），讀取時 history 會污染 attention
2. boot sequence 會讀今天 + 昨天的 daily notes（~15KB），但 daily notes 是原始對話記錄，充滿推理噪音

**具體做法**：每次 session 結束時，更新 `status.md` 的「當前焦點」section（前 3 行），用**已編譯的結論**而非「今天做了什麼」的流水帳。Boot 時只讀「當前焦點」，不讀「進度歷史」（除非主動需要回溯）。

**預期效果**：boot 載入的有效信號密度提升，Agent 的 session 間一致性改善。成本：每個 session 結束多花 30 秒更新 status.md。

### 機制 2：分散式微索引取代全域目錄（Distributed Micro-Indexes）

**原理**：LangGraph 的 namespace 隔離 + Claude Code 的按需載入。不建一個大索引，而是讓每個 scope 維護自己的小索引。每個索引的失效只影響該 scope，不會級聯。

**如何適配我們的系統**：

- `brain.md` = 任務索引（已存在）
- `memory/MEMORY.md` = 記憶索引（改造為純指標）
- `skills/_index.md` = 技能索引（新建）
- `workflows/README.md` = 工作流索引（已存在）
- 各 `context.md` = 專案內索引（已存在）

**預期效果**：Agent 不需要為了發現一個 skill 而掃描 61 個目錄。讀 `skills/_index.md`（~2KB）就知道所有 skill 的位置和觸發條件。估算：skill 發現從 O(n) 目錄掃描降到 O(1) 索引查詢，省 ~500 tokens/次。

### 機制 3：Memory Promotion Pipeline（記憶升格管線）

**原理**：MemoryBank 論文提出依時間與重要性做遺忘/強化。遞迴摘要研究證明提煉比保留更有效。核心不是「保留更多」而是「分層、提煉、淘汰」。

**如何適配我們的系統**：

目前的 weekly cleanup cron 有權更新 MEMORY.md，但沒有明確的 promotion 標準。我建議的具體流程：

```
Daily note 寫入
  ↓ (即時)
如果包含「已確認」「決定」「最終」等關鍵字 → 標記為 promotion candidate
  ↓ (weekly cleanup cron)
Cron 讀取 7 天內的 promotion candidates
  → 穩定事實 → 寫入 memory/topics/ 的對應檔案
  → 專案決策 → 寫入 projects/<name>/decisions/
  → 偏好變更 → 更新 memory/topics/user-profile.md
  → 不穩定/暫時 → 留在 daily notes，不升格
  ↓ (monthly)
review memory/topics/ 所有檔案，移除過時內容
```

**預期效果**：決策不再流失。重要事實有穩定住所。MEMORY.md 索引保持精簡。

### 機制 4：Two-Double-Dash 機器可解析命名（Parseable Naming Convention）

**原理**：AI-native 檔案管理文獻強調命名要最佳化語意密度（semantic density），讓 agent 能從檔名推斷內容、範圍和關聯，而不需要打開檔案。

**如何適配我們的系統**：

目前的 daily note 命名 `2026-03-06-cron-review.md` 有解析歧義——slug 裡的 `-` 和日期的 `-` 混在一起。改為 `2026-03-06--cron-review.md` 後：

```python
# 機器解析
date, slug = filename.rsplit('--', 1)
# date = "2026-03-06", slug = "cron-review"
```

簡單、無歧義、向後相容（沒有 `--` 的就是舊格式或主筆記）。

**預期效果**：Agent 可以不讀取檔案內容就知道這是哪天的什麼 session。搜尋 `memory/daily/2026/03/2026-03-06--*` 就能列出 2026-03-06 的所有 session。比目前的命名更適合 glob 操作。

### 機制 5：Dormant/Archive 自動降級（Automated Project Lifecycle）

**原理**：PARA 框架的 Archive 概念 + Johnny Decimal 的穩定位置。重點不是刪除，而是降低可見性——不活躍的東西不應該佔用 Agent 的注意力。

**如何適配我們的系統**：

```
Weekly cleanup cron 掃描 projects/*/status.md 的 modified time
  → >30 天無更新 + status 非 "in_progress" → 自動移入 _dormant/
  → Penchan 明確說「這個專案結束了」→ 手動移入 _archived/
  → _dormant/ 的專案被重新更新 → 自動移回 projects/ 活躍層
```

**預期效果**：`ls projects/` 只列出 10-15 個活躍專案而非 23 個（含死掉的）。Agent boot 時掃描活躍專案的成本降低 40%。Penchan 的心理負擔也降低——看到 23 個專案會焦慮，看到 10 個活躍的會舒服得多。

---

## E. 我與實用派的分歧

### 我預測實用派會說「不要改」的東西

Failure Analyst 的分析（我讀了）核心論點是：「系統已經不能遵守現有規則，加更多規則只會更糟。」他的具體否決清單：

1. Kill catalog.json
2. Kill YAML frontmatter
3. Kill Johnny Decimal
4. Kill cryptographic hashes
5. Kill bi-temporal tracking

**我同意 Kill 的**：YAML frontmatter（強制成本太高，收益不明確）、cryptographic hashes（過度工程）、bi-temporal tracking（過度工程）。

**我不同意 Kill 的**：

#### 分歧 1：MEMORY.md 拆分

Failure Analyst 說「2 次讀取 vs 1 次」。但這是**靜態分析的謬誤**。

- 現況：1 次讀取 7KB（且持續增長）= 1,750 tokens，每個 session 付一次
- 拆分後：1 次讀取索引 800 bytes + 0-2 次按需讀取 topic = 200 + 0-500 tokens
- **1 年後**：現況 = 1 次讀取 ~20KB = 5,000 tokens/session。拆分後 = 仍然 200 + 0-500 tokens/session

| 時間點 | 不拆分的 boot 成本 | 拆分後的 boot 成本 | 差異 |
|--------|-------------------|-------------------|------|
| 現在 | 1,750 tokens | 700 tokens | 省 60% |
| 6 個月後 | ~3,500 tokens | 700 tokens | 省 80% |
| 1 年後 | ~5,000 tokens | 700 tokens | 省 86% |

**結構債計算**：現在拆分成本 = 2 小時。6 個月後拆分成本 = 4 小時（因為 topics 間的邊界更模糊、引用更多）。1 年後拆分成本 = 8 小時（因為所有 Agent prompt、cron、sub-agent 都在引用巨大的 MEMORY.md 的特定 section）。

#### 分歧 2：Daily Notes 結構化目錄

Failure Analyst 沒明確反對這個，但 2,000+ 個檔案在扁平目錄裡是**物理限制**問題，不是架構偏好問題。

- 目前：`memory/2026-03-06.md`, `memory/2026-03-06-slug.md` 全部在 `memory/` 根層
- 2026-03-06 一天就 23 個檔案。1 個月 = ~500 個檔案。`ls memory/` 的輸出就是 500 行
- 歸檔目前靠 cron 移到 `archive/`，但歸檔命名不穩定（`early`, `mid`, `late`）

現在不改的成本：每次歸檔 cron 都要決定「early or mid or late？」，這個邏輯不穩定。
現在改的成本：2 小時寫遷移腳本。

#### 分歧 3：專案生命週期管理

Failure Analyst 會說「23 個專案，不需要分類」。但：

- 23 個專案裡，至少 `chart-skill/`, `pingu-avatar/`, `monitoring-system/`, `enterprise-ai-guide/`, `reading/` 已經是 dormant（最後更新 >7 天前）
- 1 年後有 75-130 個專案。`ls projects/` 列出 130 個目錄，其中 90% 是死掉的——這是 Agent 每次掃描專案時的噪音

現在改的成本：1 小時分類 + `mkdir _dormant _archived`。
1 年後改的成本：分類 130 個專案，更新上百個引用路徑。保守估計 **20 小時**。

### 結構債總帳

| 改動 | 現在的成本 | 1 年後的成本 | 倍率 |
|------|-----------|-------------|------|
| MEMORY.md 拆分 | 2 小時 | 8 小時 | 4x |
| Daily notes 結構化 | 2 小時 | 15 小時（遷移 5,000 個檔案） | 7.5x |
| 專案生命週期 | 1 小時 | 20 小時 | 20x |
| BOOT.md 啟動清單 | 1 小時 | 3 小時 | 3x |
| 根目錄清理 | 1 小時 | 5 小時（更多垃圾堆積） | 5x |
| **合計** | **7 小時** | **51 小時** | **7.3x** |

「結構債的複利比金融債更兇猛」不是比喻，是精算。

---

## F. 總結：我的藍圖

### 願景

建一個會呼吸的 workspace——它的結構能隨專案增長而自然擴張，而不是在某個臨界點突然崩潰。記憶有清楚的分層（瞬時 → 情景 → 語意 → 程序），每一層有獨立的生命週期和維護機制。專案有生老病死（活躍 → 休眠 → 歸檔），死掉的不會佔據活著的空間。Agent 的每一次啟動都是確定性的：讀 BOOT.md → 讀固定清單 → 準備好了。不多讀一個位元組，不少讀一個事實。

這不需要 Johnny Decimal 的全面編號（我同意那個遷移成本太高），不需要 YAML frontmatter 的強制合規（Agent 做不到），不需要 cryptographic hash 的完美同步（過度工程）。它需要的是：**把正確的東西放在可預測的地方，讓不需要的東西自動退出視線，讓重要的知識永遠不會流失**。

### 核心 Principle 三條

1. **最小 Boot 原則**：Agent 啟動時載入的 token 數必須是常數，不隨 workspace 大小增長。BOOT.md + brain.md + AGENTS.md + SOUL.md + USER.md + MEMORY.md（索引） = ~25KB = ~6,250 tokens，固定值。
2. **知識有家原則**：每一條穩定事實都必須有一個且只有一個永久住所（memory/topics/ 的某個檔案）。Daily notes 只是暫存區，不是最終目的地。
3. **自動降級原則**：任何超過 30 天沒有更新的東西（專案、daily notes 的活躍區、臨時檔案），都必須自動退出 Agent 的預設可見範圍。不是刪除，是降低優先級。

---

*長期架構師 | 2026-03-07*
*「如果你覺得架構重構很貴，等你不重構試試看。」*
