Sync Claude Code config across computers using a Git repo at `~/.claude/`.

## Prerequisites

This is a framework template, not drop-in. To use this command you need:

- A private Git repo for `~/.claude/` (e.g., `{{YOUR_GITHUB}}/claude-config`)
- `~/.claude/cross-machine/` directory with task system convention (self-build)
- This command assumes a specific cross-machine workflow; adapt to your sync setup (works with any git-tracked `~/.claude/`)

**This is a framework, not a drop-in tool.**

## Push mode (default)

0. **Auto-push sessions first** — run `~/.claude/hooks/session-backup.sh` to push `~/.claude/projects/` (independent git repo pointing to `claude-session-backups`). This is non-interactive: always run before touching the main `~/.claude` repo. If the script fails (network, merge conflict), log the error and continue with main sync — don't block, don't ask the user. The main point: the user should never need to manually run `session-backup.sh` or be asked about unpushed sessions; `/sync` handles it.
1. Run `cd ~/.claude && git status` to see what changed
2. Run `git add -A && git diff --cached --stat` to preview staged changes
3. **Pre-commit task check** — decide whether what was just done needs a cross-machine task. A task is needed when the change **won't translate cleanly via file sync alone**. Review both the staged diff and the actions taken in this session.

   **High-confidence triggers** (just did one of these → ask the user before committing):
   - Ran an interactive install command: `/install-plugin`, `brew install`, `uv tool install`, `apt install`, `winget install`, `npm install -g`
   - Ran an interactive auth command: `gh auth login`, `aws configure`, paste-an-API-key flows
   - Created or modified a scheduled task: `crontab`, `launchctl`, `schtasks`, `systemd timer`
   - Edited `~/.claude/settings.local.json` (gitignored — never syncs)
   - Created or modified a shell script under `~/.claude/hooks/` or similar

   **Medium-confidence triggers** (consider asking):
   - Wrote a script containing absolute paths (`~/foo`, `/Users/...`, `C:\Users\...`)
   - Used an OS-specific tool (`say`, `osascript`, `notify-send`, `pwsh`, `taskkill`)
   - Manually told the user to do something outside Claude (e.g., "go to GitHub.com and create a repo")

   **Do NOT write a task** (only these — everything else leans toward writing):
   - Pure text config changes (`settings.json`, `CLAUDE.md`, files under `skills/`, `commands/`, `rules/`, `agents/`) — git syncs these directly
   - Anything inside `~/.claude/projects/`, `sessions/`, `cache/`, etc. — gitignored

   **Bias toward writing — don't pre-judge feasibility**:
   - Feasibility on the target machine is the **receiving Claude's** problem, not origin's. Origin's job is only to flag "this action could behave differently across machines."
   - Don't skip a task because the origin-side tool "is macOS-only" or "isn't available on Windows" — the target may have a counterpart (e.g. cmux → WMUX on Windows, `brew` → `winget` / `apt`, `launchctl` → Task Scheduler / systemd) and the target Claude can research local equivalents.
   - Describe the **origin intent** (what was done and why), not a prescription — let the target adapt, port, or decline.
   - Cost of a task the target decides to skip ≈ one markdown read. Cost of a missed task ≈ silent config drift the user has to discover and re-derive later. Bias heavily toward writing.

   **Also inspect the staged diff**:
   - If any new file appears in `cross-machine/pending/`, confirm the user wrote it intentionally (not an accidental stray file)
   - If `settings.local.json` is accidentally tracked, warn and unstage

   If nothing matches, proceed silently. If a task is needed, read `~/.claude/cross-machine/README.md` for the task file format and save as `pending/<short-name>.md` before committing.
4. If there are changes, commit with a Conventional Commits message (feat/fix/chore/docs/refactor) and push
5. If no changes, report "已是最新，無需同步" / "Already up to date, nothing to sync"
6. Report the sync result with the commit hash and push status

## Pull mode

Triggered when the user says "pull" or "拉":

1. Run `cd ~/.claude && git pull --rebase origin main`
2. After pull, scan for cross-machine tasks:
   - List files in `~/.claude/cross-machine/pending/` (ignore `.gitkeep`)
   - For each `.md` file, check if a same-named file exists in `~/.claude/cross-machine/done/`
   - If exists in `done/` → skip (already completed on this or another machine)
   - If not in `done/` → present as a new task with name and one-line goal extracted from the task file
3. If the user wants to execute a task:
   - Read the full task file (and `~/.claude/cross-machine/README.md` for execution guidance)
   - Adapt to the local OS, package manager, and conventions
   - Execute, verifying end state matches the Goal
   - Copy the task file to `~/.claude/cross-machine/done/<name>.md` with appended completion frontmatter (`completed_by`, `completed_at`) and an Execution summary section
4. If no new tasks, report "無新任務" / "No new tasks"

## Notes

- The Git repo at `~/.claude/` is the single source of truth for cross-machine sync
- `.gitignore` excludes session data, plugins, cache, and machine-specific files
- Never commit `settings.local.json` or files under `projects/`, `sessions/`, `plugins/`
- For details on writing and executing tasks, read `~/.claude/cross-machine/README.md`
