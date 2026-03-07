# Cleanup Rules — 可調參數
# v2.0 — 2026-03-07 — workspace-redesign 適配：topics/ 規則、-newermt 統一、空檔偵測

Daily/Weekly cleanup agent 執行前載入此檔，作為清理的判斷依據。
Weekly cleanup 會根據 `cleanup-suggestions.md` 的建議來更新此檔。

---

## [修改] 時間判斷方法

所有基於時間的清理，統一使用 `-newermt` 而非 `-mtime`：
```bash
# 正確（v2）
find <path> -not -newermt "$(date -v-7d +%Y-%m-%d)" ...

# 棄用（v1）
find <path> -mtime +7 ...
```
理由：`-mtime` 以 24 小時為單位計算，邊界行為不直覺（03-02 案例：Feb 22 檔案被漏掉；03-04 案例：`-mtime +7` 只抓 2 檔 vs `-newermt` 抓 18 檔）。

---

## 暫存檔（daily 自動清理）

| 類型 | 路徑模式 | 保留規則 | 說明 |
|------|----------|----------|------|
| .bak | `~/.openclaw/*.bak` | 保留最新 1 個 | 根目錄備份 |
| screenshots | `~/.openclaw/screen0_*.png` | >3 天刪除 | Peekaboo 截圖 |
| temp files | `~/.openclaw/temp_*.txt` | >1 天刪除 | 臨時文字檔 |
| swap/tmp | `~/.openclaw/*.tmp`, `*.swp` | >1 天刪除 | 編輯器暫存 |

## media/inbound/（daily 自動清理）

- 閾值：**7 天**
- 動作：`trash`（優先）或 `rm`
- 豁免：無

## cron/runs/（daily 自動清理）

- Log 保留天數：**14 天**
- `jobs.json.bak.*` 備份：保留最新 **2** 個

## browser/（weekly 清理）

- 清理目標：`browser/openclaw/user-data/` 下的 cache 類目錄
- 清理清單：`Cache`, `Code Cache`, `GPUCache`, `Service Worker/CacheStorage`, `ShaderCache`
- 保留不動：`Local Storage`, `Session Storage`, `IndexedDB`, `Cookies`, `Preferences`, `Bookmarks`

## canvas/（weekly 報告）

- 報告閾值：**20MB**
- 動作：僅報告，不自動清理

## media/ 其他（weekly 報告）

- 報告閾值：任何子目錄 **>50MB**
- 動作：僅報告

## cron jobs 衛生（weekly）

- Disabled job 建議刪除閾值：**14 天**
- 動作：僅建議，不自動刪除

## memory/（daily）

- 根目錄 `.md` 檔案數上限：**15**（不含子目錄內的檔案）
- 散檔壓縮閾值：**40 行**
- 壓縮目標：**<20 行**
- 標準子目錄白名單：`topics/`, `archive/`, `state/`
- 其他子目錄視為異常 → 標記 ⚠️

## [新增] memory/topics/（daily 不碰，weekly 維護）

- 每個 topic file 大小目標：**≤ 1KB**
- 每個 topic file 硬上限：**1.5KB**（超過必須壓縮或拆分）
- topic file 總數上限：**10 個**（超過建議合併相關主題）
- 命名規範：`kebab-case.md`，不含日期前綴
- Daily cleanup **不可修改** topics/ 下的檔案
- Weekly cleanup 可依「記憶精煉 SOP」更新

## [新增] 孤立空檔偵測（daily）

- 掃描根目錄的 `.db`、`.sqlite` 檔案
- 0-byte 的檔案 → 標記 ⚠️
- 動作：僅報告，不自動刪除（可能是其他程式的 lock file）

## [新增] catalog.md（weekly 重建）

- 位置：workspace 根目錄 `catalog.md`
- 頻率：每週全量重建
- 涵蓋：`projects/`、`skills/`、`workflows/`
- 格式：markdown 列點，每項一行 + 一句話描述
- 動作：覆寫，不做差異比對

---

## 變更紀錄
- v2.0 (2026-03-07): workspace-redesign 適配
  - [新增] 時間判斷統一用 `-newermt`
  - [新增] `memory/topics/` 規則區塊（大小上限、數量上限、命名規範）
  - [新增] 孤立空檔偵測規則
  - [新增] catalog.md 重建規則
  - [新增] memory/ 標準子目錄白名單
  - [修改] 明確 memory/ 根目錄上限 15 不含子目錄
- v1.0 (2026-03-01): 初始版本，根據 workspace 現況制定
