# Goal Plan: Publish 13 skills/commands to public gggodlin/claude-skills repo

Generated: 2026-05-17 (resume-session 後)
Task: Run 19-task skill publish plan at `docs/plans/2026-05-17-skill-publish.md` end-to-end
Turn budget: 50 (hard cap 50)

## Plan

按 `docs/plans/2026-05-17-skill-publish.md` 依序執行；可 parallel dispatch independent sanitize tasks（Task 2-14 多數彼此 independent）以節省 turn budget。

| Phase | Tasks | Action | Verification (machine-checkable) |
|---|---|---|---|
| 1. Skeleton | Task 1 | git init done → 寫 0BSD LICENSE + mkdir skills/ + commands/ → commit | `head -1 LICENSE` = `BSD Zero Clause License`；`git log --oneline` 含 skeleton commit |
| 2. Drop-in sanitize | Task 2-5 | Dispatch general-purpose subagent × 4，AI semantic sanitize → diff + sensitive grep → commit per task | `grep -rE "philip@\|linhancheng\|akohub\|林瀚埕\|qwe70301\|/Users/linhancheng/" skills/{review-html,review-zh,ai-image-gen}/ commands/changelog-check.md` 全 0 hit；無 `## Prerequisites` section |
| 3. Framework sanitize | Task 6-14 | Dispatch general-purpose subagent × 9，Framework Template prompt（含 PREREQUISITES_LIST） | 同上 grep 全 0；每檔 `grep "## Prerequisites"` 有 hit |
| 4. Wrap-up docs | Task 15-17 | 寫 plugin.json / CONTEXT.md ≤ 50 lines / README.md → commit | `python3 -m json.tool < plugin.json` exit 0；`wc -l CONTEXT.md` ≤ 60；README 含 "Drop-in tools" 字串 |
| 5. Dogfood | Task 18 | Clean-state install test 到 /tmp/claude-skills-dogfood/，sensitive grep + Prerequisites check → cleanup | Dogfood sandbox sensitive grep 0 hit；4 Drop-in 無 Prerequisites、9 Framework 全有 |
| 6. Publish | Task 19 | `gh repo create gggodlin/claude-skills --public --push` → repo topics → optional npx install test | `gh repo view gggodlin/claude-skills --json url` 回 valid URL；`git log --oneline` 含 main branch push 證據 |

每個 sanitize task 內部結構：inspect → mkdir → dispatch subagent → diff/grep verify → manual fix if flagged → commit。

## Pre-flight checklist

✅ **`gh` CLI auth**: Logged in as `GGGODLIN`, token scope includes `repo` + `workflow` + `gist` + `read:org`
✅ **Local repo state**: `~/Desktop/projects/claude-skills/` git init done, `.gitignore` = `docs/`, 0 commits
✅ **Source files (13)**: 全在 `~/.claude/skills/{review-html,review-zh,ai-image-gen,research-before-answer,video-research,figma-context-budget}/` + `~/.claude/commands/{changelog-check,wiki-promote,memory-audit,memory-checkpoint,sync-claude-setting-cross-platform,pr-review,devfleet}.md`
✅ **GitHub namespace**: `gggodlin/claude-skills` repo **不存在**，Task 19 安全 create（無 collision）
✅ **`superpowers:subagent-driven-development` skill**: Available
✅ **`general-purpose` subagent**: Available
✅ **`./PLAN-goal.md`**: No prior file（this file 是本次寫的，OK）
✅ **Plan + spec 檔**: `docs/plans/2026-05-17-skill-publish.md` + `docs/specs/2026-05-17-skill-publish-design.md` 都在

**所有 prereq 都 ✅，可直接貼 condition 進 /goal 開跑。**

## Hand-off (post-goal manual steps)

1. **`~/.claude/` 改動 commit 到 `GGGODLIN/claude-config` private repo**（不屬本 plan 範圍）
   - 為什麼 Claude 做不到：本 goal scope 鎖在 `~/Desktop/projects/claude-skills/`，跨 repo commit 需要使用者主動
   - 改動內容：`grill-me-one-shot.md` rename + `plankton-code-quality/` 刪 + `prompt-optimize.md` 刪 + `INVENTORY.md` 50→49

2. **Optional X/Twitter announcement**（plan Task 19 Step 6 提到，非必要）
   - 為什麼 Claude 做不到：個人社群帳號發佈需 user 自己決定時機 + 內容

3. **Cross-reference 到私人 INVENTORY**（plan Task 19 Step 6 提到）
   - 為什麼 Claude 做不到：私人 claude-config repo 的 commit 不屬本 goal scope

4. **Verify Codex CLI plugin URL 跟 DevFleet MCP URL**（Task 13/14 內 Prerequisites placeholder）
   - 為什麼 Claude 做不到：goal 內 lookup 失敗會留 placeholder + 標 transcript；user 後續手動填或決定 drop
   - 確認命令：`cat skills/commands/pr-review.md | grep -A5 "## Prerequisites"` + `cat skills/commands/devfleet.md | grep -A5 "## Prerequisites"`

## Risk + fallback

| Risk | Fallback |
|---|---|
| Sanitize subagent 漏 sensitive token（如新 personal email 沒在 replacement table） | Sensitive grep verify 會抓到 → main agent re-dispatch subagent 補修 |
| Task 13 Codex CLI plugin URL 查不到 | Prerequisites 留 placeholder 加註「verify before use」+ transcript surface |
| Task 14 DevFleet MCP URL 查不到（是 private） | Plan 已含 fallback：drop devfleet command 從 plugin.json + README 移除該 row |
| `gh repo create` fail（rate limit / network） | Retry 同 root cause 2 次後 emit PENDING_USER_ACTION；user 手動跑 `gh repo create` 後 re-run goal 從 Task 19 |
| Dogfood test 抓到 sensitive leak | Fix in source repo → re-commit → re-run dogfood（plan Task 18 Step 9 已含） |
| Turn cap 50 不夠 | Optimize：Task 2-14 parallel dispatch（13 個 sanitize 同時跑用 1-3 turns batch instead of 4×13=52）；若仍超 cap → surface 進度 + 待 user 開新 goal 繼續 |

## Goal condition (reference copy)

```
完成 ~/Desktop/projects/claude-skills/docs/plans/2026-05-17-skill-publish.md 的 19-task skill publish plan 至 transcript 包含全部以下證據後達成：

先讀 ./PLAN-goal.md 的 ## Plan section 了解 phase 結構與 hand-off。Invoke `superpowers:subagent-driven-development` skill 後按 docs/plans/2026-05-17-skill-publish.md 依序執行。可以 parallel dispatch independent sanitize tasks（Task 2-14 多數彼此 independent）來節省 turn budget；每完成一個 sanitize task → sensitive grep verify → commit。

Done evidence（全部要 transcript print 出來）：

1. Repo 結構：
   - `cd ~/Desktop/projects/claude-skills && git log --oneline | wc -l` 輸出 ≥ 15
   - `ls ~/Desktop/projects/claude-skills/skills/` 列 6 目錄：review-html / review-zh / ai-image-gen / research-before-answer / video-research / figma-context-budget
   - `ls ~/Desktop/projects/claude-skills/commands/` 列 7 檔：changelog-check / wiki-promote / memory-audit / memory-checkpoint / sync-claude-setting-cross-platform / pr-review / devfleet（.md）

2. LICENSE：`head -1 ~/Desktop/projects/claude-skills/LICENSE` 輸出 `BSD Zero Clause License`

3. plugin.json 合法：`python3 -m json.tool < ~/Desktop/projects/claude-skills/plugin.json` 跑得過 + 顯示 6 skills + 7 commands

4. Docs：
   - `grep "Drop-in tools" ~/Desktop/projects/claude-skills/README.md` 有 hit
   - `wc -l ~/Desktop/projects/claude-skills/CONTEXT.md` ≤ 60

5. Sensitive grep 0 hit：`grep -rE "philip@|linhancheng|akohub|林瀚埕|qwe70301|/Users/linhancheng/" ~/Desktop/projects/claude-skills/skills/ ~/Desktop/projects/claude-skills/commands/` 空輸出（exit 1）

6. Prerequisites classification：
   - 4 Drop-in 無 Prerequisites：`grep -l "## Prerequisites" ~/Desktop/projects/claude-skills/skills/review-html/SKILL.md ~/Desktop/projects/claude-skills/skills/review-zh/SKILL.md ~/Desktop/projects/claude-skills/skills/ai-image-gen/SKILL.md ~/Desktop/projects/claude-skills/commands/changelog-check.md 2>/dev/null` 空輸出
   - 9 Framework 全有：跑迴圈檢查 9 檔，每檔 `grep "## Prerequisites"` 有 hit

7. Dogfood test：Task 18 跑完，sandbox sensitive grep 0 hit 印在 transcript，cleanup done

8. GitHub published：`gh repo view gggodlin/claude-skills --json url` 輸出含 `github.com/gggodlin/claude-skills` 的 URL

Constraints:
- 不要動 ~/.claude/skills/ 跟 ~/.claude/commands/ source 檔（read-only reference；sanitize 寫到 ~/Desktop/projects/claude-skills/）
- 不要碰 ~/Desktop/projects/claude-skills/docs/specs/ 跟 docs/plans/（gitignored 個人空間，read-only reference）
- 不要洗掉現有 .gitignore 或 git init 狀態
- 不要因 turn budget 緊就改用 sed / 手寫 sanitize 取代 subagent dispatch（plan 內 AI semantic sanitize 是設計決策）
- 不要跳過 sensitive grep verify

Retry / unblock policy:
- 同 root cause 最多 retry 2 次；發現新 root cause 不算 retry
- 同 root cause retry 2 次仍 fail → emit "⏸ PENDING_USER_ACTION: <具體動作>" 等 user 完成
- 遇到 stale fact（Codex CLI plugin URL / DevFleet MCP URL 等）重新 invoke /research-before-answer，不要硬編
- Task 13 Codex URL / Task 14 DevFleet URL 查不到 → Prerequisites 留 placeholder + transcript surface（plan Task 14 已含 drop fallback）

External unblock 狀態:
- Phase 卡 sudo / GUI / 互動式 auth → transcript surface "⏸ PENDING_USER_ACTION: ..." 視為 pending
- 不算 phase failure、不消耗 budget

Sensitive credential 處理:
- 任何 token / API key 不要請 user 貼進 chat；走 hand-off 由 user 自設

or stop after 50 turns
```
