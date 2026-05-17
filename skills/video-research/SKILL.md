---
name: video-research
description: Use when user pastes a video URL (YouTube / Bilibili / other platform) and wants research, summary, discussion, or any analysis of the video. Pulls baseline summary via NotebookLM CLI then layers personal-context analysis using user's wiki and project memory. Single-paste UX — user pastes URL, this skill takes over end-to-end. Trigger phrases also include "研究這支影片"、"幫我看這個影片"、"這影片在講什麼"、"摘要一下這個".
---

## Prerequisites

This is a framework template, not drop-in. To use this skill you need to self-build:

- **NotebookLM CLI (`nlm`)** — install: https://github.com/199-biotechnologies/nlm-cli (third-party). Required for baseline video summary.
- **Personal wiki + memory infrastructure** — without `~/.claude/wiki/` entities and `~/.claude/projects/{{SCOPE}}/memory/` cluster, the "personal-context analysis" layer will analyze against empty content. You can either build the infra or use this skill purely for NotebookLM CLI baseline summary (skip personal-context section).
- **Output directory convention** — skill assumes `~/Desktop/projects/{{YOUR_IDEAS_DIR}}/ideas/` exists for output. Adapt to your own ideas directory or override at runtime.

**This is a framework, not a drop-in tool.**

# Video Research with Personal Context

固化的影片研究流程：**baseline 摘要由 NotebookLM 自動生成 + Claude 用使用者個人 context 做「對你的意義」分析**。

## When to Use

- 使用者貼影片 URL（YouTube / Bilibili / Vimeo / X / 其他）
- 使用者明確要求「研究這支影片」「摘要」「對我的意義」「值得看嗎」之類

## When NOT to Use

- 純問「Gemini 能吃影片嗎」「NotebookLM 多少錢」這類 meta 問題（用 research-before-answer）
- 使用者只要 raw transcript（用 yt-dlp 直接抓）

## Why This Pattern

2026-04-26 對「五大 Agent Memory 框架」影片做了三方對比實驗（baseline / NotebookLM v2 strict / subagent independent web research），結論：

1. **baseline 對影片忠實度最高**——5 entity 都 cover，UP 主想講什麼一目了然
2. **NotebookLM 衍生研究** 抓 arXiv 論文（學術深度）但漏 GitHub stars / 最新動態 / 社群口碑
3. **subagent independent research** 揭露現狀但跟使用者無關
4. **真正稀缺的是「對使用者的意義」**——這是 NotebookLM 跟 web search 都不做的層

詳見 `memory/reference_video_research_three_way_pattern.md`。

## Protocol（嚴格按順序執行）

### Step 1: URL Detection + Notebook 建立

```bash
# 從使用者訊息抽 URL（regex 抓 https?:// 開頭）
URL="<paste from user message>"
# 抽影片標題給 notebook 命名
TITLE=$(echo "$URL" | xargs -I{} python3 -c "
import urllib.parse as u
p = u.urlparse('{}')
print(f'{p.netloc} {p.path}'.replace('/', '_')[:60])
")
# 建 notebook
nlm notebook create "Video research: $TITLE $(date +%Y-%m-%d)"
# 從輸出抽 notebook id
```

### Step 2: 加 source + 等處理完

```bash
nlm source add <NB_ID> --url "$URL" --wait --wait-timeout 60
```

如果 type 不是 `youtube_video`（如 Bilibili 是 `web_page`），主動告訴使用者「這個平台 NotebookLM 只抓頁面 metadata，不看影片內容；摘要會基於 UP 主寫的描述」。

### Step 3: 跑 baseline Briefing Doc

```bash
nlm report create <NB_ID> --format "Briefing Doc" --language zh-Hant --confirm
# 從輸出抽 artifact_id
# poll status 直到 completed（通常 10-30 秒）
nlm studio status <NB_ID>
# 下載
DEST_DIR=~/Desktop/projects/{{YOUR_IDEAS_DIR}}/ideas/$(date +%Y-%m-%d)-video-$(echo "$TITLE" | tr -cd '[:alnum:]-' | head -c 30)
mkdir -p "$DEST_DIR"
nlm download report <NB_ID> --id <ART_ID> -o "$DEST_DIR/briefing.md"
```

### Step 4: 讀 baseline + cross-reference 個人 context

讀 `briefing.md` 之後，**主動翻**：
- `~/.claude/wiki/index.md` — 找跟主題相關的 wiki entity
- `memory/MEMORY.md` + `_index_*.md` cluster files — 找相關 user / feedback / project / reference 記憶（兩層 index 結構，2026-05-07 起；MEMORY.md 看到相關 cluster pointer → drill 進對應 `_index_<topic>.md` 拿 sub-entries → 再 Read 具體 topic file）
- 當下對話脈絡

**對話開場給使用者的「對你的意義」分析應該包含**（按需要選 3-5 項，不必全寫）：
- 跟使用者現有 project 的關聯
- 跟使用者 wiki entity 的對話
- 哲學或設計重疊度
- 可立即採用的 action item
- 哪些值得觀望 vs 馬上跳坑
- 跟使用者偏好的 tradeoff

**Fact-check 紀律（critical）**

寫進「對你的意義」分析的具體**技術名詞**（framework 名 / API name / feature name / 機制名 / 版本號 / 數據），baseline 提到的當作**未驗證 lead**，不直接寫進對比表 / 結論。要寫之前必須第一手驗證：

- 走 [`research-before-answer`](file://~/.claude/skills/research-before-answer/SKILL.md) skill 的並行查證
- Source 優先序：官方 docs → 官方 repo（程式碼層）→ 論文 → DeepWiki / 第三方文件
- 過不了第一手就**標「baseline 提到，未驗證」** 或乾脆不寫

特別針對 Bilibili / 字幕抓不到的平台：baseline 本質是 UP 主**描述頁**的二次轉述，技術細節錯誤率高（2026-04-26 真實踩坑：把「Letta Sleeptime 異步學習」當 V1 廢的舊機制寫進對比表，第一手查證才發現 sleeptime 跟 heartbeat 是兩個獨立概念，sleeptime V1 後保留並主推；詳見 `feedback_no_secondhand_research_as_fact.md`）。

**不要做的事**
- 不要平鋪直敘地把 baseline 摘要重複貼給使用者——他自己能讀
- 不要 cite NotebookLM 的內容當「我的研究」——baseline 是別人的工作
- 不要建議他「也應該跑 NotebookLM Create Your Own」「也應該跑 deep research」——這些已驗證對「對你的意義」場景過度設計
- 不要派 subagent 去做 web research 除非使用者明確要求「上網查最新狀況」
- 不要把 archive 內既有的 subagent independent research / 上次 session 結論當第一手事實——它們只是 lead，重新進入新 session 時也要重新驗證

### Step 5: Optional — 想擴展再加產物

只在使用者**明確要求**時才跑：
- 想看完整學術背景：`Create Your Own` + 嚴格 prompt（必須列名 + 等量篇幅 + 禁偷渡，詳見 `reference_nlm_deep_research_prompt_dependency.md`）
- 想看市場現狀：派 general-purpose subagent 做 web research（範本見 `reference_video_research_three_way_pattern.md`）
- 想生 mindmap / podcast / infographic：`nlm mindmap/audio/infographic create`

## Known Issues / Caveats

1. **Bilibili 字幕路徑死路**：yt-dlp 對 Bilibili HTTP 412（TLS fingerprint）；連帶 `--cookies-from-browser chrome` 都繞不過。Bilibili 走 `--url` 只抓 UP 主描述，不抓字幕。BibiGPT $19.80/月對中文 platforms 字幕有護城河。
2. **`nlm download audio` 在 0.5.30 壞掉**：lh3 redirect 後 404；audio 在 web 端可聽但 CLI 拿不到
3. **`nlm research start --auto-import` 沒生效**：要手動 `nlm research import`
4. **`nlm chat configure --response-length` 對 report create 沒生效**：只影響 chat query
5. **Deep research 預設不開**：除非使用者要求 + 必配 strict prompt，否則無關 source 會綁架輸出（infographic 跑題到「On-Premise vs. Cloud」是真實案例）

## Verification

跑完後告訴使用者：
- ✅ baseline 已下載到 `<path>`
- 主要 takeaway（跟個人 context 連結的版本）
- 待你決定的 follow-up question（不要逼他做決定）

## References

- `~/.claude/projects/{{SCOPE}}/memory/reference_video_research_three_way_pattern.md`
- `~/.claude/projects/{{SCOPE}}/memory/reference_notebooklm_mcp_cli_repo.md`
- `~/.claude/projects/{{SCOPE}}/memory/reference_nlm_deep_research_prompt_dependency.md`
- 實驗 archive: `~/Desktop/projects/{{YOUR_IDEAS_DIR}}/ideas/2026-04-26-video-research-experiment/`
