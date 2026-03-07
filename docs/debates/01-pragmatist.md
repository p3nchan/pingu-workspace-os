# 保守實用派：Workspace 重構辯論

> 角色：Pragmatic Minimalist
> 核心信念：「不修沒壞的東西。過度工程殺死動能。每次結構變動都有遷移成本。」
> 日期：2026-03-07

---

## A. 現有系統的防禦：我們已經做對的事

在讀完兩份研究文件後，我的第一反應不是「天啊我們落後好多」，而是「我們已經走在正確路上了」。以下是現有系統已經符合或等效於研究建議的具體項目：

### 1. Hierarchical Supervisor + Stateless Sub-agents（已完全實現）

研究文件（學術篇 Section 4）強力推薦 Two-Tier Agent Model，sub-agent 作為無狀態純函數。我們的 `ORCHESTRATION.md` 第一章就寫死了這個架構：「Pingu = 唯一 Orchestrator，Sub-agents = 無狀態純函數」。這不是巧合，這是一個月實戰經驗的產物。連 sub-agent 寫入保護（`AGENTS.md` 第 57-60 行）都已經到位。

**結論：研究建議 = 我們的現狀。不需要改。**

### 2. 分層啟動序列 / Cold Start Boot Sequence（已實現）

Perplexity 研究建議「分層載入：先讀全域規則、使用者偏好、專案主規則、精簡記憶索引，再依任務動態讀專案狀態」。我們的 `AGENTS.md` 第 3-9 行就是這個流程：SOUL.md -> USER.md -> daily notes -> MEMORY.md -> brain.md -> ORCHESTRATION.md（條件式）。`ORCHESTRATION.md` Section 3 還有完整的 Cold Start Boot Sequence（7 步驟），包含掃描 `projects/*/status.md` 找 in_progress 任務。

**結論：等效機制已存在。差異只是我們沒有用 JSON 而是用 markdown，但這反而更好（人類可讀、版本控制友善）。**

### 3. Lazy Loading / 按需讀取（已實現）

研究文件說「啟動時只讀少數入口檔最有效率」，Claude Code 把 topic files 設為按需載入。我們的 `AGENTS.md` 第 91-94 行已經有完整的 Lazy Loading 流程：提到專案才讀 context.md，需要進度才讀 status.md，深入子專案才順著索引讀。

**結論：已是現有設計的核心。**

### 4. 記憶分層（Working / Episodic / Semantic）（已實現）

LangGraph 研究把記憶分成 short-term thread state、episodic、semantic 三層。我們的 `ORCHESTRATION.md` Section 3 明確定義了三層：Working Memory（task-level context bundle）、Project State（`status.md` + `tasks/` + checkpoints）、Long-term Memory（`MEMORY.md` + `memory/*.md`）。

再加上 `memory/YYYY-MM-DD.md` 是天然的 episodic memory，`MEMORY.md` 是 semantic memory（穩定事實），`projects/*/status.md` 是 working memory。完全對應。

**結論：架構等效，只是命名不同。**

### 5. 記憶壓縮 / Promotion 機制（已實現）

研究建議「daily note 中的重要事實升格到 permanent memory」。我們的 `AGENTS.md` 第 47-54 行已經有「記憶壓縮 SOP（聯合重寫）」：讀現有 MEMORY.md -> 讀最近日誌 -> 聯合重寫 -> 控制 <1KB。這就是 promotion 機制。

**結論：已有等效流程，且有明確的 SOP。**

### 6. Single Source of Truth + Cross-reference 紀律（已實現）

學術研究要求「防止 file drift」和「文件與執行狀態綁定」。我們的 `AGENTS.md` 第 119-123 行明確規定：每個知識只有一個權威檔案，其他地方只放相對路徑連結不複製內容，修改只改權威檔案。

**結論：原則已建立並執行中。**

### 7. ADR（Architecture Decision Records）（已實現）

研究建議「決策一律 append-only 寫進 decisions/」。我們的 `AGENTS.md` 第 102-117 行已有完整的 ADR 機制：獨立成檔、放母專案 `decisions/`、有精簡模板、有判斷標準。

**結論：已超越多數研究建議的細緻度。**

### 8. Circuit Breaker + Error Recovery（已實現）

這在很多研究中被當作進階功能，但我們的 `ORCHESTRATION.md` Section 4 已有完整的 Circuit Breaker 規則（3 次連續失敗 -> OPEN -> 5 分鐘冷卻 -> HALF-OPEN）、5 層 Fallback 金字塔、Retry vs Restart vs Escalate 決策樹。這些都是從實際 debug 經驗（Gemini Rate Limit 雪崩事件）中長出來的。

**結論：不只有，而且是從實戰中驗證過的。**

### 9. HITL Escalation 分級（已實現）

研究建議用 risk-tier 分級 Human-in-the-Loop。我們的 `ORCHESTRATION.md` Section 5 有完整的三級分類（Autonomous / Notify / Gate），附帶 Decision Card 格式和反模式警告（over-escalation / under-escalation）。

**結論：已有，且比研究建議更細緻。**

### 10. Markdown 作為 Source of Truth + 檔案系統友善（已實現）

兩份研究都同意：markdown 保留為可審計、可手改、可版本控制的 source of truth，再加一層索引。我們整個系統就是建立在 markdown 上的，包含 `.git` 版本控制。

**結論：這是我們的根基，而且是正確的根基。**

---

## B. 最高 ROI 前 3 名

從所有研究建議中，我只挑三個。不是因為其他不好，而是因為這三個解決的是**真實痛點**，而不是理論上的完美。

### ROI #1：MEMORY.md 索引化 + 主題拆分

**問題描述**：`MEMORY.md` 目前 7KB / 105 行，雖然有 <1KB 目標但已經超標 7 倍。隨著系統運行，它會持續膨脹。每次啟動都全量載入這 7KB 到 context window，其中大部分內容跟當前任務無關（例如做 crypto 研究時不需要載入八字命盤和 macOS 自動化資訊）。

**解法**：
- 把 `MEMORY.md` 瘦身為純索引檔（<1KB），只放分類目錄 + 檔案路徑
- 長內容拆到 `memory/permanent/` 下的主題檔案：`preferences.md`、`tools.md`、`model-config.md`、`debug-patterns.md` 等
- 啟動時只讀索引，按需讀主題檔

**預期效果**：啟動載入 token 減少 80%+，context window 留給真正有用的內容。

**實施成本**：低。只是把現有內容切分到新檔案，加幾行索引。不影響任何現有流程。

**時間估計**：1-2 小時，一次性。

### ROI #2：Daily Notes 命名規範化 + 清理機制

**問題描述**：看 `memory/` 目錄，2026-03-06 一天就產生了 20 多個檔案（`2026-03-06-binance-guide.md`、`2026-03-06-content-pipeline.md`、`2026-03-06-thesis-research.md`...），加上主 daily note `2026-03-06.md`。目前 `memory/` 已有 297 個 md 檔案，但系統才運行不到 2 週。照這個速度，3 個月後會有 1000+ 檔案，6 個月後超過 2000 個。這些 sub-agent session log 和任務產出混在 memory 目錄裡，跟「記憶」沒有關係。

**解法**：
- 建立命名規範：`memory/YYYY-MM-DD.md` 只留主 daily note（一天一個）
- Sub-agent 的 session log 移到 `logs/sessions/YYYY-MM-DD/` 或對應專案目錄下
- 任務產出（如 `2026-03-06-thesis-research.md` 這種 48KB 的研究檔）歸到 `projects/thesis/` 或 `references/`
- 每週 cron 清理：30 天前的 daily notes 壓縮歸檔到 `memory/archive/`

**預期效果**：`memory/` 目錄保持乾淨（任何時候 <50 個檔案），檔案歸位到正確的目錄，discovery 效率提升。

**實施成本**：低到中。需要調整 sub-agent spawn 時的檔案寫入路徑規則，加一條 cron job。

**時間估計**：2-3 小時設定規則 + cron，之後自動運行。

### ROI #3：建立輕量 catalog 索引

**問題描述**：目前有 23 個專案目錄、60+ 個 skill 目錄、10+ 個 workflow。當需要找「某個東西在哪裡」時，agent 要掃描多個目錄。沒有一個地方能快速回答「目前活躍的專案有哪些？」「哪個 skill 處理 crypto？」。`brain.md` 部分扮演了這個角色，但它是手動維護的待辦清單，不是系統索引。

**解法**：
- 建立 `orchestration/catalog.md`（不是 JSON，不是 SQLite，就是一個 markdown 檔）
- 內容：活躍專案清單（名稱 + 狀態 + 路徑）、skill 分類索引、workflow 索引
- 每週 cron 自動重建（掃 `projects/*/status.md` 的狀態 + `skills/*/SKILL.md` 的描述）
- 啟動時可選擇性載入（只在需要跨專案導航時讀）

**預期效果**：減少 agent 在找檔案時的無效探索，提升跨專案任務的啟動速度。

**實施成本**：低。一個 cron job + 一個 markdown 模板。

**時間估計**：1-2 小時。

---

## C. 我反對的建議（7 個）

### 1. 反對：引入 Johnny Decimal 編號系統

**研究主張**：Johnny Decimal 提供 O(1) 確定性檔案定位，消除路由幻覺。

**我的反對理由**：我們目前有 23 個專案。二十三個。不是 2,300 個。agent 用 `ls projects/` 一條指令就能看到全部。Johnny Decimal 的優勢在於「大型組織中人類記不住幾百個資料夾位置」，但我們的 agent 可以瞬間讀取整個目錄結構。此外，遷移成本巨大：所有現有路徑引用（`projects/fiscus/status.md` 變成 `10-19_Content/11_Blog_Posts/11.01_...`）都要改，`AGENTS.md`、`ORCHESTRATION.md`、所有 `context.md` 中的相對路徑全部失效。這種遷移本身就是 file drift 的最大來源。

**判決**：解決的問題在我們的規模不存在，遷移成本遠大於收益。

### 2. 反對：引入 Zettelkasten 原子筆記系統

**研究主張**：原子筆記 + 雙向連結，適合 agent 的語意圖譜遍歷。

**我的反對理由**：Zettelkasten 的核心價值在於「意外發現」——在筆記之間建立非預期的連結。我們的 agent 不需要「意外發現」，它需要的是「精確定位」。目前 `MEMORY.md` 105 行，`memory/permanent/` 還不存在。在我們只有幾十個知識節點的規模下，引入雙向連結和原子化筆記是把 5 件衣服放進一個能裝 500 件的衣櫃——系統開銷完全不成比例。更重要的是，Zettelkasten 需要持續高品質的連結維護，這是額外的 token 成本和複雜度，而我們的 agent 時間應該花在做事上。

**判決**：知識規模太小，引入成本不合理。等知識庫超過 500 個節點再說。

### 3. 反對：部署 SQLite + 向量資料庫作為檢索層

**研究主張**：hybrid search（vector + BM25），semantic retrieval，O(1) 語意查詢。

**我的反對理由**：`memory/main.sqlite` 已經存在（5.7MB），但那是 OpenClaw 框架自己的。研究建議的是自建一套向量索引 + 全文搜尋。我們的 workspace 總共約 470 個 md 檔案，`memory/` 8.3MB，`projects/` 14MB。這個規模，`grep` 就能在毫秒內搜完。向量資料庫適合的是「10,000+ 文件，語意搜尋比關鍵字搜尋更有效」的場景。我們連用 grep 都感覺不到延遲。加上維護 embedding pipeline（每次檔案更新要重新 embed）的複雜度，完全不值得。

**判決**：規模差兩個數量級。這是未來 12-18 個月的事，不是現在。

### 4. 反對：YAML Frontmatter 全面導入

**研究主張**：每個重要檔案加 YAML frontmatter（id、type、project、status、created、updated、reviewed、source_of_truth、related）。

**我的反對理由**：目前 agent 讀一個 `status.md` 只需要看內容就知道狀態。加了 frontmatter 後，每個檔案多了 8-10 行 metadata，且必須在每次修改時更新 `updated` 欄位。170+ 個專案 md 檔 + 297 個記憶檔 + 60+ 個 skill 檔 = 全面加 frontmatter 是一個龐大的遷移工程。更關鍵的是：frontmatter 的價值在於有工具消費它（如 catalog 生成器）。在我們沒有建立自動消費 frontmatter 的工具之前，它只是增加每次編輯的認知負擔。

**判決**：先建 catalog，如果 catalog 的手動維護成本過高，再考慮用 frontmatter 自動化。不要倒著來。

### 5. 反對：狀態檔從 Markdown 改為 JSON/YAML

**研究主張**：「State Freezing and Injection」——把專案狀態編譯成 machine-checkable 的結構化格式（JSON/YAML），取代敘事型 markdown。

**我的反對理由**：我們的 `status.md` 模板（`AGENTS.md` 第 69-88 行）已經是半結構化的：有固定欄位（概要、當前焦點、Phases、進度歷史、Blockers），控制在 500 字以內。Penchan 可以直接在 Discord 或 Finder 裡讀。改成 JSON 後，人類可讀性歸零，debug 時要先解析才能看到內容。我們是單人+單 agent 系統，不是需要 machine-checkable schema validation 的企業系統。Markdown 的半結構化已經夠用，且 agent 理解 markdown 的能力（因為訓練資料分佈）實際上比 JSON 更穩定。

**判決**：解決的問題（schema validation / machine parsing）在單用戶系統中不存在。markdown 的人機雙讀優勢是我們的核心資產。

### 6. 反對：Cryptographic Hash 驅動的 Drift Prevention

**研究主張**：文件引用的程式碼片段附帶 hash，每次 commit 前驗證，不一致就 fail-closed。

**我的反對理由**：這是為企業級多團隊 CI/CD pipeline 設計的。我們的 workspace 不是 codebase，我們的 markdown 不引用程式碼片段，我們沒有 deploy pipeline。File drift 在我們這裡的表現是「status.md 忘記更新」，解法是提醒 agent 更新（已經在 `AGENTS.md` 寫入規則中），不是建一套 hash 驗證系統。Hash-based drift prevention 的實施成本（建驗證腳本、每個文件加 hash、每次修改要重算 hash）遠超我們的需求。

**判決**：用大砲打蚊子。繼續用現有的 SOP 規則即可。

### 7. 反對：Session Init 驗證腳本（JSON log + validation script）

**研究主張**：每次啟動必須生成結構化 session JSON log，通過獨立 Python 驗證腳本，才能開始工作。

**我的反對理由**：我們的 boot sequence 是 agent 讀 markdown 入口檔。加一個強制的 validation gate 意味著：(1) 要開發並維護一個 Python 腳本；(2) 每次啟動多一個 tool call（執行驗證）；(3) 驗證失敗時要有 fallback 機制。這帶來的複雜度和我們實際遇到的「啟動問題」不成比例——我們的啟動問題是偶爾忘記讀某個檔，解法是在 `AGENTS.md` 的清單裡加一行，不是建一套 validation pipeline。

**判決**：為罕見問題引入永久性複雜度。用清單就好。

---

## D. 風險評估

### 如果什麼都不改，6 個月後最可能出的問題：

1. **memory/ 目錄爆炸**：照目前速度（每天 10-20 個檔案），6 個月後 `memory/` 會有 2,000-3,000 個檔案。`ls memory/` 會變慢，agent 找特定日期的筆記要翻很多頁。這是**最急迫的真實問題**。

2. **MEMORY.md 持續膨脹**：雖然有 <1KB 目標，但實際已經 7KB 且持續增長。每次啟動都把不相關的 7KB 灌進 context window，日積月累的 token 浪費可觀。

3. **跨專案 discovery 變慢**：專案從 23 個增長到 40-50 個時，沒有索引的情況下，agent 需要更多 tool calls 來找到正確的專案目錄。

4. **Daily notes 中的有價值資訊被埋沒**：重要的決策和發現如果只停留在 daily note 裡，3 個月後就沒有人（人或 agent）會回去翻了。promotion 流程雖然有 SOP，但執行頻率不夠。

### 如果改太多，最可能出的問題：

1. **遷移期的路徑斷裂**：大規模重命名目錄（如改 Johnny Decimal）會導致所有硬編碼路徑失效。`AGENTS.md`、`ORCHESTRATION.md`、各專案的 `context.md` 中的路徑引用、cron jobs 中的路徑——每一個都是潛在的斷裂點。

2. **兩套系統並存的混亂**：部分遷移比不遷移更糟。如果一半專案用新命名、一半用舊命名，agent 的 discovery 效率反而下降。

3. **維護負擔轉嫁**：引入 frontmatter、hash validation、JSON schema 等機制，每一個都需要持續維護。agent 花在維護系統上的 token 和時間，就是不能花在做事上的 token 和時間。

4. **打斷當前動能**：系統才運行 2 週，正處於快速迭代和專案推進期。花一整週做架構遷移 = 停下所有專案。等 4-6 週系統穩定後再做大改動，ROI 更高。

5. **新架構的 bug**：任何新系統都有 bug。新的 cron job 可能漏檔，新的目錄結構可能跟某個 skill 的硬編碼路徑衝突。除錯這些新問題的成本是隱性的但真實的。

---

## E. 總結：我的處方

我的立場很明確：**這個系統的骨架是健康的**。兩份研究的核心建議——hierarchical orchestration、stateless sub-agents、分層記憶、lazy loading、markdown as source of truth、HITL escalation——我們已經全部有了，而且是從實戰經驗中長出來的，不是照搬的。

**做三件事**：
1. 把 `MEMORY.md` 瘦身為索引檔，長內容拆到 `memory/permanent/` 主題檔案
2. 規範化 `memory/` 目錄：sub-agent session log 歸位、每週清理、一天一個主 daily note
3. 建一個 `orchestration/catalog.md` 自動索引（每週 cron 重建）

**不做六件事**：
Johnny Decimal 編號遷移、Zettelkasten 原子筆記、向量資料庫、全面 YAML frontmatter、JSON/YAML 狀態檔取代 markdown、hash-based drift prevention

**觀察三件事**：
1. `memory/` 目錄的增長速度——如果清理規範到位後仍然快速膨脹，再考慮更激進的歸檔策略
2. 專案數量突破 40 個時的 discovery 效率——如果 catalog.md 不夠用，再考慮 frontmatter + 自動化 catalog
3. 知識庫總量超過 50MB 時的搜尋效能——如果 grep 開始感覺慢了，那時候再上向量資料庫

**用一句話說**：做最小的三個改動來解決最大的三個痛點，讓系統繼續跑，讓數據告訴我們下一步該做什麼。不要因為研究報告裡的理想架構圖，就把一個正在運作的系統拆掉重建。
