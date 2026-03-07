# Cleanup Cron 優化 — 差距分析與建議報告

> 日期：2026-03-07
> 分析者：Workspace Redesign Agent
> 依據：synthesis.md（4-agent 辯論合成結論）、現有 maintenance/*.md、AGENTS.md

---

## 1. 架構變更摘要

workspace-redesign 確認的核心改動（影響 cleanup）：

| # | 改動 | Tier | 影響 cleanup 的面向 |
|---|------|------|---------------------|
| 1 | MEMORY.md → 純索引 + `memory/topics/*.md` | Tier 1 | weekly 記憶精煉 SOP 需大改 |
| 2 | memory/ 命名規範 + 清理規則 | Tier 1 | daily 記憶整理邏輯需調整 |
| 3 | catalog.md 自動重建（weekly cron） | Tier 2 | weekly 新增任務 |
| 4 | Session-end 3-line state freeze | Tier 2 | daily 可加驗證 |
| 5 | CONVENTIONS.md（命名規範政策檔） | Tier 2 | daily 可引用做命名合規檢查 |

---

## 2. 差距分析

### 2.1 Daily Cleanup 差距

#### 🔴 需修改：記憶整理段落（Section 1）

**現況**：Daily cleanup 會主動處理 `memory/` 根目錄的 `YYYY-MM-DD-*.md` topic 檔——歸檔、壓縮、追加到日記。

**問題**：新架構下：
- `memory/topics/*.md` 是**長期穩定知識主題檔**，daily 不應該碰
- `YYYY-MM-DD-*.md` 格式的散檔在根目錄應已減少（命名規範限制一天一個主 daily note）
- 舊邏輯「提取結論追加到 YYYY-MM-DD.md」在新架構下仍合理，但範圍需縮小

**建議**：
- 明確排除 `memory/topics/` 目錄，daily 只處理 `memory/` 根目錄散檔
- `YYYY-MM-DD-*.md`（帶後綴的散檔）仍然做歸檔/壓縮，但不動 topics/
- 新增檢查：根目錄是否有不符命名規範的檔案

#### 🔴 需修改：`-mtime` → `-newermt`

**現況**：cleanup-rules.md 和 daily 指令隱含使用 `find -mtime`。

**問題**：cleanup-suggestions 有兩條實際案例（03-02、03-04）證實 `-mtime` 不可靠，`-newermt` 更準確。

**建議**：統一規範用 `-newermt` 或 reference file 方法。

#### 🟡 建議新增：`openclaw.db` 空檔檢查

**現況**：03-06 建議提到根目錄 `openclaw.db` 為 0B 孤立空檔。

**建議**：daily 健康檢查加一條：偵測根目錄 0-byte 的 `.db` 或 `.sqlite` 檔案。

#### 🟡 建議新增：Session-end state freeze 驗證

**現況**：不存在。

**建議**：daily 掃描當天的 `projects/*/status.md`，檢查最近修改過的 status.md 是否包含「當前焦點」段落（state freeze 的最小驗證）。這是 Tier 2 改動的輕量驗證，不阻塞。

#### 🟢 維持不變

- media/inbound/ 7 天閾值：合理，保持
- cron/runs/ 14 天 log 保留：合理，保持
- 暫存檔清理規則：合理，保持
- 報告格式：微調即可

---

### 2.2 Weekly Cleanup 差距

#### 🔴 需大改：記憶精煉 SOP（Section 1）

**現況**：weekly 的「聯合重寫」是直接重寫 MEMORY.md（作為全文知識庫）。

**問題**：MEMORY.md 現在是純索引（<600 bytes），知識分散在 topic files。

**建議**：SOP 改為：
1. 讀 MEMORY.md 索引 → 讀各 topic file → 讀本週 daily notes
2. 更新對應的 topic files（新增/修改/移除過時內容）
3. 如有新主題 → 建新 topic file → 更新 MEMORY.md 索引
4. 如有主題過時 → 歸檔或刪除 topic file → 更新 MEMORY.md 索引
5. 各 topic file 控制在 1KB（AGENTS.md 寫 <1.5KB，建議實際目標 1KB、硬上限 1.5KB）
6. MEMORY.md 索引控制在 600 bytes

#### 🟡 建議新增：catalog.md 自動重建

**現況**：不存在。

**建議**：Tier 2，先預留框架。Weekly cron 掃描：
- `projects/` — 列出各專案 + 讀 context.md 第一行作描述
- `skills/` — 列出各 skill + 讀 SKILL.md 第一行
- `workflows/` — 列出各 workflow + 第一行
- 產出 `catalog.md` 到 workspace 根目錄

**風險**：Failure Analyst 反對，但 weekly 自動重建成本極低，且不依賴 agent 紀律。

#### 🟡 建議新增：topics/ 健康檢查

**現況**：不存在。

**建議**：weekly 掃描 `memory/topics/*.md`：
- 每個 topic file 大小 ≤ 1.5KB（硬上限）
- 超過的標記 ⚠️，在週報中建議壓縮
- topic file 數量上限：暫定 10 個

#### 🟡 建議調整：cleanup-suggestions review

**現況**：5 條未處理建議。

**建議處理方式**：

| # | 建議 | 處置 | 理由 |
|---|------|------|------|
| 1 | `-mtime` 改 `-newermt`（03-02） | ✅ 採納 | 有實際案例佐證 |
| 2 | `-newermt` 二次佐證（03-04） | ✅ 採納 | 同上，合併處理 |
| 3 | nvm node 路徑斷裂（03-04） | ⚠️ 需人工確認 | 非 cleanup 範疇，建議排入 weekly 健康檢查 |
| 4 | agent-sociology/ 重複目錄（03-06） | ✅ 採納 | daily 加入偵測 + 週報建議手動處理 |
| 5 | openclaw.db 0B 空檔（03-06） | ✅ 採納 | daily 健康檢查新增 |

#### 🟢 維持不變

- Projects 健檢邏輯：完善，保持
- 空間清理邏輯：完善，保持
- 工具與 Skill 檢查：保持
- Cron 衛生：保持

---

### 2.3 Cleanup Rules 差距

#### 🔴 需更新

- 版本號：v1.0 → v2.0
- 新增 `memory/topics/` 規則區塊
- `-newermt` 統一規範
- 新增孤立空檔偵測規則

#### 🟡 建議調整

- `memory/` 根目錄 `.md` 檔案數上限：15 → 維持 15（topics/ 是子目錄不計入，合理）
- 新增 `memory/topics/` 區塊：
  - 每個 topic file ≤ 1KB（目標）/ 1.5KB（硬上限）
  - topic file 總數上限：10 個
  - 命名：`kebab-case.md`，不含日期前綴

---

## 3. 向後相容性分析

| 項目 | 相容性 | 說明 |
|------|--------|------|
| media/inbound/ 清理 | ✅ 完全相容 | 只改時間判斷方式 |
| cron/runs/ 清理 | ✅ 完全相容 | 無改動 |
| 暫存檔清理 | ✅ 完全相容 | 無改動 |
| 記憶整理 | ⚠️ 行為改變 | 不再觸碰 topics/，縮小處理範圍 |
| MEMORY.md 精煉 | ⚠️ 行為改變 | 從全文重寫改為索引+topics 分散更新 |
| 健康檢查 | ✅ 向後相容 | 只新增檢查項，不移除 |
| catalog.md | ✅ 純新增 | 不影響現有功能 |

所有改動都是**收窄範圍或新增檢查**，不會破壞現有能跑的清理功能。

---

## 4. 總結

- **必須做**：記憶整理 SOP 適配新架構、`-newermt` 統一、cleanup-suggestions 處理
- **建議做**：catalog.md 框架、topics/ 健康檢查、state freeze 驗證
- **不做**：不改動 projects 健檢、空間清理、cron 衛生等已穩定的模組

預估影響：daily cleanup 指令修改幅度約 30%，weekly cleanup 約 40%，cleanup-rules 約 25%。
