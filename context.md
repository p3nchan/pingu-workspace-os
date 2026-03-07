# Workspace Redesign - Context

## 目的
重新設計 Pingu AI Agent 的檔案管理、記憶系統、專案管理架構，解決以下核心問題：
1. **Files drift** — 檔案跑到錯誤目錄或成為孤兒檔
2. **Context fragmentation** — 相關資訊分散在 daily notes、project files、chat history
3. **Memory decay** — 重要決策/context 隨 daily notes 老化而遺失
4. **Naming inconsistency** — 無強制命名規範
5. **Discovery problem** — agent 和 Penchan 有時忘記有哪些檔案/資源

## 研究基礎
- 3 份外部 Deep Research（ChatGPT / Perplexity / 學術導向報告）
- 現有系統：AGENTS.md + ORCHESTRATION.md + MEMORY.md

## 研究材料位置
- `research/chatgpt-deep-research.txt` — ChatGPT Deep Research 完整報告
- Perplexity: `../../media/inbound/b6ca437b-42fc-4e0b-9956-218a91e4d3a6.txt`
- 學術報告: `../../media/inbound/6dd963d0-899c-4750-990c-547c593db8bf.txt`

## 核心共識（3/3 研究都同意）
1. Markdown 當 source of truth，索引層另建
2. 啟動時只載入少量入口檔，其餘按需讀取
3. PARA 不適合 agent，Johnny Decimal 穩定 ID 概念最值得借鑑
4. 記憶分層：ephemeral → episodic → semantic/permanent
5. Sub-agent = 無狀態純函數，主 agent 持有全域 state
6. 防 context rot 核心是 State Freezing，不是保留更多歷史

## 現有系統文件
- `~/.openclaw/AGENTS.md` — 當前規則與結構
- `~/.openclaw/ORCHESTRATION.md` — 編排框架
- `~/.openclaw/MEMORY.md` — 長期記憶索引
