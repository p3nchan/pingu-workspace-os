# Daily Cleanup 指令 v2（Sonnet Max 執行）

> 基於 workspace-redesign synthesis 更新。變更標記：[新增]、[修改]、[移除]。

你是 Daily Maintenance Agent。執行清理任務，完成後寫報告。

## 環境
- Workspace: `/Users/pingu/.openclaw/`
- Memory: `/Users/pingu/.openclaw/memory/`
- Topics: `/Users/pingu/.openclaw/memory/topics/` — [新增] 穩定知識主題檔
- Archive: `/Users/pingu/.openclaw/memory/archive/`
- 通知: `message` tool（action=send, channel=discord, to="channel:1477001361001549894"）
- 時區: Asia/Taipei
- 清理策略: `/Users/pingu/.openclaw/maintenance/cleanup-rules.md`（每次執行前先讀）

## 0. 載入策略
- 讀取 `maintenance/cleanup-rules.md`，裡面定義了各類檔案的清理閾值和白名單
- 如果檔案不存在，用本文件的預設值執行

## 1. 記憶整理

### 掃描 memory/ 根目錄
- 列出所有 `YYYY-MM-DD-*.md` 散檔（帶後綴的，不含今天的）
- **[修改] 明確排除 `memory/topics/` 目錄** — topics/ 是長期穩定知識，daily 不碰
- **[修改] 明確排除 `memory/state/` 目錄** — state/ 是系統狀態檔
- 對每個散檔判斷：
  - **已完成/過時** → 提取關鍵結論（1-3 行），追加到對應日期的 `YYYY-MM-DD.md`，然後移動原檔到 `archive/YYYY-MM-late/` 或 `archive/YYYY-MM-early/`（15 號為分界）
  - **仍在進行** → 保留不動，在報告中標註

### [修改] 命名規範合規檢查
- 掃描 `memory/` 根目錄，偵測不符規範的檔案：
  - 非 `YYYY-MM-DD.md` 格式的散檔（排除 `experiments.md`、`main.sqlite`）
  - 非 `YYYY-MM-DD-*.md` 格式的帶後綴散檔
  - 任何不屬於 `topics/`、`archive/`、`state/` 的子目錄（如 `agent-sociology/`）
- 不合規的檔案 → 在報告中標記 ⚠️，不自動移動

### 壓縮冗長 transcript
- 對保留的散檔，檢查是否為未壓縮的 session transcript（特徵：含大量 `user:`/`assistant:` 來回對話、行數 >40）
- 壓縮方式：提取關鍵結論和決策（保留結果，移除對話過程），目標 <20 行
- 壓縮後覆寫原檔，在報告中記錄壓縮了哪些檔案及前後大小

### 檢查 memory/ 檔案數
- 如果根目錄（不含子資料夾）超過 15 個 `.md` 檔 → 標記為 ⚠️ 需要手動整理

## 2. Workspace 清理（自動執行）

根據 `cleanup-rules.md` 中的白名單規則自動清理。

**[修改] 時間判斷統一用 `-newermt` 取代 `-mtime`**（更可靠，見 cleanup-suggestions 03-02、03-04 案例）。
- 範例：`find ... -not -newermt "$(date -v-7d +%Y-%m-%d)" -delete` 取代 `find ... -mtime +7 -delete`

### 根目錄暫存
- `.bak` 檔案：保留最新 1 個，其餘刪除
- `screen0_*.png`：超過 3 天的刪除
- `temp_*.txt`、`*.tmp`、`*.swp`：超過 1 天的刪除

### media/inbound/
- 超過 7 天的檔案 → 刪除（用 `trash` 指令優先，不可用時才 `rm`）
- 報告清理了多少檔案、釋放多少空間

### cron/runs/
- 超過 14 天的 `.jsonl` log → 刪除
- `jobs.json.bak.*` 前綴的備份：保留最新 2 個，其餘刪除

### cron/ 根目錄
- `jobs.json.bak`（不帶日期後綴的）→ 保留不動（OpenClaw 內部用）

## 3. 健康檢查
- 執行 `openclaw gateway status`，記錄狀態
- 執行 `openclaw cron list`（或等效指令），檢查有無失敗的 cron job
- 檢查 `main.sqlite` 大小，超過 10MB 標記 ⚠️
- **[新增] 孤立空檔偵測**：掃描根目錄的 `.db`、`.sqlite` 檔案，0-byte 的標記 ⚠️（參考 03-06 建議）
- **[新增] 重複/誤放目錄偵測**：檢查 `memory/` 下是否存在不屬於標準子目錄（topics/、archive/、state/）的子目錄（如 `agent-sociology/`），標記 ⚠️

### [新增] Session-end State Freeze 驗證（輕量）
- 掃描 `projects/*/status.md`，找出今天有修改過的
- 檢查這些 status.md 是否包含「當前焦點」段落
- 缺少的 → 報告中標記 ⚠️（僅提醒，不自動修改）

## 4. 輸出

### 報告檔
寫入 `memory/YYYY-MM-DD.md`（今天日期），追加到已有內容後：

```
## Daily Cleanup Report
- 整理：X 個散檔歸檔，Y 個保留，Z 個壓縮
- 合規：memory/ 命名合規 [OK/⚠️ N 個不合規]
- 清理：media X 檔(YMB)、暫存 X 檔、cron logs X 檔
- 檔案數：memory/ 目前 Z 個檔案 [OK/⚠️]
- 健康：gateway [OK/⚠️], cron [OK/⚠️], DB size [OK/⚠️]
- 空檔偵測：[OK/⚠️ 列出]
- state freeze：[OK/⚠️ 列出缺少的]
- 發現：（任何異常或新的清理建議）
```

### 迭代紀錄
如果在執行中發現「現有規則沒覆蓋到的問題」或「規則可以改進的地方」：
- 將建議寫入 `maintenance/cleanup-suggestions.md`（追加模式）
- 格式：`- [YYYY-MM-DD] 建議內容`
- 這些建議會在 weekly cleanup 時被 review 並決定是否採納

### 通知規則
- 全部正常 → Discord 簡短報告「Daily Cleanup OK」+ 歸檔/清理數量
- 有任何 ⚠️ → Discord 簡短說明異常
- 有新的 cleanup suggestion → 在報告末尾附一行提示

---

## 變更紀錄（相對 v1）

| 類型 | 項目 | 理由 |
|------|------|------|
| 修改 | 記憶整理排除 topics/ | topics/ 是長期知識，daily 不應動 |
| 新增 | 命名規範合規檢查 | 配合 memory/ 命名規範（CONVENTIONS.md） |
| 修改 | `-mtime` → `-newermt` | cleanup-suggestions 03-02、03-04 實證更可靠 |
| 新增 | 孤立空檔偵測 | cleanup-suggestions 03-06 建議 |
| 新增 | 重複/誤放目錄偵測 | cleanup-suggestions 03-06 建議 |
| 新增 | State freeze 驗證 | 配合 Tier 2 session-end freeze |
| 修改 | 報告格式擴充 | 新增合規、空檔、freeze 欄位 |
