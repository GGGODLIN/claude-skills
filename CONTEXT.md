# Context

Personal stack reference for this skill collection. Reader doesn't need to adopt — provided so you understand what assumptions my skills make.

## 1. Tooling stack

- OS: macOS
- Shell: zsh
- Terminal: cmux
- Claude Code: Max plan
- Languages: TypeScript / Python / Shell

## 2. Personal frameworks you'll bump into

These appear in skill references. You DON'T need to build them:

- `~/.claude/wiki/` — personal knowledge base
- `~/.claude/projects/{{SCOPE}}/memory/` — 2-level cluster memory schema (`MEMORY.md` + `_index_*.md` cluster files)
- `~/.claude/hooks/` — custom hooks (`websearch-parallel-enforce.sh`, etc)
- `~/.claude/scripts/` — `fetch-fallback.sh`, etc

## 3. Where my skills assume what

- `research-before-answer` — assumes 2-level memory cluster + websearch parallel hook + fetch-fallback script
- `wiki-promote` / `memory-audit` / `memory-checkpoint` — assume specific `MEMORY.md` schema with 4 sections (User / Feedback / Reference / Project)
- `video-research` — assumes NotebookLM CLI + personal wiki / memory for personal-context analysis layer

Read each skill's `## Prerequisites` section for specifics.

## 4. How to fork & make it yours

- **Drop-in tools (4)** — copy as-is, no infra needed
- **Framework templates (9)** — copy + rewrite assumed paths / schemas / hooks to match your setup
- Don't sync this repo blindly; cherry-pick what fits your workflow

## CLAUDE.md excerpt (personal rules that shape these skills)

- Use TaskCreate aggressively to track multi-step work
- Dispatch subagents for >3 file reads, >5 file edits, >2 web searches, or 30s+ commands
- Verify before claiming completion (show expected vs actual, ask if matches)
- 反晶晶體: prefer Chinese translation over mixing English in Chinese sentences (technical terms exempted)
- New independent projects go to `~/Desktop/projects/<kebab-case-name>/` with `git init`
