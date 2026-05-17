---
description: Dual-model PR review (Opus + Codex) with Traditional Chinese comparison report.
argument-hint: "<PR-URL>"
---

## Prerequisites

This is a framework template, not drop-in. To use this command you need:

- **Codex CLI plugin** — for dual-model review (Opus + Codex). Look up upstream marketplace URL via `ls ~/.claude/plugins/cache/` or `cat ~/.claude/plugins/marketplaces/*/marketplace.json | grep -A5 codex`. **Verify the upstream install command before relying on this**; placeholder install path: `/plugin install codex@claude-plugins-official`.
- For Bitbucket PR support: `bitbucket-pr-review` skill from author's private repo (NOT in this public release; reader can adapt to GitHub PR URL only if Codex CLI plugin alone is desired)

**This is a framework, not a drop-in tool.**

# PR Review — Dual-Model Comparison Workflow

Orchestrate a dual-model code review (Opus + Codex) for a pull request and produce a Traditional Chinese comparison report.

**Scope**: Workflow orchestration only. Review criteria and checklists are the responsibility of each reviewer agent — this command does not define what to review, only how to run and compare.

## Input

- `/pr-review <number>` — PR number in current repo
- `/pr-review <GitHub URL>` — `https://github.com/owner/repo/pull/123`
- `/pr-review <Bitbucket URL>` — `https://bitbucket.org/workspace/repo/pull-requests/123`

## Step 1: Parse Input & Detect Platform

| Input                                            | Platform    | Extract                                    |
| ------------------------------------------------ | ----------- | ------------------------------------------ |
| `github.com/owner/repo/pull/123`                 | GitHub      | owner, repo, PR number                     |
| `bitbucket.org/workspace/repo/pull-requests/123` | Bitbucket   | workspace, repo, PR ID                     |
| Plain number (e.g. `278`)                        | Auto-detect | Infer from `git remote -v` of current repo |

## Step 2: Fetch PR Data

### GitHub

```bash
gh pr view <number> --json title,body,state,baseRefName,headRefName,author,additions,deletions,changedFiles,commits
gh pr diff <number>
gh pr view <number> --json files --jq '.files[].path'
```

### Bitbucket

No native CLI equivalent to `gh pr view`. Follow the authentication and API workflow defined in `~/.claude/skills/bitbucket-pr-review/SKILL.md` (private skill, not in this public release; reader needs to adapt for GitHub PR URL only or build their own Bitbucket workflow) to fetch the same fields (title, body, base/head ref, author, additions/deletions, changed files, diff).

Key reference:

- Auth: grep token from `~/.zshrc`, use `bb_api.sh` helper
- Endpoints: PR details, diffstat, diff, comments
- Equivalent of `gh pr view` for GitHub: combine `pullrequests/<id>` + `pullrequests/<id>/diff` API calls

## Step 2.6: Detect Spec / Plan Docs in PR

PRs produced via Superpowers workflow (brainstorming → writing-plans → executing-plans) often include a markdown spec/plan/design doc that states intent, scope, and explicit non-goals. Reviewers should use these as ground truth for "what this PR is supposed to do" before flagging "missing X" or "should also handle Y" — the spec may explicitly rule something out of scope.

### Detection heuristic

From the PR's changed-file list, flag a `.md` file as a spec if ANY of:

- Path contains `/specs/`, `/plans/`, `/brainstorm/`, `/design/`, `/proposals/`, `.claude/plans/`
- Filename matches `*-spec.md`, `*-plan.md`, `*-design.md`, `*-brainstorm.md`, `*-requirements.md`, `*-proposal.md`
- Filename looks like `YYYY-MM-DD-*.md` (common Superpowers plan naming)
- File starts with frontmatter containing `type: plan` / `type: spec` / `type: design` / `phase:` / `goals:` / `non_goals:`

Explicitly NOT specs: `README.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, `LICENSE.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`.

### What to do with detected specs

1. Read the full content of each detected spec (use `gh api` or local `git show` depending on platform)
2. If total spec content > ~8000 tokens, summarize before passing to reviewers (keep goals, non-goals, decisions, constraints — drop prose)
3. Pass to both Opus and Codex in Step 3
4. Surface in Step 5 report under 「Spec 依據」section

If no spec is detected, note it in the report ("此 PR 未附 spec／plan 文件") and proceed normally — absence of a spec is not itself a problem.

## Step 2.7: Prepare Search Path (before review)

To enable search-before-flag discipline in both reviewers, the default search tool is **Grep** (across the repo working tree).

If a semantic-search MCP happens to be active in the current session AND the repo is indexed, the reviewer agents may use it as a faster alternative — but Grep is the default and always works.

Record the chosen search tool path — you will include it in both reviewer prompts below so they know what's available.

## Step 3: Dispatch Dual Reviews (Parallel)

Launch both reviews **simultaneously** using parallel Agent calls.

### Opus Review (Multi-Agent Routing)

Do NOT hard-code `code-reviewer`. Route to language-specific + domain-specific reviewers based on PR contents. All dispatched reviewers run **in parallel**. Cost is not a concern for review quality at this scope.

#### Primary Language Reviewer (pick exactly one)

Detect dominant source-code language by counting changed lines per extension (exclude tests, configs, docs, lockfiles):

| Extensions                                             | Reviewer                           |
| ------------------------------------------------------ | ---------------------------------- |
| `.ts`, `.tsx`, `.js`, `.jsx`, `.mjs`, `.cjs`           | `typescript-reviewer`              |
| `.py`                                                  | `python-reviewer`                  |
| `.go`                                                  | `go-reviewer`                      |
| `.rs`                                                  | `rust-reviewer`                    |
| `.java`                                                | `java-reviewer`                    |
| `.kt`, `.kts`                                          | `kotlin-reviewer`                  |
| `.c`, `.cc`, `.cpp`, `.h`, `.hpp`                      | `cpp-reviewer`                     |
| `.dart`                                                | `flutter-reviewer`                 |
| mixed with no dominant (>40%) language / none of above | `code-reviewer` (generic fallback) |

"Dominant" = language with the most changed lines among source files. If the top language has < 40% of changes, or the PR is cross-language-heavy, fall back to `code-reviewer`.

#### Parallel Domain Reviewers (dispatch in addition when triggered)

Run alongside the primary reviewer when the PR touches the corresponding surface:

- **`security-reviewer`** — trigger if ANY of:
  - Paths under `auth/`, `security/`, `crypto/`, `middleware/auth*`, `middleware/csrf*`, `oauth/`
  - New env reads matching `/API_KEY|TOKEN|SECRET|PASSWORD|CREDENTIAL/`
  - New code handling request body / query / cookies / form data / file uploads
  - Changes to session / cookie / JWT logic
  - Changes to permission / role / RBAC code
- **`database-reviewer`** — trigger if ANY of:
  - Paths under `migrations/`, `db/migrations/`, `prisma/migrations/`, `supabase/migrations/`
  - Changes to `schema.prisma`, `*.sql`, `schema.sql`, `*.dbml`
  - SQL strings in code touching DDL (CREATE/ALTER/DROP TABLE, CREATE INDEX)
  - Changes to ORM model files (`models/*.py`, `*.model.ts`, `entity.ts`) with structural diffs

Do NOT run domain reviewers on every PR — only when triggered. Purely frontend-styling PRs do not need security-reviewer.

#### Shared prompt to every dispatched Opus reviewer

Each dispatched reviewer (primary + any domain reviewers) receives the same base prompt, differing only in whose agent rules apply:

- Project context (framework, SSR environment, etc.)
- Changed files with their purpose (highlight which files triggered this reviewer's specialty)
- Full diff
- **Spec / plan content from Step 2.6** (verbatim if ≤ 8k tokens, summarised otherwise); if absent, state "no spec attached"
- Available search tool per Step 2.7 (Grep by default; semantic-search MCP if Step 2.7 detected one)
- Request severity ratings: CRITICAL, HIGH, MEDIUM, LOW
- Reminder: "Apply your Context-Gathering Discipline — for any 'missing X / should handle Y' finding, search the codebase first using the tool noted above AND check the attached spec for explicit scope/non-goals before flagging. If spec marks something as out-of-scope, do not flag. Attach search-proof AND spec-reference when relevant. Strict-liability defects (hardcoded secrets, SQL injection via concat, eval with user input, innerHTML with user input) may be flagged without search."

#### Consolidating Opus-side findings

Each reviewer returns its own finding list. Merge into a single "Opus findings" bucket for Step 4 comparison:

- **Dedup**: if two reviewers flag the same file:line with the same concern, keep one entry, note both reviewer sources (e.g. `[typescript-reviewer + security-reviewer]`)
- **Severity on dedup**: use the HIGHER of the two
- **Disagreement on severity**: keep higher severity; note the disagreement in 備註
- **Source tag**: every consolidated Opus finding must carry its source reviewer tag (e.g. `[typescript-reviewer]`, `[security-reviewer]`, `[database-reviewer]`) in the final report, so user sees which specialty caught which issue
- **No finding dropped** in consolidation — only deduplicated

### Codex Review

**Preferred path** — use `codex review` built-in when the PR branch can be checked out locally.

#### Pre-flight (critical — both of these have bitten us)

1. **Freshen the base ref before diffing.** `codex review --base origin/<base>` diffs against whatever `origin/<base>` resolves to in your local refs. If you only fetched the PR branch earlier, `origin/<base>` may still point at an older commit — Codex will then diff against unrelated changes and return findings for entirely different PRs. Always `git fetch origin <base-branch>` right before the review, and verify the tip matches the PR metadata's `destination.commit.hash` (Bitbucket) / `base.sha` (GitHub) via `git rev-parse origin/<base>`.
2. **Check out the PR branch.** `codex review` reviews the current HEAD. If HEAD is a different branch, the diff is wrong. Record the original branch so you can restore afterwards:
   ```bash
   ORIG_BRANCH=$(git rev-parse --abbrev-ref HEAD)

   # For same-repo PRs (head branch lives in origin):
   git fetch origin <head-branch>
   git checkout "origin/<head-branch>" --quiet   # detached HEAD is fine

   # For GitHub fork PRs (head branch lives in the fork's repo, ref is pull/<n>/head):
   # use gh pr checkout instead — it handles the fork remote automatically
   gh pr checkout <number> --detach
   ```
   If both fail (private fork without access, no gh auth, etc.) → see Fallback below.

#### Run

```bash
# Direct (interactive-capable):
codex review --base origin/<base-branch>

# Or via the plugin companion (non-interactive, writes to a log file).
# Resolve the path dynamically — codex plugin version changes over time:
CODEX_PLUGIN_DIR=$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/ 2>/dev/null | sort -V | tail -1)
node "${CODEX_PLUGIN_DIR}scripts/codex-companion.mjs" \
  review --wait --base origin/<base-branch> --scope branch
```

- `codex review` already carries its own review prompt contract and output schema (look up `${CODEX_PLUGIN_DIR}schemas/review-output.schema.json` — fields: `verdict`, `findings[]`, `next_steps`). **Do NOT author a long custom prompt.** If you want to nudge focus, append one short sentence of `[focus text]` — that's all.
- **When checkout is possible, do NOT route Codex review through `codex:codex-rescue` or `codex task`.** Those modes wrap Codex in a generic task/rescue prompt and lose the built-in review contract, forcing you to re-author the contract yourself in XML tags — longer, more fragile, and no better signal than calling `codex review` directly. (Exception: see Fallback below when checkout is genuinely impossible.)

#### Restore

```bash
git checkout "$ORIG_BRANCH" --quiet
```

**Fallback** — only when local checkout genuinely failed (e.g. `git fetch origin <head-branch>` returned access-denied for a fork without permissions, `gh pr checkout` errored, or running in a detached environment without git):

- Use `codex:codex-rescue` subagent with the diff as prompt (this is the one place where the local-checkout prohibition above is lifted)
- Request the same severity scale (CRITICAL/HIGH/MEDIUM/LOW) for comparability with the Opus side
- Expect lower signal quality than the built-in review path

### Codex: intentionally kept diff-only

Do **NOT** inject the search-before-flag rule into Codex's prompt. Codex's value in this dual-model workflow is its **no-context perspective** — it catches things by reading only the diff and applying general intuition, the way a reviewer sees a PR email without checking out the repo.

The context-aware verification happens in Step 4 below (Opus re-reads each Codex finding using Grep, optionally augmented by a semantic-search MCP if available). This preserves Codex's first-pass signal while letting Opus act as the fact-checker.

Do send Codex:

- Same severity scale (CRITICAL/HIGH/MEDIUM/LOW) for comparability
- Structured output schema so findings can be iterated individually in Step 4
- Project background (framework, language) so it isn't flying blind — but NOT existing-code patterns
- **Spec / plan content from Step 2.6** if detected — this helps Codex distinguish "missing feature" from "out of scope per spec". Without spec context, Codex would flag many non-issues.

## Step 4: Opus Verification Pass on Codex Findings

**Trigger**: only after BOTH Step 3 reviews (Opus first-pass + Codex) have returned. This step consumes Codex's findings as input, so Codex must finish; Opus first-pass is also required because already-confirmed-by-Opus findings are skipped (see Scope below).

Run a **fact-checking pass** using the `code-reviewer` agent again (Opus), feeding it Codex's raw findings one by one and asking it to verify each against the codebase using Grep (or a semantic-search MCP if available per Step 2.7).

### Scope

- Verify **every non-strict-liability finding** from Codex
- Skip strict-liability findings (hardcoded secrets, SQL via concat, eval/innerHTML on user input, plaintext password compare) — these stand without verification
- Also skip findings already confirmed by Opus's first-pass (those are consensus)

### Verification Prompt to Opus (per Codex finding)

```
Codex flagged: [severity] [title] at [file:line_start-line_end]
Body: [finding body]

Spec / plan attached (if any): [content or "none"]

Task: verify whether this is a real gap.
1. If spec is attached and explicitly marks this concern as out-of-scope or non-goal → output "OUT_OF_SCOPE" with the spec quote.
2. Use Grep to search for related patterns in this codebase (e.g. how similar concerns are handled elsewhere, upstream middleware, existing utilities, test coverage). If a semantic-search MCP is available per Step 2.7, you may use it in addition.
3. Output one of:
   - "CONFIRMED": search found no coverage; Codex's concern is valid. Attach search-proof (query + what you found).
   - "REFUTED": search found the concern is already handled elsewhere at file:line. Attach the proof.
   - "PARTIAL": covered in some paths but not the path Codex identified. Describe the gap precisely.
   - "OUT_OF_SCOPE": spec explicitly excludes this. Quote the spec.
Do NOT delete the finding — the final report will show your verdict alongside Codex's original.
```

### Output per finding

```typescript
{
  codex_original: { severity, title, body, file, line_start, line_end, confidence },
  opus_verdict: "CONFIRMED" | "REFUTED" | "PARTIAL",
  opus_evidence: string,  // what was searched, what was found (file:line)
  opus_searched_via: "Grep" | "Grep+semantic-search"
}
```

### Guardrails (CRITICAL — do not violate)

- **Never drop a Codex finding** based on verification result. Every Codex finding appears in the final report with its verdict attached.
- **REFUTED does not mean wrong** — it means "context suggests the concern is already handled." User may still disagree with Opus's evidence. Keep the original so user can judge.
- **Verification is one pass only** — don't loop Opus forever. If Opus can't reach a verdict, mark "INCONCLUSIVE" and move on.

## Step 5: Compile Comparison Report

**Reporting principle — transparency over filtering.** Codex's raw perspective AND Opus's verification appear side-by-side. Refuted findings keep their own row in 發現總覽 and their own block in 參考用. (Authoritative rule: see Step 4 Guardrails.)

### Report Structure

````markdown
# PR #<number> Code Review 比較報告

**PR**: [owner/repo#number](URL)
**標題**: ...
**作者**: ...
**分支**: `head` → `base`
**變更**: N 檔案, +A / -D
**審查日期**: YYYY-MM-DD
**審查工具**: Opus (code-reviewer agent, context-aware) + Codex (diff-only, fresh eyes) + Opus verification pass

---

## Spec 依據

- 若偵測到 spec / plan 檔：列出檔名 + 關鍵 goals / non-goals / decisions 摘要
- 若未偵測到：註明「此 PR 未附 spec／plan 文件，按一般 PR 流程 review」

## 變更概要

Per-file table: filename, change type, description

## 發現總覽

**Row ordering**: order rows by 最終建議 group — Must Fix → Should Fix → Nice to Have → 參考用. Within a group, keep finding number order. Severity (CRITICAL/HIGH/MEDIUM/LOW) stays visible inside the cells but does NOT drive ordering — the user reads this report to decide what to fix first, so actionable priority (the user's final call) outranks raw severity. Renumber findings after sorting so that #1, #2, #3 ... read top-to-bottom in priority order.

| #   | 問題 | Opus first-pass | Codex first-pass | Opus 對 Codex 複查                                           | 最終建議     |
| --- | ---- | --------------- | ---------------- | ------------------------------------------------------------ | ------------ |
| 1   | ...  | CRITICAL        | CRITICAL         | CONSENSUS                                                    | Must Fix     |
| 2   | ...  | HIGH            | (not flagged)    | —                                                            | Must Fix     |
| 3   | ...  | (not flagged)   | MEDIUM           | CONFIRMED                                                    | Should Fix   |
| 4   | ...  | MEDIUM          | (not flagged)    | —                                                            | Nice to Have |
| 5   | ...  | (not flagged)   | HIGH             | REFUTED (handled at path/to/file.ts:42)                      | 參考用       |
| 6   | ...  | (not flagged)   | HIGH             | OUT_OF_SCOPE (spec 2026-04-21-plan.md line 34 標為 non-goal) | 參考用       |

### Inline Comments per Finding（直接複製貼到 PR review）

For each row in 發現總覽, emit a copy-paste-ready inline-comment block. The user takes these blocks straight into GitHub / Bitbucket PR inline review without rewording — this is what makes a long report actionable. Order the blocks by 最終建議 group, same order as 發現總覽.

Each block:

- **Heading**: `#### #N <短標題>` — N matches the finding number in 發現總覽
- **File**: repo-relative path
- **Line**: single line or range (e.g. `162-166`). If a single finding spans multiple file locations (e.g. the same XSS pattern in render.ts and modal.ts), split into separate blocks with `#1a` / `#1b` suffixes — a PR inline comment pins to one file:line, so one block per pin point.
- **Comment**: a fenced code block (` ``` `) containing the ready-to-paste text. 繁中. Include:
  - severity tag prefix in brackets, e.g. `[CRITICAL — strict-liability XSS]` / `[Must Fix — 違反 spec]` / `[Should Fix — defence-in-depth]` / `[Nice to Have]`
  - 1–3 lines describing the problem with concrete code reference
  - spec line citation if applicable (`Spec line NNN: "..."`)
  - 「建議」段落 with the minimum viable fix as a short code snippet

範例：

```markdown
#### #1a 商品圖 altText 未 escape → strict-liability XSS

**File**: `widgets/apps/loyalty-app-blocks/src/js/reward-products/render.ts`
**Line**: 255-257

**Comment**:
```
````

[CRITICAL — strict-liability XSS]
`product.image.altText` 沒過 escapeHtml，直接插進 innerHTML 組出的 <img alt='...'>。
同 template literal 內 src 已 escape，alt 漏掉。
攻擊路徑：merchant 在 Shopify product image alt 設 ' onmouseover='alert(1)
就能 break 出 attribute 注 event handler。

建議：
alt='${escapeHtml(product.image.altText)}'

```

```

OUT_OF_SCOPE / REFUTED 的 Codex finding 也出 inline-comment block（屬於「參考用」群組），但 comment 內容開頭明確標示「不是 PR 缺陷」並說明原因（spec 引用 / Opus 找到的現有處理位置），讓 PR 作者一眼分辨優先級。

排除：純架構/設計類觀察（無具體 file:line）不做 inline comment，這類放在「審查工具比較」或「總結」段落即可。

### Opus 原始 findings (first-pass, context-aware)

Each finding (verbatim from Opus):

- Severity
- File:line_start-line_end
- Problem description
- Impact
- Suggested fix
- Search-proof (from code-reviewer agent's Context-Gathering Discipline)

### Codex 原始 findings (first-pass, diff-only)

Each finding (verbatim from Codex, do NOT edit or filter):

- Severity
- File:line_start-line_end
- Problem description
- Impact
- Suggested fix
- `confidence` from Codex schema

### Opus 對 Codex 的複查結果

For each Codex non-strict-liability finding, show:

| Codex # | Codex title                     | Verdict                                | Opus evidence                                                                                                          | 備註                                                             |
| ------- | ------------------------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| 2       | `path/x.ts:88 lacks null check` | CONFIRMED                              | searched "null handling for X" via Grep; only covered in `src/middleware/auth.ts:42`, doesn't cover job path | 採納                                                             |
| 3       | `missing zod validation`        | REFUTED                                | searched "zod schema request body"; found `src/routes/validate.ts:12` wraps all handlers in this tree                  | Codex 未看到上游 validator，但 Codex 的 concern 方向合理，列參考 |
| 5       | `hardcoded secret in test`      | SKIPPED (strict-liability passthrough) | —                                                                                                                      | 不經驗證直接採納                                                 |

Strict-liability findings from Codex pass through without verification and appear under 「採納」.

## Action Items

Weighted by verification verdict, but **all Codex findings are still shown below** — refuted ones in 「參考用」so user can override.

### Must Fix（合併前必修）

- Consensus findings (both Opus + Codex flagged, not refuted)
- CRITICAL severity from either reviewer (strict-liability always here)
- Opus CONFIRMED verifications of Codex findings

### Should Fix（強烈建議）

- Opus first-pass HIGH findings
- Opus PARTIAL verifications of Codex (some paths covered, gap remains)
- Codex HIGH that Opus couldn't verify (INCONCLUSIVE)

### Nice to Have（可選優化）

- MEDIUM / LOW findings from either reviewer

### 參考用 / Codex 第一眼 signal（Opus 判斷為 REFUTED 或 OUT_OF_SCOPE）

- List every Codex finding marked REFUTED or OUT_OF_SCOPE, verbatim
- REFUTED format: `Codex 擔心 X → Opus 於 file:line 找到現有處理 Y → 使用者自行判斷是否採納 Codex 的顧慮`
- OUT_OF_SCOPE format: `Codex 擔心 X → spec 明確標為 non-goal（引用 spec 段落）→ 如 scope 應擴大，使用者自行決定`

## 審查工具比較 (qualitative)

- Opus 視角: context-aware, 善於找出「跨檔缺失」
- Codex 視角: diff-only, 善於從 diff 本身語感找 smell
- 兩者重疊率: X% (consensus 筆數 / 總 finding 數)
- Opus 複查 Codex 的結果分佈: CONFIRMED N, REFUTED M, PARTIAL K, INCONCLUSIVE L

```

### Report Generation Rules

- **Never drop a Codex finding** — every first-pass finding appears in both 「Codex 原始 findings」and 「發現總覽」table
- **Never fabricate verification evidence** — if Opus couldn't search (MCP down + Grep didn't match), mark INCONCLUSIVE
- **Disagreement on severity** — if Opus and Codex rate the same issue differently, show both in the 發現總覽 table
- **Strict-liability findings from Codex** — list under a clear 「Codex strict-liability 採納」note so user sees they skipped verification by design
- **Final suggestion column** must be one of: Must Fix / Should Fix / Nice to Have / 參考用
- **Inline Comments per Finding section is mandatory** — every actionable finding (including OUT_OF_SCOPE / REFUTED ones tagged as 「參考用 / not a PR defect」) gets a copy-paste block with File / Line / Comment. Each Comment must be self-contained — do NOT write "see finding #3 above"; PR authors read inline comments without cross-referencing the main report. Multi-location findings split into `#Na` / `#Nb` per file:line pin.

## Step 6: Output

- Write to: `pr-<number>-review.md` in the **PR's repo root** (i.e. the cwd you checked out in Step 3). If reviewing a Bitbucket PR via API only (no local checkout), write to the current cwd and note this in the report header.
- Language: 繁體中文
- Do NOT git-add or commit

## Error Handling

- If either reviewer fails or times out, proceed with the available result and note the gap in the report
- If PR data fetch fails (e.g. private repo via MCP), fall back to `gh` CLI or local git
- If Bitbucket API returns 401, direct user to regenerate token per `bitbucket-pr-review` skill instructions
```
