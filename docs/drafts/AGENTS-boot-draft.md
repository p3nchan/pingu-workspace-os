# AGENTS.md Boot Sequence — 修改建議草稿

> 作者：Pingu (workspace-redesign sub-agent)
> 日期：2026-03-07
> 依據：synthesis.md + 01-pragmatist + 02-architect + 03-failure-analyst 共識
> 狀態：草稿，待 Penchan 審閱

---

## 修改摘要

| # | 修改段落 | 類型 | 理由 |
|---|---------|------|------|
| 1 | **Session 啟動清單** (lines 3-21) | 修改 | 轉成嚴格 numbered checklist（Tier 1 #2）；加入失敗處理；明確 daily note 只讀主筆記 |
| 2 | **Session 結束** | 新增 | Session-end state freeze（Tier 2 #5）——離開前寫 3 行到 status.md 或 brain.md |
| 3 | **記憶系統 > 目錄結構** (lines 52-58) | 修改 | 反映已完成的 topic-based 結構；更新路徑描述 |
| 4 | **記憶系統 > 規則** (lines 62-65) | 修改 | 補入 topic file 觸發條件清晰化 |
| 5 | **記憶壓縮 SOP** (lines 67-72) | 修改 | 改為 rewrite topic files（非 rewrite MEMORY.md）；加入 sub-agent notes 合併步驟 |
| 6 | **寫入保護 > 豁免** (line 78) | 修改 | weekly cleanup 的範圍從「更新 MEMORY.md」改為「更新 MEMORY.md 索引 + topic files」 |

---

## 完整修改後的段落

### 段落 1：Session 啟動清單（修改）

> 替換範圍：AGENTS.md lines 3-21（從 `## Session 啟動清單` 到 `### MEMORY.md Topic 檔` 結尾）

```markdown
## Session 啟動清單

### Step 0：自動載入（Project Context，不需工具呼叫）
- AGENTS.md, SOUL.md, USER.md, IDENTITY.md, MEMORY.md（純索引，<600 bytes）

### Step 1-4：Boot 工具呼叫（嚴格按順序）

| # | 動作 | 失敗處理 |
|---|------|----------|
| 1 | `read brain.md` — 全局待辦與進行中任務 | 必須成功。讀不到 = 回報異常，不繼續 |
| 2 | `read memory/YYYY-MM-DD.md`（今天的主筆記）| 不存在就跳過（今天還沒建） |
| 3 | `read memory/YYYY-MM-DD.md`（昨天的主筆記）| 不存在就跳過 |
| 4 | 如果 brain.md 有 orchestration 任務 → `read ORCHESTRATION.md` | 條件不符就跳過 |

**注意**：Step 2-3 只讀 `YYYY-MM-DD.md` 主筆記，不讀 `YYYY-MM-DD-*.md` 的 sub-agent session notes。

### Thread / Topic 開場
- 有 `thread_label` / `topic_name` → 用主題 contextualize，不泛泛打招呼
- 無主題 → 根據 brain.md 的當前焦點開場

### Topic 檔按需載入（非 boot，對話中觸發）

| 觸發條件 | 讀取檔案 | 限制 |
|----------|----------|------|
| 個人話題、八字、Penchan 偏好 | `memory/topics/penchan.md` | 群聊不讀 |
| 研究工具、安全 skills、macOS 自動化 | `memory/topics/tools.md` | — |
| Discord server 結構、頻道操作 | `memory/topics/discord.md` | — |
| 工作方法、Anthropic best practices | `memory/topics/practices.md` | — |
| 特定專案 | `projects/<name>/context.md` → `status.md` | Lazy loading |

**原則**：寧可多一次 read 也不要 boot 時全灌。Context window 是 RAM，不是倉庫。
```

**修改理由**：
1. Synthesis Tier 1 #2：「把自然語言描述轉成嚴格清單」——改為 numbered table 格式，每步有失敗處理
2. Failure Analyst 發現 #5：一天 22 個 sub-agent notes，boot 讀不完——明確只讀主筆記
3. Architect 的 Context Window = RAM 原則——boot 載入量必須是常數
4. Topic 檔觸發條件從模糊的 bullet list 改為 table，觸發條件更明確
5. 加入「無主題時用 brain.md 焦點開場」的 fallback，避免空泛開場

---

### 段落 2：Session 結束（新增）

> 插入位置：在「Session 啟動清單」段落之後、「價值優先級」段落之前（即 line 22 之後）

```markdown
## Session 結束

### State Freeze（每次 session 離開前必做）

在結束 session 前，寫 3 行 state freeze 到**最相關的 status.md**（如果沒有明確專案，寫到 `brain.md`）：

```
## State Freeze YYYY-MM-DD HH:MM
1. Done: [本次完成的關鍵成果，1-2 句]
2. Incomplete: [未完成的事項 / blockers，沒有就寫 None]
3. Next: [下一步動作，給下一個 session 的自己看]
```

### 規則
- **格式固定**：Done / Incomplete / Next 三行，不多不少
- **寫在哪**：有明確專案 → 該專案 `status.md` 的「當前焦點」section 上方；無明確專案 → `brain.md`
- **覆蓋舊的**：同一個 status.md 只保留最新一次 state freeze，舊的刪掉（不是 append-only）
- **不寫的情況**：純閒聊 session、<3 分鐘的快速問答
- **目的**：下次 boot 讀 brain.md 或 status.md 時，第一眼就知道上次停在哪
```

**設計理由**：
1. Synthesis Tier 2 #5：「session 結束前寫 3 行到 status.md：做了什麼、沒做完什麼、下一步」
2. Architect 的 State Freezing and Injection 機制——用編譯後的結論取代原始對話噪音
3. 「覆蓋舊的」而非 append：防止 status.md 膨脹，始終保持最新狀態快照
4. 排除純閒聊：避免無意義的 state freeze 製造噪音
5. Pragmatist 精神：不建驗證腳本、不產 JSON、就寫 3 行文字——成本趨近零

---

### 段落 3：記憶系統 > 目錄結構（修改）

> 替換範圍：AGENTS.md lines 52-58（`### 目錄結構` 整段）

```markdown
### 目錄結構
- `MEMORY.md` — **純索引**（<600 bytes），指向 topic files + 格式速查
- `memory/YYYY-MM-DD.md` — 每日主筆記（一天只有一個）
- `memory/YYYY-MM-DD-<slug>.md` — Sub-agent session notes（boot 不讀，按需查閱）
- `memory/topics/*.md` — 穩定知識主題檔（按需載入，見啟動清單的觸發條件表）
  - `penchan.md` — 個人、八字（群聊不讀）
  - `tools.md` — 研究工具、安全 skills、macOS 自動化
  - `discord.md` — Server 結構、Forum
  - `practices.md` — 工作原則（補充 AGENTS.md）
- `memory/archive/` — 歸檔的舊筆記（按 `YYYY-MM/` 分目錄）
- `memory/experiments.md` — 實驗記錄
- `logs/sessions/` — session transcripts 歸檔（不放 memory/）
- `projects/<name>/context.md` — 專案記憶（按需載入，控制 1KB 以內）
```

**修改理由**：
1. MEMORY.md 已完成拆分為純索引（目前 17 行，<600 bytes），描述需要對齊現狀
2. 明確區分「每日主筆記」和「sub-agent session notes」的不同角色
3. 列出所有 topic files 的名稱和用途，讓 agent 不用讀 MEMORY.md 就知道有哪些 topic
4. 歸檔目錄統一為 `YYYY-MM/` 格式（Failure Analyst 發現 #5 的 archive 命名不一致問題）

---

### 段落 4：記憶系統 > 規則（修改）

> 替換範圍：AGENTS.md lines 62-65（`### 規則` 整段）

```markdown
### 規則
- 想記住的東西 → 寫檔案，不要「心裡記」
- 新穩定知識 → 加到對應 topic file（`memory/topics/*.md`），不塞 MEMORY.md
- MEMORY.md 只更新索引條目（新增/刪除 topic file 時）
- Session transcripts → `logs/sessions/`，不留在 memory/
- 一天只有一個主 daily note（`YYYY-MM-DD.md`），sub-agent notes 用 `YYYY-MM-DD-<slug>.md`
- Topic file 超過 1.5KB → 拆分為更細的 topic，更新 MEMORY.md 索引
```

**修改理由**：
1. 加入「MEMORY.md 只更新索引條目」的明確規則，防止知識又被塞回索引
2. 加入 topic file 大小上限和拆分規則（Architect 的原子化原則）
3. 明確 sub-agent notes 的命名格式

---

### 段落 5：記憶壓縮 SOP（修改）

> 替換範圍：AGENTS.md lines 67-72（`### 記憶壓縮 SOP` 整段）

```markdown
### 記憶壓縮 SOP
1. 讀 MEMORY.md 索引 → 確認 topic files 清單
2. 讀最近 7 天的 daily notes（主筆記）+ 需要處理的新資訊
3. **更新 topic files**：
   - 穩定事實 → 寫入對應的 `memory/topics/*.md`
   - 過時事實 → 從 topic file 移除
   - 新主題（現有 topic 都不適合）→ 建新 topic file + 更新 MEMORY.md 索引
4. **合併 sub-agent notes**：
   - 讀 `memory/YYYY-MM-DD-*.md`（最近 7 天的 sub-agent notes）
   - 重要結論合併到當天主筆記 `YYYY-MM-DD.md`
   - 合併後的碎片 notes → 移到 `memory/archive/YYYY-MM/` 或刪除
5. 驗證：MEMORY.md 索引 <600 bytes、各 topic file <1.5KB
- 不確定是否穩定 → 留在 daily notes，不急著升級到 topic file
```

**修改理由**：
1. 壓縮目標從「rewrite MEMORY.md」改為「rewrite topic files」——對齊新的 topic-based 結構
2. 新增 Step 4「合併 sub-agent notes」——Failure Analyst 發現一天 22 個碎片 note 的問題
3. 新增 Step 3 的「新主題建立」流程——讓 topic file 能自然增長
4. 驗證標準更新：MEMORY.md <600 bytes（索引更小），topic file <1.5KB

---

### 段落 6：寫入保護 > 豁免（修改）

> 替換範圍：AGENTS.md line 78（`- **豁免**：...` 那一行）

```markdown
- **豁免**：`system/weekly-cleanup` cron 有權更新 `MEMORY.md` 索引 + `memory/topics/*.md`（記憶精煉是其核心任務）
```

**修改理由**：
- Weekly cleanup 的工作範圍已從「重寫 MEMORY.md」擴展為「更新索引 + 重寫 topic files」
- 如果不更新豁免範圍，cron 在技術上無權修改 topic files，會導致記憶壓縮 SOP 無法被 cron 執行

---

## 刪除的段落

無。所有修改都是「替換」或「新增」，沒有需要完全移除的段落。

---

## 附錄：修改前後對照速查

### Boot Sequence 變化

| 項目 | 修改前 | 修改後 |
|------|--------|--------|
| 格式 | 自然語言 bullet list | Numbered table + 失敗處理欄 |
| Daily note 讀取 | 未區分主筆記和 sub-agent notes | 明確只讀 `YYYY-MM-DD.md` 主筆記 |
| Topic 觸發條件 | 4 個 bullet 簡列 | Table 格式，含觸發條件和限制欄 |
| 無主題 fallback | 無 | 用 brain.md 焦點開場 |
| Session 結束 | 無 | 3 行 state freeze（Done / Incomplete / Next） |

### 記憶系統變化

| 項目 | 修改前 | 修改後 |
|------|--------|--------|
| MEMORY.md 角色 | 純索引（已實施） | 純索引，<600 bytes（加入到規則） |
| Topic files | 提及但未列舉 | 完整列出 4 個 topic + 用途 |
| 壓縮目標 | 聯合重寫 MEMORY.md | 更新 topic files + 合併 sub-agent notes |
| Sub-agent notes | 未提及處理方式 | 壓縮合併到主筆記，碎片歸檔或刪除 |
| Weekly cleanup 豁免 | 只有 MEMORY.md | MEMORY.md 索引 + topic files |
| Archive 命名 | 未規範 | 統一 `YYYY-MM/` 格式 |

---

## 實施建議

1. **先審閱此草稿**——確認每個修改段落的方向正確
2. **一次性替換**——6 個段落可以在同一個 session 內全部替換，不需要分階段
3. **替換後驗證**——下一次 session boot 時觀察是否正確遵循新的 checklist
4. **State freeze 試跑**——前 3 天刻意執行 session-end freeze，觀察自然融入程度

預估修改耗時：15-20 分鐘（純文字替換，無檔案搬遷）。

---

*草稿完成。等待 Penchan 審閱。*
