---
description: Vendor model 手動寫 project memory（hook Defense 0 不自動 nudge 時用）
---

## Prerequisites

This is a framework template, not drop-in. To use this command you need:

- `~/.claude/projects/{{SCOPE}}/memory/` cluster memory structure
- A Vendor model setup — this command was designed for situations where the hook Defense 0 doesn't auto-nudge memory writes; reader without same hook setup may not need this command

**This is a framework, not a drop-in tool.**

# Memory Checkpoint

判斷當前對話是否「告一段落」（解決問題 / 完成任務 / 結束 topic / 使用者明示結束）。

**否** → 回 `skip` 一字結束，不要解釋。

**是** → 跑下列流程寫入 / 更新 project memory。

## 1. 抓作者資訊

```bash
PROJECT_HASH=$(pwd | sed 's|^/|-|; s|/|-|g')
MEMORY_DIR="$HOME/.claude/projects/$PROJECT_HASH/memory"
SESSION_ID=$(ls -t "$HOME/.claude/projects/$PROJECT_HASH"/*.jsonl 2>/dev/null | head -1 | xargs -I{} basename {} .jsonl)
echo "model=${ANTHROPIC_MODEL:-unknown}  sess=${SESSION_ID:0:8}  dir=$MEMORY_DIR  ts=$(date '+%Y-%m-%dT%H:%M')"
mkdir -p "$MEMORY_DIR"
```

## 2. 決定 type 與 filename

4 類擇一：
- **reference** — 外部資源 / API / 工具 behavior 等可引用的事實
- **project** — 進行中工作 / decisions / deadlines（body 含 `**Why:**` + `**How to apply:**`）
- **feedback** — user 對工作方式的指示（body 含 `**Why:**` + `**How to apply:**`）
- **user** — user 的角色 / 偏好 / 知識背景

Filename：`<descriptive-kebab-name>.md`，能讓未來 session 從檔名判斷主題。

## 3. 寫入 memory file

`$MEMORY_DIR/<filename>.md` frontmatter：

```yaml
---
name: <短標題>
description: <一行 hook，未來判斷相關性用，要具體>
type: <reference|project|feedback|user>
editors:
  <model-name>: <ISO timestamp> sess:<8 char prefix>
staleBy: <YYYY-MM-DD>   # optional，僅當內容是會過期的外部事實（pricing / API spec 版本）才加
---
```

範例：
```yaml
---
name: GLM 200K context behavior
description: GLM-5.1 / 4.7 200K context 跟 CC fallback 對齊，AutoCompact 行為自然
type: reference
editors:
  deepseek-v4-pro: 2026-05-03T18:00 sess:1501e762
---
```

**只用上面列出的欄位**。不要自創 `originVendor` / `author*` / 其他 metadata 欄位 —— schema 保持精簡。

**關於 `originSessionId`**：CC binary 會在 Write/Edit 後自動 inject `originSessionId: <最後一次 Edit 的 session>` 到 frontmatter。這**不是你加的**，是 CC 內部 auto memory 機制。**不用刻意處理**（不要寫進去、不要試圖刪掉，CC 會持續維護）。

Body 寫主體內容。

## 4. 修改既有 memory（同 model 多次編輯）

若 memory file 已存在：
- Read 現有 frontmatter
- `editors` map 中，**覆蓋** 自己 model 的 entry（timestamp + sessionId 都更新為當前），其他 model 的 entry **保留**
- 同一個 model 永遠只佔一筆，不爆漲

## 5. 維護索引（含 cluster check, 2026-05-07 起）

⚠️ MEMORY.md 用兩層 index 結構 —— 加 entry 前必須走 cluster check，**不可** 直接 append 到 MEMORY.md（會抹掉 trim 結構）。原則性指引 + 完整決策樹見 `feedback_memory_cluster_maintenance.md`。

### 動態 audit cluster 結構

寫 reference / project 前先：
```bash
ls "$MEMORY_DIR"/_index_*.md 2>/dev/null
```

各 cluster file frontmatter `description` 描述涵蓋範圍。判斷新 entry 屬於哪個 cluster（**不要記住固定 cluster list，每次 audit 拿當下結構**——cluster 會被 add / split / merge / rename）。

### 決策

| Type | 流程 |
|---|---|
| **feedback** / **user** | 永遠 standalone in MEMORY.md（不入 cluster） |
| **reference** / **project** | 走 cluster check → 屬於既有 cluster → 更新 `_index_<topic>.md`（append entry + bump frontmatter count）；MEMORY.md 不動 |
| 同主題 standalone 累積 ≥3 條 | 建新 cluster `_index_<新 topic>.md` + 在 MEMORY.md `## Reference / ### Cluster indexes` 加 cluster pointer |
| 找不到 cluster 也不夠累積 | 加 MEMORY.md 對應 section standalone（最後手段） |

### `ls scan` 場景

新 project memory dir → 建 MEMORY.md。**`ls $MEMORY_DIR/*.md` 要排除 `_index_*.md`** —— cluster file 不該被當 standalone entry 索引進 MEMORY.md。

### 修改既有 memory

不動 MEMORY.md / cluster file（除非新內容顯著擴增 cluster description 範圍才 update cluster frontmatter）。

## Don't write

- Code patterns / file paths / git history（從 current state derive 即可）
- **ccp-* 函式內具體配置**（model 對應、env var 值、subagent routing）—— 這些 read `shell/ccp-functions.sh` 就有
- Ephemeral task / 當前對話 state
- 重複 CLAUDE.md / docs/ 已記錄的內容（用引用代替：「詳見 `docs/X.md`」）

## 完成後

簡短報告：寫了 / 改了哪個檔案、type、是否更新 MEMORY.md。不要重複 frontmatter 內容。
