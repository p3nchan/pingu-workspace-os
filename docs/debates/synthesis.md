# Workspace Redesign — Final Synthesis

> Synthesized by: Pingu (Main Session)
> Date: 2026-03-07
> Inputs: System Architect + Migration Strategist + Failure Analyst + Pragmatist (4x Opus agents)
> Research base: ChatGPT Deep Research + Perplexity (GPT 5.4) + Academic report

---

## Meta-Insight: The Core Tension

**Failure Analyst 的核心洞察是全場最重要的一句話：**

> "The system already cannot follow its own simplest, most clearly stated rule (MEMORY.md <1KB, currently 7KB). Adding more rules will not fix this. It will make it worse."

這是正確的診斷。但處方不完全對。

**Architect + Migrator 的反論同樣有效：**

MEMORY.md 膨脹不是因為「沒遵守規則」，而是因為**結構上沒有其他地方放那些知識**。知識只有兩個去處：MEMORY.md（太大）或 daily notes（會被歸檔消失）。建立 topic files 不是加規則，是**讓正確行為成為阻力最小的路徑**。

**最終結論：問題不是架構 vs 紀律的二選一。是「改善架構讓紀律自然成立」。**

---

## Decision Matrix

### ✅ Tier 1: Do Now（4/4 agents 同意）

| # | 改動 | 理由 | 預估耗時 |
|---|------|------|----------|
| 1 | **MEMORY.md → 純索引 + topic files** | 全員同意。7KB → <500 bytes index。主題拆到 `memory/topics/`。啟動 token 減少 80%+。 | 1-2 hr |
| 2 | **Boot sequence 明確化** | 已 80% 完成。把 AGENTS.md 的自然語言描述轉成嚴格清單。 | 30 min |
| 3 | **memory/ 目錄清理 + 命名規範** | 297 個檔案在 <2 週內產生。sub-agent log 歸到 projects/ 或 logs/，一天一個主 daily note。 | 2-3 hr |

### 🔶 Tier 2: Do Soon（3/4 同意，可控風險）

| # | 改動 | 理由 | 預估耗時 |
|---|------|------|----------|
| 4 | **輕量 catalog.md**（markdown，不是 JSON） | 解決 discovery problem。Architect 想用 JSON 但 Pragmatist 和 Migrator 都偏好 markdown（人類可讀）。Failure Analyst 反對，但 weekly cron 自動重建的成本極低。 | 1-2 hr |
| 5 | **Session-end state freeze（輕量版）** | 不是 JSON 編譯引擎。就是 session 結束前寫 3 行到 status.md：做了什麼、沒做完什麼、下一步。 | 30 min（加到 AGENTS.md） |
| 6 | **CONVENTIONS.md** | 寫下已經在遵守的命名規範。零遷移成本，純政策文件。 | 30 min |

### ⏳ Tier 3: Defer（等數據說話）

| # | 改動 | 觸發條件 |
|---|------|----------|
| 7 | YAML frontmatter | 如果 catalog.md 手動維護太痛苦 → 用 frontmatter 自動化 |
| 8 | SQLite / 向量索引 | 檔案 >2000 或 grep 明顯變慢時 |
| 9 | 記憶分層自動 decay | promotion workflow 穩定運行 3 個月後 |

### ❌ Tier 4: Kill（多數 agents 反對）

| # | 改動 | 反對理由 |
|---|------|----------|
| 10 | Johnny Decimal 目錄重編號 | 遷移成本致命，23 個專案 + 所有硬編碼路徑。語意名稱（`fiscus`）比數字（`11.02`）對 LLM 更有意義。 |
| 11 | Zettelkasten 原子筆記 | 知識規模太小，開銷不成比例。 |
| 12 | Cryptographic hash drift prevention | 單人系統不需要。`git diff` 就夠了。 |
| 13 | Bi-temporal memory tracking | 39 個 daily notes 沒有時間模糊危機。 |
| 14 | status.md → JSON | 殺死人類可讀性，Discord iOS 更難看。 |
| 15 | Session init validation script | 用清單取代腳本。 |

---

## Failure Analyst 的 5 個隱藏假設（值得記住）

1. **「Agent 像資料庫運作」** — 錯。Agent 是 LLM，讀文字推理，不是做 key-value lookup。
2. **「檔案發現是主要瓶頸」** — 部分對。真正的瓶頸是「知道要去找」而不是「找得到」。
3. **「更多結構 = 更可靠」** — 錯。結構創造耦合，耦合創造脆弱性。
4. **「系統會大幅擴展」** — 過早。先為 2x 設計，在 2x 時重新評估。
5. **「Agent 會遵守新規則」** — 最危險的假設。每加一行規則，現有規則的注意力就被稀釋。

---

## Architect 的理想架構 — 我們保留什麼

Architect 提出的完整目錄樹（`00-system/`, `10-projects/`, `30-memory/`, etc.）整體過激，但以下設計值得保留為**長期北極星**：

1. **四層記憶模型**（Ephemeral → Working → Episodic → Semantic）— 概念正確，用現有結構漸進實現
2. **catalog.json 的 schema 設計** — 先用 markdown，但如果未來需要自動化，這個 schema 是好起點
3. **成功指標表** — 量化目標讓我們知道改動是否有效
4. **Sub-agent boot（只給 task envelope）** — 已在執行，繼續

---

## 實施建議

### Phase 1: 本週末（2-3 小時）
1. MEMORY.md 拆分
2. Boot sequence 明確清單
3. memory/ 命名規範 + 清理規則

### Phase 2: 下週（累計 2 小時）
4. catalog.md 建立 + cron
5. Session-end 3-line freeze 加入 AGENTS.md
6. CONVENTIONS.md 撰寫

### Phase 3: 一個月後 Review
- 檢查 MEMORY.md 是否維持 <1KB
- 檢查 catalog.md 是否被使用
- 檢查 memory/ 目錄增長速度
- 決定是否需要 Tier 3 改動

---

## 代表性語錄

**Failure Analyst**: "The status quo is a B-. The proposals aim for an A+ but risk landing at a C."

**Pragmatist**: "做最小的三個改動來解決最大的三個痛點，讓系統繼續跑，讓數據告訴我們下一步該做什麼。"

**Architect**: "The agent's context window is RAM; the filesystem is disk. Every architectural choice must minimize what loads into RAM while maximizing what can be paged in on demand."

**Migrator**: "Every phase has a clean rollback path. This is the most important property of the migration plan."

---

*Synthesis complete. Ready for Penchan approval.*
