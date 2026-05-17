# claude-skills

Personal Claude Code skill / command collection by [gggodlin](https://github.com/gggodlin).

**Reading list, not a tool.** Browse, fork, copy what you like; nothing here is guaranteed to work on your machine. Read [`CONTEXT.md`](CONTEXT.md) before installing.

---

## Drop-in tools (4)

Install as-is. No personal framework dependencies.

| Name | Type | What it does |
|---|---|---|
| [`review-html`](skills/review-html/SKILL.md) | skill | Render long spec/plan markdown into interactive Traditional Chinese HTML for review |
| [`review-zh`](skills/review-zh/SKILL.md) | skill | Auto-generate Traditional Chinese review copy of English markdown |
| [`ai-image-gen`](skills/ai-image-gen/SKILL.md) | skill | Single entry point for any image generation request |
| [`changelog-check`](commands/changelog-check.md) | command | Fetch Shopify changelog entries since last check |

## Framework templates with prerequisites (9)

Tied to author's personal stack. Treat as templates; expect to fork and adapt. Each has a `## Prerequisites` section.

| Name | Type | What it does |
|---|---|---|
| [`research-before-answer`](skills/research-before-answer/SKILL.md) | skill | Research discipline for fact questions: individual layer (wiki+memory) + external parallel |
| [`video-research`](skills/video-research/SKILL.md) | skill | NotebookLM CLI baseline + personal-context layered analysis |
| [`figma-context-budget`](skills/figma-context-budget/SKILL.md) | skill | Context-efficient Figma MCP usage patterns |
| [`wiki-promote`](commands/wiki-promote.md) | command | Promote mature memory conclusions to publishable wiki entity |
| [`memory-audit`](commands/memory-audit.md) | command | Memory system health diagnosis |
| [`memory-checkpoint`](commands/memory-checkpoint.md) | command | Manual memory write (vendor model workaround) |
| [`sync-claude-setting-cross-platform`](commands/sync-claude-setting-cross-platform.md) | command | Cross-machine `~/.claude/` sync flow |
| [`pr-review`](commands/pr-review.md) | command | Dual-model PR review (Opus + Codex) |
| [`devfleet`](commands/devfleet.md) | command | Orchestrate parallel Claude Code agents via DevFleet MCP |

## Install

```bash
git clone https://github.com/gggodlin/claude-skills.git
cp -r claude-skills/skills/* ~/.claude/skills/
cp claude-skills/commands/* ~/.claude/commands/
```

The manual `cp` approach is intentional: **read it before installing**.

<details>
<summary>Advanced</summary>

- `npx skills@latest add github:gggodlin/claude-skills` (vercel-labs CLI)
- Cherry-pick: download single file via `curl` or GitHub web UI
</details>

## Support policy

This is a public mirror for reading.

- **Issues / PRs are NOT actively monitored.**
- For bug reports about my setup, use GitHub Discussions or reach me at [@gggodlin on X](https://x.com/gggodlin).
- This repo follows my personal `~/.claude/` periodically; expect periods of inconsistency.

## License

[0BSD](LICENSE) — take it, no attribution required.

## Acknowledgments

借鑑來源 / Inspirations:

- [anthropics/skills](https://github.com/anthropics/skills) — official Claude Code skills
- [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official) — plugin marketplace
- [mattpocock/skills](https://github.com/mattpocock/skills) — personal `.claude/` directory style (this repo follows the same philosophy)
- [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills) + [vercel-labs/skills](https://github.com/vercel-labs/skills) — agent skill ecosystem
