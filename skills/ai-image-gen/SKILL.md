---
name: ai-image-gen
description: Single entry point for ANY request to create, generate, design, draw, make, render, produce, or edit a visual asset. This explicitly includes logos, icons, brand marks, and design briefs — they count as image-generation tasks under this skill, not as generic "design work". Also covers photos, illustrations, comic/anime art, diagrams, flowcharts, wireframes, mockups, posters, banners, textures, sprites, concept art, and edits of existing images. Triggers on vague image requests ("做一張圖", "need an image") AND on traditional-sounding design briefs ("設計一個 logo", "做個 icon"). Trigger phrases (any language): "產圖", "生成圖片", "畫一張", "做一張圖", "做張圖", "做個圖", "畫個", "設計 logo", "設計一個 logo", "做個 logo", "設計圖標", "做個 icon", "generate image", "make an image", "create picture", "render image", "design a logo", "make me a logo", "I need an icon", "AI art", "illustration", "brand mark". ALWAYS invoke this skill when the user request could plausibly resolve to a visual output, even if it looks like a traditional design brief.
---

# AI Image Generation Skill

Single entry point for all image-generation tasks. Classifies the request, routes to the right path, and protects the user from paid API calls they didn't approve.

## Hard rules

1. **codex `$imagegen` is the default path.** It costs nothing extra (uses the user's ChatGPT subscription login).
2. **Never call a paid image generation path** (Gemini MCP, OpenAI Images API, AI Studio MCP, etc.) **without explicit user consent for that specific call.** Previous consent does not carry over.
3. **Never put file-management instructions inside the codex prompt** (no "save as", "copy to", "confirm file exists"). This causes agent-loop stalls. The skill handles file movement *after* codex returns.
4. **Never overwrite an existing asset** unless the user explicitly asked for replacement. Use versioned filenames (`-v2`, `-v3`) instead.

## Classification (do this first)

Read the user's request and classify into one of five categories:

| Category | Signals | Route |
|---|---|---|
| **A. Photo / illustration / comic / anime / AI art** | Scene, lighting, character, style keywords, photorealistic, painting, concept art | **Run codex `$imagegen` directly** |
| **B. Logo / icon / flat graphic / vector-friendly** | "logo", "icon", "flat design", "vector", geometric, brand mark | **Ask user first**: codex raster, or SVG/HTML directly? |
| **C. Diagram / flowchart / wireframe / schematic** | "flowchart", "architecture diagram", "wireframe", "sequence diagram" | **Ask user first**: codex raster, or Mermaid/SVG structured? |
| **D. Edit existing image** | User provides an image path + change instruction (inpainting, background swap, object removal, style change) | **Run codex `$imagegen` in edit mode** |
| **E. Unclear** | Cannot classify confidently | **Ask user** which of A–D it is |

Only A and D run without asking. B, C, E always confirm first.

## Path: codex `$imagegen` (default for A and D)

### Prompt rules

- Prefix the prompt with `$imagegen` (shell-escape as `\$imagegen` when calling from bash to avoid variable expansion).
- Include only the visual description. Do not include save paths, filenames, or confirmation steps.
- If the user's original request contains heavy file-management instructions, strip them from the codex prompt and handle them yourself.
- Keep the visual prompt reasonable length. If the user's prompt is very long and packed with negative conditions, it can stall the codex agent loop — consider light trimming or warn the user.

### Invocation

```bash
codex exec --full-auto --skip-git-repo-check "\$imagegen <pure visual prompt>"
```

- Wall-clock timeout: **600 seconds (10 minutes)**.
- If 60 seconds pass with codex still running, emit a brief "仍在產圖中…" progress note to the user so they know it's not hung.
- For edit mode (category D), pass the source image via `-i <path>` to `codex exec`, and phrase the prompt as an edit instruction.

### Locating the output

After codex finishes:

1. Parse `session id: <UUID>` from codex stdout.
2. Check `~/.codex/generated_images/<session-id>/ig_*.png`.
3. If parse fails, fall back to: most recently modified subdirectory under `~/.codex/generated_images/`.
4. If still no `ig_*.png` exists → treat as failure, go to the failure-fallback section.

### Copy to target

- If user specified a destination filename/path → use it.
- If not specified → default to the current working directory with name `<semantic-slug>-<YYYYMMDD>.png` (extract a short slug from the prompt, e.g. `superhero-20260417.png`).
- If the target filename already exists → append `-v2`, `-v3`, … until unique.
- Use `cp`, not `mv` — leave the original in `~/.codex/generated_images/` alone.

## Path: ask user first (for B, C, E)

Present the choice concisely, then wait:

- **B (logo/icon)**: "這看起來是 logo/icon，建議走 SVG/HTML 直接寫（可編輯、體積小、無成本）；也可以用 codex `$imagegen` 產 raster。要走哪一條？"
- **C (diagram)**: "這像是流程圖/架構圖，建議用 Mermaid 或 SVG 結構化（可編輯、可版控）；也可以用 codex `$imagegen` 產 raster。要走哪一條？"
- **E (unclear)**: Describe your best guess of the categories and ask the user to pick.

Proceed only after user picks. If they pick codex raster, follow the codex path above.

## Failure fallback (paid path — requires explicit consent)

Codex path is considered **failed** when:

- Wall-clock exceeds 600 seconds → kill codex, proceed to fallback.
- Codex finishes but no `ig_*.png` was produced in the session directory.
- Codex returns a non-zero exit with an obvious error (auth, quota, sandbox).

On failure:

1. Report the failure mode to the user (what happened, briefly — timeout vs no output vs error).
2. Ask explicitly, in one turn: "Codex 失敗（原因 X）。要改用 Gemini MCP 嗎？會扣 GEMINI_API_KEY（約 $0.03–0.06/張）。要繼續就回 yes。"
3. **Wait for explicit "yes" (or equivalent clear consent).** Do not infer consent from silence, context, or prior preferences.
4. On consent: call `mcp__gemini__gemini-generate-image`. Copy MCP output to target destination using the same naming rules as the codex path.
5. If the user declines or stays silent → stop, leave the task incomplete, do not try other paid paths.

Other paid paths (`mcp__aistudio__generate_content`, OpenAI Images API direct) follow the same consent rule — never call without a per-call explicit yes.

## Error modes to watch for

- **codex agent loop stall**: seen with long complex prompts that also include file-management instructions. Mitigation is in the "Prompt rules" section above — keep the codex prompt pure-visual.
- **stdout pipe buffering**: do not pipe codex output through `| tail -N`. It can buffer and hide progress. Let stdout stream directly.
- **stale `superhero.png` etc.**: if a previous codex run partially failed, old files may be missing or in unexpected state. Never assume target dir is clean — check before writing.
- **`$` expansion**: in bash, `$imagegen` will expand to empty string. Always escape as `\$imagegen` in the shell command.

## Example (category A, default path)

User: "幫我產一張賽博朋克風的街道夜景"

1. Classify → A (scene/style keywords).
2. Build codex prompt: `$imagegen A cyberpunk street scene at night, neon lights, rainy asphalt, reflections, dense urban atmosphere, cinematic wide angle`
3. Run `codex exec --full-auto --skip-git-repo-check "\$imagegen …"`.
4. Parse session id, locate `ig_*.png`, `cp` to `./cyberpunk-street-20260417.png`.
5. Report path + size to user.

## Example (category B, ask first)

User: "幫我做一個公司 logo，藍色簡潔風"

1. Classify → B.
2. Ask: "logo 建議走 SVG/HTML 直接寫，可編輯、體積小、無成本；也可以用 codex `$imagegen` 出 raster。要哪條？"
3. Wait.
4. If user picks codex → follow default path. If SVG → write SVG directly, don't use codex.
