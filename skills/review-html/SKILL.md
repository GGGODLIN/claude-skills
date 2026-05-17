---
name: review-html
description: 把 spec.md / plan.md 渲染成繁中互動 HTML，方便 review 數百到上千行的長文件。讀懂 spec 段落語意、抽 semantic kind、用對應 component 重排，不是純 markdown 美化。Context-aware 翻譯：若 context 已有中文版 md 就用既有翻譯、否則 inline 翻。輸出單檔 HTML 到 ~/.claude/review-html/，自動開瀏覽器。
---

ultrathink

You are **review-html** — a skill that transforms long spec / plan / design markdown into an interactive HTML review explorer.

## When to use

Trigger:
- `/review-html <file>` — explicit
- `/review-html` (no args) — infer from recent context, or ask via AskUserQuestion

Use cases:
- 千行 task plan / spec / design doc，user 想方便 review
- 超過 200 行、結構複雜的 markdown

**Don't use as a markdown previewer.** This skill's value is **semantic restructure** (read paragraph meaning → render with appropriate component), not just CSS prettify. If user just wants prettified markdown, redirect them to a previewer (typora / markdown-preview-enhanced / VS Code preview).

## Phase 1: Resolve source

1. If `$ARGUMENTS` is a file path → Read it
2. If empty → AskUserQuestion: "Which spec / plan should I render? (paste path or describe which recent file)"
3. If file is too short (<100 lines) → confirm intent: short docs don't benefit much from this skill

## Phase 2: Translation source detection (Context-Aware)

**Don't hardcode "always run review-zh first" pipeline.** Check current context:

1. **Has user already requested / produced a translated version in this session?** Scan recent messages for indicators like 「翻譯」「review-zh」「中文版」.
2. **Does `~/.claude/review-zh/<source-stem>.zh.md` exist?** → use that translation if so.
3. **Is the source already mostly Traditional Chinese?** Quick scan first 50 lines, count CJK density.
4. **None of the above → inline translate during HTML rendering.**

This is a runtime decision per-invocation, not a fixed pipeline. The user explicitly noted: Claude should know whether translation has been done in this context, not be told to follow a rigid order.

## Phase 3: Semantic Kind Detection

Scan the spec content and identify which `kinds` appear. Common kinds:

**Type A — Task plan / executable**:
- `task` — Task N + Goal + Files + Steps block
- `phase` — Implementation Phases declaration
- `file-structure` — File create / modify list
- `step` — checkbox step (with optional code / commit)
- `commit` — git commit message

**Type B — Architecture spec**:
- `api-endpoint` — REST / GraphQL endpoint with method / path / body / response
- `data-model` — schema definition (entity + fields + relationships)
- `user-story` — As a X, I want Y, so that Z
- `persona` — user persona card
- `risk` — risk + impact + mitigation
- `decision` — decision + rationale (with optional alternatives considered)
- `acceptance-criteria` — testable AC list
- `goals` / `non-goals`
- `user-journey` — sequential steps with state
- `timeline` / `roadmap` — phased milestones

For each detected kind, note count + location. Anything that doesn't match → render as plain prose (don't force a template).

If kind detection is ambiguous (rare), default to prose rather than guessing wrong template.

## Phase 4: Render HTML

1. Read `STYLE_PRESETS.md` from this skill's directory for the complete HTML shell, CSS design tokens, and component templates
2. Build HTML inline by mapping each detected kind to its component template
3. Apply translation rules (see below) to user-facing content
4. Output to `~/.claude/review-html/<source-stem>-<YYYYMMDD-HHmmss>.html`

### Translation rules

**Translate to Traditional Chinese:**
- Task name / step title prose
- Goal / Architecture / description / rationale prose
- Section headings (Goal → 目標, Architecture → 架構, Files → 檔案, Steps → 步驟, etc.)
- UI shell label (Phases → 階段, Files → 檔案, lines → 行, click to expand → 點擊展開)
- File-impact group titles (Backend → 後端, Frontend create → 前端 (新增), Frontend modify → 前端 (修改))
- Phase descriptive name (Welcome banner → 歡迎橫幅, Page shell → 頁面骨架)
- Risk severity / impact wording (high → 高, medium → 中, low → 低)

**Keep original (do NOT translate):**
- Code blocks (any content inside ``` fences)
- Inline code (`function_name`, `file/path.ext`, `--flag`)
- Function names / API names / class names / variable names
- File paths / URLs
- Figma node ids (`8909:35741`)
- Commit messages content (`feat(scope): ...`)
- Brand / proper nouns (Polaris, Koa, Next.js, shadow-cljs, Shopify, React, GraphQL, Postgres, etc.)
- Acronyms (API, SDK, JWT, SSR, CSR, DAG, ER, AC, MVP)
- The label "Commit" itself (user prefers English for git verbs per CLAUDE.md)

**Translation tone:** 自然繁中、避免晶晶體、能翻則翻。判準：
- 有自然中文對應 → 翻（component → 元件、step → 步驟、create → 新增、modify → 修改、preflight → 預檢、verify → 驗證）
- 無自然對應或屬工具 / 協議 / 術語 → 留英文（API、prefetch、SSR、JWT、auth、handler、middleware）
- 模糊地帶不確定 → 保留原文，不要編造
- 技術專名留英文（Polaris、Koa 等品牌或框架名）

## Phase 5: Output & open

1. Write HTML to `~/.claude/review-html/<source-stem>-<YYYYMMDD-HHmmss>.html`
   - `<source-stem>` = source filename without extension (e.g. `v1-superpowers.md` → `v1-superpowers`)
   - `<timestamp>` = `date '+%Y%m%d-%H%M%S'`
2. Open in browser:
   - macOS: `open <path>`
   - Linux: `xdg-open <path>`
   - Windows: `start <path>`
3. Brief user: "Rendered to <path>. 看一下給反饋。"

## Component coverage hint

When rendering, prioritize components that match detected kinds (don't render empty sections):

**Type A spec** typical layout:
- Sticky sidebar (280px, left): 階段 / 檔案 toggle (Phase TOC + File Impact)
- Header card: extracted preamble metadata (目標 / 架構 / 技術堆疊 / 測試策略)
- Phase strip (horizontal pills, top of main)
- Stacked task cards (collapsible code, expanded step bodies, commit highlight)

**Type B spec** typical layout:
- Same sidebar shell, but TOC by section / kind
- Header card with goals / non-goals two-column
- Risk matrix block (if risks present)
- Decision log timeline (if decisions present)
- API endpoint cards (if APIs present)
- Mermaid embeds for ER / DAG / journey / state

If a spec mixes Type A and Type B, render both — they don't conflict.

## Caveats

- Don't strip user comments / inline annotations — preserve as quote blocks
- Don't "improve" spec content — render only, never edit semantics
- Source `.md` is untouched; HTML is derived view, not source of truth
- Unsupported kinds → fall back to prose (don't crash, don't fake structure)
- If spec is already Traditional Chinese, skip translation; still do semantic restructure

## Example invocation flow

```
User: /review-html ~/Desktop/work/{{YOUR_WORKSPACE}}/openspec/changes/archive/.../plans/v1-superpowers.md
(example from my work setup — adapt to yours)

Claude:
1. Read source (1419 lines, English task plan)
2. Context check: no review-zh副本 found, no prior translation in session → inline translate
3. Detect kinds: task × 19, phase × 10, file-structure × 1, commit × ~17 → Type A
4. Read STYLE_PRESETS.md for templates
5. Render HTML with inline ZH translation, save to ~/.claude/review-html/v1-superpowers-20260509-165430.html
6. open <path>
7. "Rendered to ~/.claude/review-html/v1-superpowers-20260509-165430.html. 看一下給反饋。"
```
