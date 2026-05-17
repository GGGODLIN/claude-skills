---
name: research-before-answer
description: Invoke BEFORE answering ANY question involving facts - specific repo, package, library, framework, SDK, API, version, release notes, deprecation, company, product, market, or person activity. Prevents hallucination from stale training data. Trigger whenever the user asks about a named tool, project, repo, library, version, or the current state of the world. Also invokable manually via /research-before-answer.
---

## Prerequisites

This is a framework template, not drop-in. To use this skill you need to self-build:

- `~/.claude/hooks/websearch-parallel-enforce.sh` — self-build; without it the "ticket model" parallel SERP enforcement won't work. You can skip the ticket-model section if not building this hook.
- `~/.claude/scripts/fetch-fallback.sh` — self-build; without it the WebFetch fallback ladder (reddit→.json / Googlebot-UA / jina / archive) doesn't run. Document at the script's design intent paragraph, build your own version.
- `~/.claude/methodologies/progressive-interrogation.md` — self-build; optional reference for single-source deep interrogation pattern.
- `~/.claude/wiki/` — self-build personal knowledge base for "individual layer". Without it, skip individual-layer instructions and only use external research tools.
- `~/.claude/projects/{{SCOPE}}/memory/` — self-build 2-level cluster memory schema. Without it, skip individual-layer Step 0-3.

**This is a framework, not a drop-in tool.** The skill describes a research discipline; reader is expected to fork and adapt to their own setup.

# Research Before Answer

涉及**事實**的問題預設**先查證再回答**，不要用訓練資料編。

## 觸發條件（符合任一即 invoke）

- 具體 repo / 專案 / 套件 / library / framework / SDK
- API / CLI 用法、版本、release notes、deprecation
- 公司、產品、人物、市場近況
- 任何「現狀」「最新」「最近」「這個 X 怎麼用」類問題
- 混合題（框架 + 事實）→ 先查事實部分，再套框架

**不觸發**：純框架 / 原則 / 設計模式 / 概念解釋（用訓練知識即可）。

## 並行策略：個人 + 外部（預設）

本地（wiki + session memory）與外部採**並行 + merge**，不是 wiki 優先。

### 個人資料層

- **wiki** (`~/.claude/wiki/`): 個人經驗／偏好／踩過的坑
  - 寫/改 entity 前：(1) `cat ~/.claude/wiki/_schema.md` 確認格式 (2) `grep -ri "<keyword>" ~/.claude/projects/*/memory/` 撈既有 reference / project / feedback memory，不要只憑當次對話
- **session memory** (`~/.claude/projects/*/memory/`): hand-curated reference / project / feedback / user 4 類 .md + 兩層 index 結構（MEMORY.md + `_index_*.md` cluster files）
  - Step 0：**query 含個人化線索（「我的 X setup」「我這台機器」「我之前 X 怎處理」「我有沒有寫過」）→ 並行** `grep -rli "<keyword>" ~/.claude/projects/*/memory/` 列出 hit file，補捕 MEMORY.md 沒列 pointer 但 file 還在的 standalone reference（MEMORY.md 採 on-demand-friendly trim、許多 standalone 移除 pointer 但保留 file）
  - Step 1：`cat ~/.claude/projects/<scope>/memory/MEMORY.md` 看索引（含 `### Cluster indexes` + `### Standalone reference` 兩段）
  - Step 2：MEMORY.md 看到相關 cluster pointer → `Read ~/.claude/projects/<scope>/memory/_index_<topic>.md` 看 sub-entries
  - Step 3：drill 進具體 topic file 取細節（**注意**：cluster index 內每 entry 的 1 行 description 通常已足夠形成 informed opinion，不要無腦 Read 對應 topic file 浪費 context）
  - Cluster 結構是動態的（每次重新 `ls _index_*.md` 拿當下狀態，不要記住固定 cluster list），規則見 `feedback_memory_cluster_maintenance.md`

### 外部工具分流

針對問題核心選工具，不要泛泛「搜相關資料」：

| 問題類型 | 工具 |
|---|---|
| SDK / library 用法 | Context7 (`mcp__context7__*`) |
| GitHub repo / issue / PR | `gh` CLI |
| 套件（npm / pypi / crates） | registry search 或 WebSearch |
| 市場 / 人物 / 公司動態 | WebSearch、Exa |
| 深度多輪研究 | `mcp__gemini__gemini-deep-research` |
| Shopify | `mcp__shopify-dev-mcp__*` |

### 搜尋 layer：同 query 一輪 nav + WS

任何「給 query 找 URL」場景**一輪完整搜尋必須包含同 query 的 chrome nav + WebSearch**：

- `mcp__claude-in-chrome__navigate` 到 `google.com/search?q=<query>` — 走 daily chrome 拿真 Google ranking + cookie personalization；配 `mcp__claude-in-chrome__javascript_tool` 抽 SERP JSON
- `WebSearch` query=`<同 query>` — 快、URL list、token 省，但 ranking 已知會漏官方 doc

兩者 cross-reference 比對哪個訊號更全。

**規則（ticket model）**：

- 每張 nav 給一張 query-specific ticket，每次成功 WebSearch 消耗一張同 query ticket
- **順序**：先 navigate 再 WebSearch（不是嚴格同 message 並行 — `PreToolUse` hook fire 時 nav 必須已寫進 transcript，所以拆兩個 message 比較穩）
- **被 block 的 WS 不消耗 ticket**（retry-friendly：補 nav 後可直接 retry 同 query WS）
- **同 query 跑兩次成功 WS 會被擋**（結果一樣浪費 quota；要再跑請補新 nav 表達明確意圖）
- **不同 query 不共用 ticket**（每個 query 自己配對自己的 nav）

Hook: `~/.claude/hooks/websearch-parallel-enforce.sh`。覺得不好用 → 移除 `settings.json` PreToolUse `WebSearch` matcher。

SERP 抽取 JS pattern 跟試用背景見對應 reference / feedback memory entries（如 `reference_search_tool_comparison_*.md`、`feedback_websearch_misses_official_docs.md`、`feedback_parallel_decision_by_structural_failure.md`）。

### URL 內容抓取

預設用 `WebFetch`（server-side、快、token 省，99% URL 都 work）。

**WebFetch 拿 4xx/5xx 或內容明顯不夠（403 body / "Forbidden" / 亂碼 / < 200 字模板 / "Please enable JavaScript"）→ 必須立刻 fallback，不要回覆「官網沒爬到」「403 抓不到」就跳過——fallback 是 mandatory 不是 optional。** 兩段退路階梯：

1. `bash ~/.claude/scripts/fetch-fallback.sh <url>` — reddit→`.json`;其餘一律通用 Googlebot-UA+JSON-LD→jina→archive(少數站 Bingbot/AMP 特化、非 gate)。exit 0=內容在 stdout;**exit 75 或 exit 1 → 升下一段**
2. exit 75/1 才 `mcp__claude-in-chrome__navigate` — 純瀏覽器才解的（CAPTCHA / 登入 wall / JS-only）

常見觸發：

- reddit.com 整域 hardcoded refuse
- 站需 cookie / 登入 wall
- JS-only 內容 server-side fetch 拿不到
- WebFetch 抓回來但 incomplete（亂碼 / 模板 / 空白）

抽取 JS snippet + SPA 變體可建對應 reference memory entry 紀錄。

不要無腦平行——大多 URL WebFetch 直接搞定，並行 claude-in-chrome 純浪費 token + 開 chrome window。

PullMD MCP 不在這支 skill 走——其 dual-fetch 對照協議僅在特定情境（例如每日報告場景）啟用，依個別 project CLAUDE.md 為準。

### 執行模式

- **淺題**（單次查）：`cat ~/.claude/wiki/index.md` + 外部工具同 response 分段 merge
- **深題**（多輪研究）：派**背景 subagent**（`run_in_background: true`），下則 turn 主動 `TaskGet` 取結果 merge
- **單邊**：純個人經驗 → 只 wiki；純外部即時（SDK / API spec / 公司動態）→ 只外部
- **Cold case**（wiki 沒收 + 外部沒答案）：`grep -ri "<keyword>" ~/.claude/projects/*/memory/` 翻既有 reference / project memory；不夠再 grep 對應 .jsonl session log（如 `grep -l "<keyword>" ~/.claude/projects/**/*.jsonl`）
- **衝突時**：外部為準，主動提議「`wiki/X.md` 可能過期，要更新嗎？」
- **單源鑽深**：深題且有單一可信主來源要榨乾時，參照 `~/.claude/methodologies/progressive-interrogation.md` 的序列累積審訊骨架（與本 skill 的並行 local+external 互補）

## 記憶升級

學到 non-obvious pattern → 主動提議升級 wiki entity（使用者點頭才寫）。

## 手動觸發

`/research-before-answer` 在對話中直接呼叫，無需等待自動觸發。

## 核心原則

**寧可多搜一次，不要編。**
