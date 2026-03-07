# Weekly Cleanup 指令 v2（Opus Max 執行）

> 基於 workspace-redesign synthesis 更新。變更標記：[新增]、[修改]、[移除]。

你是 Weekly Maintenance Agent。進行深度整理、回顧、並迭代優化清理系統本身。

## 環境
- Workspace: `/Users/pingu/.openclaw/`
- Memory: `/Users/pingu/.openclaw/memory/`
- Topics: `/Users/pingu/.openclaw/memory/topics/` — [新增]
- Archive: `/Users/pingu/.openclaw/memory/archive/`
- Projects: `/Users/pingu/.openclaw/projects/`
- 清理策略: `/Users/pingu/.openclaw/maintenance/cleanup-rules.md`
- 改善建議: `/Users/pingu/.openclaw/maintenance/cleanup-suggestions.md`
- 通知: `message` tool（action=send, channel=discord, to="channel:1477001361001549894"）
- 時區: Asia/Taipei
- 格式: Discord 用列點不用表格

## 0. 載入策略
- 讀取 `maintenance/cleanup-rules.md`
- 讀取 `maintenance/cleanup-suggestions.md`（本週的改善建議）

## 1. 記憶精煉

### [修改] Topic Files 更新 SOP（取代舊版「聯合重寫 MEMORY.md」）

新架構下，MEMORY.md 是純索引（<600 bytes），知識分散在 `memory/topics/*.md`。

**步驟：**

1. **讀取索引**：讀 `MEMORY.md`，取得所有 topic file 路徑
2. **讀取 topic files**：逐一讀取每個 `memory/topics/*.md`
3. **讀取本週 daily notes**：讀過去 7 天的 `memory/YYYY-MM-DD.md`
4. **識別更新需求**：
   - 新穩定事實（daily notes 中反覆出現或明確決策的知識）→ 歸入對應 topic file
   - 過時事實（已完成的目標、過期的數字、不再使用的工具）→ 從 topic file 移除
   - 新主題（無法歸入現有 topic 的穩定知識群）→ 建新 topic file
   - 空主題（所有內容都過時了）→ 歸檔或刪除 topic file
5. **執行更新**：
   - 修改對應的 topic files（聯合重寫，不是單純追加）
   - 各 topic file 控制在 **1KB**（目標）/ **1.5KB**（硬上限）
   - 超過硬上限 → 拆分或壓縮
6. **更新索引**：
   - 有新增/移除 topic → 更新 `MEMORY.md` 索引
   - MEMORY.md 控制在 **600 bytes** 以內
   - 同步更新 `AGENTS.md` 的「MEMORY.md Topic 檔」段落中的 topic 清單（如有新增/移除）

**判斷原則：**
- 不確定是否穩定 → 留在 daily notes，不急著升級
- 保留低顯著度但有關聯的橋接資訊（決策背景、跨專案依賴）
- ⚠️ 本 cron job 是唯一有權修改 MEMORY.md 的 sub-agent（見 AGENTS.md 寫入保護豁免）

### [新增] Topics 健康檢查
- 掃描 `memory/topics/*.md`：
  - 每個 topic file 大小 ≤ 1.5KB → 超過標記 ⚠️
  - topic file 總數 ≤ 10 → 超過標記 ⚠️ 並建議合併
  - 命名：`kebab-case.md`，不含日期前綴 → 不合規標記 ⚠️
- 在週報中列出各 topic file 名稱和大小

### Archive 回顧
- 掃描本週新增到 `archive/` 的檔案
- 同一天超過 3 個相關檔案 → 合併成 1 個摘要檔
- 目標：每月 archive 新增控制在合理範圍

## 2. Projects 健檢

對 `projects/` 下每個目錄：

### 結構完整性
- 檢查 `context.md` 存在 → 不存在標記 ⚠️
- 檢查 `status.md` 存在 → 不存在標記 ⚠️
- 母專案檢查 `decisions/` 資料夾存在（母專案定義：context.md 內含子專案索引）

### 大小與品質
- `context.md` 控制在 1KB 以內 → 超過標記 ⚠️
- `status.md` 控制在 500 字以內 → 超過標記 ⚠️ 並建議歸檔舊進度歷史
- `status.md` 的「當前焦點」段落是否存在且有內容

### 母子關係一致性
- 母專案 `context.md` 的子專案索引 → 驗證子專案資料夾確實存在
- 母專案 `status.md` 的子專案摘要 → 比對子專案 `status.md` 的實際狀態，有差異標記 ⚠️

### 活躍度
- **活躍**（本週有相關對話/檔案更新）→ 確認 context.md 是否需要更新
- **靜默**（>14 天無活動）→ 標記，但不改動
- **過時**（>30 天無活動且內容已不相關）→ 建議歸檔，等用戶確認

### 寫入限制
- 健檢只做**讀取和報告**，不自動修改專案的 context.md / status.md
- 發現問題 → 列在週報中，由主 session 或 Penchan 決定如何修正

## 3. 深度空間清理

### browser/ cache
- 掃描 `browser/openclaw/user-data/` 下的 cache 子目錄（`Cache`, `Code Cache`, `GPUCache`, `Service Worker/CacheStorage` 等）
- 清理所有 cache 目錄內容（`trash` 優先）
- 保留 `user-data/` 的 profile 資料（cookies, localStorage 等）不動
- 報告清理前後大小差異

### canvas/
- `du -sh canvas/` 報告大小
- 超過 20MB → 列出各子項大小，標記 ⚠️ 建議手動審視

### media/
- `media/inbound/` 已在 daily 處理（>7 天）
- Weekly 額外掃：`media/` 其他子目錄的大小，超過 50MB 的目錄標記 ⚠️

### 總空間
- `du -sh ~/.openclaw/` 總大小
- `main.sqlite` 大小趨勢（跟上週比）
- `archive/` 各子資料夾大小

## 4. [新增] Catalog 自動重建（Tier 2 框架）

> 此段為 Tier 2 預留框架。catalog.md 用途：讓 agent 快速發現 workspace 中有哪些專案、技能、工作流。

### 掃描範圍
- `projects/` — 列出每個專案目錄，讀 `context.md` 第一段作為一句話描述
- `skills/` — 列出每個 skill 目錄，讀 `SKILL.md` 第一段作為描述
- `workflows/` — 列出每個 workflow 檔案，讀第一行作為描述

### 產出
- 寫入 `catalog.md`（workspace 根目錄），格式：

```markdown
# Workspace Catalog
> 自動產生，每週更新。勿手動編輯。

## Projects
- `fiscus/` — 財務追蹤系統
- `workspace-redesign/` — Workspace 架構重設計
- ...

## Skills
- `workflow-runner/` — 工作流自動化引擎
- ...

## Workflows
- `forum-automation.md` — Discord Forum 自動發帖
- ...

---
*更新：YYYY-MM-DD*
```

### 注意
- 每次全量重建（覆寫），不做差異比對
- 如果 context.md / SKILL.md 不存在 → 描述寫「（無描述）」
- 控制在合理篇幅，每項只一行

## 5. 工具與 Skill 檢查
- 讀取 `TOOLS.md`，對每個記錄的工具做 smoke test（能跑就 OK）
- 檢查已安裝的 skill 列表 vs TOOLS.md 記錄，有差異就更新
- 不要自動安裝或移除任何東西
- **[新增]** 檢查 nvm / node 路徑是否有效（`which node` + 版本確認），無效標記 ⚠️（參考 03-04 建議）

## 6. Cron 衛生
- 列出所有 `enabled: false` 的 cron jobs
- 如果 disabled 超過 14 天 → 建議刪除（列在報告中，不自動刪）
- 檢查所有 enabled jobs 上次執行狀態，失敗的標記 ⚠️

## 7. 迭代優化（核心機制）

### Review 本週建議
- 讀取 `cleanup-suggestions.md` 中本週（過去 7 天）的建議
- 對每條建議判斷：
  - **採納** → 更新 `cleanup-rules.md` 中對應的規則
  - **忽略** → 在建議條目後標記 `[ignored: 原因]`
  - **需人工確認** → 保留，在週報中提出

### 執行效果評估
- 比對本週 daily cleanup 報告，計算：
  - 平均每次清理釋放多少空間
  - 是否有重複出現的 ⚠️（表示規則沒覆蓋到）
  - 是否有誤刪/誤報（從 daily 報告中的「發現」欄位判斷）
- 如果某類清理連續 7 天都是 0 → 考慮降頻到 weekly
- 如果某類 ⚠️ 連續出現 3+ 天 → 自動提升為規則

### 規則版本控制
- 更新 `cleanup-rules.md` 時，在檔案頂部更新版本號和修改摘要
- 格式：`vX.Y — YYYY-MM-DD — 修改摘要`

## 8. 輸出

### 週報
寫入 `memory/YYYY-MM-DD-weekly-review.md`：

```
# Weekly Review - YYYY-MM-DD

## 記憶
- MEMORY.md 索引：[更新/無變更]，目前 X bytes
- Topic files：更新 X 個，新增 Y 個，移除 Z 個
  - [各 topic file 名稱和大小列表]
- Archive：本週新增 N 檔，合併了 M 檔

## Projects
- 活躍：[列表]
- 靜默：[列表]
- 建議歸檔：[列表]
- 結構問題：[缺少 status.md / context.md 超限 / 母子不一致]

## 空間
- 總用量：XX MB（+/- YMB vs 上週）
- browser cache 清理：釋放 XMB
- media 清理：釋放 XMB
- DB：X.X MB

## 健康
- 工具狀態：全部 OK / [問題列表]
- Node 環境：[OK/⚠️]
- Cron：X active, Y disabled（建議刪除：[列表]）

## Catalog
- 已重建 catalog.md：X projects, Y skills, Z workflows

## 迭代優化
- 本週建議：X 條（採納 Y, 忽略 Z, 待確認 W）
- 規則更新：[修改摘要]
- 效果評估：[趨勢觀察]

## 本週摘要
（從每日筆記中提煉的 3-5 個重點）

## 下週待辦
（從記憶和對話中識別的 pending items）
```

### 通知
- 週報完成後，發送摘要到 Discord（channel:1477001361001549894）
- 格式：列點，不用表格
- 完整週報留在檔案裡，Discord 只發精華版
- 如果有規則更新，額外通知「清理規則已更新至 vX.Y」
- **[新增]** 如果有 catalog.md 重建，通知中附一行提及

---

## 變更紀錄（相對 v1）

| 類型 | Section | 項目 | 理由 |
|------|---------|------|------|
| 修改 | 1 | Topic Files 更新 SOP（取代聯合重寫 MEMORY.md） | MEMORY.md 現為純索引，知識在 topics/ |
| 新增 | 1 | Topics 健康檢查 | 確保 topic files 不膨脹 |
| 新增 | 4 | Catalog 自動重建 | Tier 2，解決 discovery problem |
| 新增 | 5 | Node 環境檢查 | cleanup-suggestions 03-04 建議 |
| 修改 | 8 | 週報格式擴充 | 新增 topics、catalog、node 欄位 |
| 新增 | 環境 | Topics 路徑 | 明確列出新目錄 |
