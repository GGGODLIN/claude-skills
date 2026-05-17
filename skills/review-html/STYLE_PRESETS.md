---
name: review-html-style
description: HTML template + CSS design tokens + component patterns for review-html skill. Read by Claude when rendering. NOT a standalone skill — it is a resource file consumed by review-html/SKILL.md.
---

# review-html Style Presets

Claude reads this file when generating an interactive HTML review explorer. Follow the structure exactly; replace `{{PLACEHOLDER}}` markers with content. CSS, JS, design tokens, and component HTML patterns are all in this file.

---

## 1. Design tokens (CSS variables)

Paste this `:root` block inside `<style>` at the top.

```css
:root {
  /* Background */
  --bg-primary: #0e1419;
  --bg-card: #161d24;
  --bg-elevated: #1c252e;
  --bg-hover: #232d37;

  /* Border */
  --border: #2a3540;
  --border-subtle: #1f2830;

  /* Text */
  --text-primary: #e6edf3;
  --text-secondary: #9ba5b1;
  --text-muted: #6b7681;

  /* Accent (muted teal) */
  --accent: #4dc4a8;
  --accent-hover: #5fd4b8;
  --accent-bg: rgba(77, 196, 168, 0.12);

  /* Semantic colors */
  --create: #4ade80;
  --modify: #fbbf24;
  --delete: #f87171;
  --research: #94a3b8;

  /* Risk severity */
  --risk-high: #f87171;
  --risk-medium: #fbbf24;
  --risk-low: #4ade80;

  /* Layout */
  --shadow: 0 4px 16px rgba(0, 0, 0, 0.4);
  --radius: 10px;
  --radius-sm: 6px;
}
```

## 2. Asset CDNs (head)

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=JetBrains+Mono:wght@400;500&family=Noto+Sans+TC:wght@400;500;700&display=swap" rel="stylesheet">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/highlightjs/cdn-release@11.9.0/build/styles/github-dark-dimmed.min.css">
```

Mermaid (only include if spec has diagram-ready kinds: data-model, user-journey, task DAG, timeline, state):

```html
<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true, theme: 'dark', themeVariables: { primaryColor: '#4dc4a8' } });
</script>
```

highlight.js (always include for code blocks):

```html
<script src="https://cdn.jsdelivr.net/gh/highlightjs/cdn-release@11.9.0/build/highlight.min.js"></script>
```

## 3. Base CSS

```css
* { margin: 0; padding: 0; box-sizing: border-box; }
html { scroll-behavior: smooth; }

body {
  background: var(--bg-primary);
  color: var(--text-primary);
  font-family: 'Inter', 'Noto Sans TC', -apple-system, sans-serif;
  font-size: 14px;
  line-height: 1.65;
  -webkit-font-smoothing: antialiased;
}

code, pre { font-family: 'JetBrains Mono', 'SF Mono', monospace; font-size: 13px; }

::-webkit-scrollbar { width: 8px; height: 8px; }
::-webkit-scrollbar-track { background: transparent; }
::-webkit-scrollbar-thumb { background: var(--border); border-radius: 4px; }
::-webkit-scrollbar-thumb:hover { background: var(--text-muted); }

.app {
  display: grid;
  grid-template-columns: 280px 1fr;
  min-height: 100vh;
}

main {
  padding: 32px 40px 80px;
  max-width: 980px;
}
```

## 4. HTML shell template

Use this exact structure. Replace `{{TITLE}}`, `{{KIND_COUNTS}}`, `{{TOC_PHASES}}`, `{{TOC_FILES}}`, `{{HEADER_META}}`, `{{PHASE_STRIP}}`, `{{TASK_CARDS}}`, `{{TYPE_B_BLOCKS}}` with generated content. Omit empty sections.

```html
<!DOCTYPE html>
<html lang="zh-Hant">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{TITLE}} — Review</title>
  <!-- Asset CDNs from §2 here -->
  <style>
    /* Design tokens from §1 */
    /* Base CSS from §3 */
    /* All component CSS from §5 */
  </style>
</head>
<body>
<div class="app">

  <!-- ============ SIDEBAR ============ -->
  <aside class="sidebar">
    <div class="sidebar-header">
      <h1>{{TITLE}}</h1>
      <div class="sidebar-meta">{{KIND_COUNTS}}</div>
      <div class="tab-switch">
        <button data-view="phases" class="active">階段</button>
        <button data-view="files">檔案</button>
      </div>
    </div>
    <nav class="toc toc-phases" id="toc-phases">
      {{TOC_PHASES}}
    </nav>
    <nav class="toc toc-files hidden" id="toc-files">
      {{TOC_FILES}}
    </nav>
  </aside>

  <!-- ============ MAIN ============ -->
  <main>
    <section class="meta-header">
      <h1>{{TITLE_LONG}}</h1>
      <dl class="meta-grid">
        {{HEADER_META}}
      </dl>
    </section>

    <!-- Phase strip (Type A only) -->
    {{PHASE_STRIP}}

    <!-- Task cards (Type A) -->
    {{TASK_CARDS}}

    <!-- Type B blocks (api / data / risk / decision / etc.) -->
    {{TYPE_B_BLOCKS}}
  </main>
</div>

<script>
  /* All JS from §6 */
</script>
</body>
</html>
```

## 5. Component CSS + HTML patterns

### 5.1 Sidebar

```css
.sidebar {
  position: sticky;
  top: 0;
  height: 100vh;
  border-right: 1px solid var(--border);
  background: var(--bg-card);
  overflow-y: auto;
  display: flex;
  flex-direction: column;
}
.sidebar-header { padding: 20px 18px 14px; border-bottom: 1px solid var(--border-subtle); }
.sidebar-header h1 { font-size: 14px; font-weight: 600; margin-bottom: 4px; }
.sidebar-meta { font-size: 11px; color: var(--text-muted); letter-spacing: 0.04em; text-transform: uppercase; }
.tab-switch { display: flex; gap: 4px; margin-top: 14px; padding: 3px; background: var(--bg-primary); border-radius: var(--radius-sm); }
.tab-switch button {
  flex: 1; padding: 6px 10px; background: transparent; color: var(--text-secondary);
  border: none; border-radius: 4px; font-size: 12px; font-family: inherit; cursor: pointer;
  transition: all 0.15s;
}
.tab-switch button.active { background: var(--accent-bg); color: var(--accent); }
.toc { flex: 1; padding: 14px 12px; overflow-y: auto; }
.toc.hidden { display: none; }
```

### 5.2 TOC Phases pattern

```css
.toc-phase { margin-bottom: 10px; }
.toc-phase-title {
  font-size: 10px; font-weight: 600; color: var(--text-muted);
  letter-spacing: 0.08em; text-transform: uppercase; padding: 6px 8px 4px;
}
.toc-link {
  display: flex; align-items: center; gap: 8px; padding: 6px 10px;
  color: var(--text-secondary); text-decoration: none; font-size: 13px;
  border-radius: 4px; border-left: 2px solid transparent; transition: all 0.12s;
}
.toc-link:hover { background: var(--bg-hover); color: var(--text-primary); }
.toc-link.active { background: var(--accent-bg); color: var(--accent); border-left-color: var(--accent); }
.toc-link-id { font-family: 'JetBrains Mono', monospace; font-size: 11px; color: var(--text-muted); min-width: 28px; }
.toc-link.active .toc-link-id { color: var(--accent); }
```

```html
<div class="toc-phase">
  <div class="toc-phase-title">{{PHASE_LABEL_ZH}}</div>
  <a href="#task-{{N}}" class="toc-link"><span class="toc-link-id">T{{N}}</span>{{TASK_NAME_PROSE_ZH}}</a>
  <!-- repeat per task in this phase -->
</div>
```

### 5.3 TOC Files (file impact) pattern

```css
.toc-file-group { margin-bottom: 12px; }
.toc-file-group-title {
  font-size: 10px; font-weight: 600; color: var(--text-muted);
  letter-spacing: 0.08em; text-transform: uppercase; padding: 6px 8px 4px;
}
.toc-file {
  padding: 6px 10px; cursor: pointer; border-radius: 4px;
  font-size: 12px; font-family: 'JetBrains Mono', monospace; color: var(--text-secondary);
}
.toc-file:hover { background: var(--bg-hover); }
.toc-file-icon { display: inline-block; width: 14px; font-weight: 700; }
.toc-file.create .toc-file-icon { color: var(--create); }
.toc-file.modify .toc-file-icon { color: var(--modify); }
.toc-file-tasks {
  padding-left: 22px; margin-top: 4px; font-size: 11px;
  color: var(--text-muted); display: none;
}
.toc-file.expanded .toc-file-tasks { display: block; }
.toc-file-tasks a {
  color: var(--text-muted); text-decoration: none; display: inline-block;
  padding: 2px 6px; border-radius: 3px; margin-right: 4px; margin-bottom: 2px;
  background: var(--bg-primary);
}
.toc-file-tasks a:hover { color: var(--accent); }
```

```html
<div class="toc-file-group">
  <div class="toc-file-group-title">{{GROUP_TITLE_ZH}}</div>
  <div class="toc-file create">
    <span class="toc-file-icon">+</span>{{FILE_PATH}}
    <div class="toc-file-tasks">
      <a href="#task-{{N}}">T{{N}}</a>
      <!-- one anchor per task touching this file -->
    </div>
  </div>
  <div class="toc-file modify">
    <span class="toc-file-icon">✎</span>{{FILE_PATH}}
    <div class="toc-file-tasks">...</div>
  </div>
</div>
```

### 5.4 Header metadata card

```css
.meta-header {
  background: var(--bg-card); border: 1px solid var(--border);
  border-radius: var(--radius); padding: 24px 28px; margin-bottom: 24px;
}
.meta-header h1 { font-size: 22px; font-weight: 700; margin-bottom: 16px; }
.meta-grid {
  display: grid; grid-template-columns: 110px 1fr; gap: 10px 18px; font-size: 13px;
}
.meta-grid dt {
  color: var(--text-muted); font-weight: 500; text-transform: uppercase;
  font-size: 11px; letter-spacing: 0.06em; padding-top: 2px;
}
.meta-grid dd { color: var(--text-secondary); line-height: 1.6; }
.meta-grid dd code {
  background: var(--bg-primary); padding: 1px 6px; border-radius: 3px;
  color: var(--accent); font-size: 12px;
}
```

```html
<dl class="meta-grid">
  <dt>目標</dt>
  <dd>{{GOAL_PROSE_ZH}}</dd>
  <dt>架構</dt>
  <dd>{{ARCHITECTURE_PROSE_ZH}}</dd>
  <dt>技術堆疊</dt>
  <dd>{{TECH_STACK}}</dd>
  <dt>測試策略</dt>
  <dd>{{TESTING_PROSE_ZH}}</dd>
</dl>
```

### 5.5 Phase strip (Type A)

```css
.phase-strip {
  background: var(--bg-card); border: 1px solid var(--border);
  border-radius: var(--radius); padding: 14px 18px; margin-bottom: 24px;
  display: flex; gap: 8px; overflow-x: auto;
}
.phase-pill {
  flex-shrink: 0; padding: 8px 14px; background: var(--bg-elevated);
  border: 1px solid var(--border-subtle); border-radius: 20px;
  font-size: 11px; color: var(--text-secondary); cursor: pointer;
  white-space: nowrap; transition: all 0.15s;
}
.phase-pill:hover { background: var(--bg-hover); color: var(--text-primary); }
.phase-pill.active { background: var(--accent-bg); border-color: var(--accent); color: var(--accent); }
.phase-pill .phase-num { font-family: 'JetBrains Mono', monospace; font-weight: 600; margin-right: 6px; }
.phase-pill .phase-tasks { color: var(--text-muted); font-size: 10px; margin-left: 6px; }
```

```html
<section class="phase-strip">
  <div class="phase-pill"><span class="phase-num">P{{N}}</span>{{PHASE_NAME_ZH}}<span class="phase-tasks">T{{TASK_RANGE}}</span></div>
  <!-- repeat per phase -->
</section>
```

### 5.6 Task card (Type A)

```css
.task-card {
  background: var(--bg-card); border: 1px solid var(--border);
  border-radius: var(--radius); padding: 20px 24px; margin-bottom: 16px;
  scroll-margin-top: 20px;
}
.task-header { display: flex; align-items: baseline; gap: 12px; margin-bottom: 12px; }
.task-id {
  font-family: 'JetBrains Mono', monospace; font-size: 12px; font-weight: 600;
  color: var(--accent); background: var(--accent-bg); padding: 3px 8px; border-radius: 4px;
}
.task-name { font-size: 16px; font-weight: 600; flex: 1; color: var(--text-primary); }
.task-name code {
  background: var(--bg-primary); padding: 2px 6px; border-radius: 3px;
  color: var(--accent); font-size: 14px;
}
.task-phase {
  font-size: 11px; color: var(--text-muted);
  padding: 3px 8px; background: var(--bg-elevated); border-radius: 4px;
}
.task-goal { color: var(--text-secondary); margin-bottom: 14px; font-size: 13px; line-height: 1.6; }
.task-goal strong { color: var(--text-primary); font-weight: 500; }
```

```html
<article class="task-card" id="task-{{N}}">
  <header class="task-header">
    <span class="task-id">任務 {{N}}</span>
    <h2 class="task-name">{{TASK_NAME_ZH}}</h2>
    <span class="task-phase">階段 {{PHASE_N}}</span>
  </header>
  <p class="task-goal"><strong>目標:</strong>{{GOAL_PROSE_ZH}}</p>
  <!-- Files section (5.7) -->
  <!-- Steps (5.8) -->
</article>
```

### 5.7 Files section / file chip

```css
.files-section {
  display: flex; flex-wrap: wrap; gap: 6px; margin-bottom: 16px;
  padding: 10px 12px; background: var(--bg-primary); border-radius: var(--radius-sm);
}
.file-chip {
  display: inline-flex; align-items: center; gap: 6px; padding: 4px 10px;
  background: var(--bg-elevated); border-radius: 4px;
  font-family: 'JetBrains Mono', monospace; font-size: 11px; color: var(--text-secondary);
}
.file-chip-icon { font-weight: 700; font-size: 13px; line-height: 1; }
.file-chip.create { border-left: 2px solid var(--create); }
.file-chip.create .file-chip-icon { color: var(--create); }
.file-chip.modify { border-left: 2px solid var(--modify); }
.file-chip.modify .file-chip-icon { color: var(--modify); }
.file-chip.research { border-left: 2px solid var(--research); color: var(--text-muted); }
```

```html
<div class="files-section">
  <span class="file-chip create"><span class="file-chip-icon">+</span>{{FILE_PATH}}</span>
  <span class="file-chip modify"><span class="file-chip-icon">✎</span>{{FILE_PATH}}</span>
  <span class="file-chip research"><span class="file-chip-icon">∅</span>無檔案修改 — 純研究</span>
</div>
```

### 5.8 Steps (with checkbox + collapsible)

```css
.steps-label {
  font-size: 11px; font-weight: 600; color: var(--text-muted);
  letter-spacing: 0.06em; text-transform: uppercase; margin-bottom: 8px;
}
.step { border-radius: var(--radius-sm); margin-bottom: 6px; }
.step-summary {
  display: flex; align-items: flex-start; gap: 10px; padding: 8px 10px;
  cursor: pointer; list-style: none; border-radius: var(--radius-sm); transition: background 0.12s;
}
.step-summary:hover { background: var(--bg-elevated); }
.step-summary::-webkit-details-marker { display: none; }
.step-checkbox {
  flex-shrink: 0; width: 16px; height: 16px; border: 1px solid var(--border);
  border-radius: 3px; background: var(--bg-primary);
  display: inline-flex; align-items: center; justify-content: center;
  margin-top: 2px; cursor: pointer;
}
.step-checkbox.checked { background: var(--accent); border-color: var(--accent); }
.step-checkbox.checked::after { content: "✓"; color: var(--bg-primary); font-size: 11px; font-weight: 700; }
.step-title { flex: 1; font-size: 13px; color: var(--text-primary); }
.step-title-num {
  font-family: 'JetBrains Mono', monospace; font-size: 11px;
  color: var(--text-muted); margin-right: 6px;
}
.step-toggle { color: var(--text-muted); font-size: 11px; transition: transform 0.15s; }
details[open] .step-toggle { transform: rotate(90deg); }
.step-body { padding: 6px 12px 14px 36px; color: var(--text-secondary); font-size: 13px; }
.step-body p { margin-bottom: 10px; }
.step-body ul { margin: 8px 0 12px 18px; }
.step-body ul li { margin-bottom: 4px; }
.step-body code {
  background: var(--bg-primary); padding: 1px 6px; border-radius: 3px;
  font-size: 12px; color: var(--accent);
}
```

```html
<div class="steps-label">步驟 · {{COUNT}}</div>
<details class="step" open>
  <summary class="step-summary">
    <span class="step-checkbox" onclick="event.preventDefault(); event.stopPropagation(); this.classList.toggle('checked'); persistChecks();"></span>
    <span class="step-title"><span class="step-title-num">步驟 {{N}}</span>{{STEP_TITLE_ZH}}</span>
    <span class="step-toggle">›</span>
  </summary>
  <div class="step-body">
    {{STEP_BODY_PROSE_ZH}}
    <!-- code-fold (5.9) inline if step has code -->
    <!-- commit-block (5.10) inline if step is a commit step -->
  </div>
</details>
```

### 5.9 Code fold

```css
.code-fold {
  background: var(--bg-primary); border: 1px solid var(--border-subtle);
  border-radius: var(--radius-sm); margin: 10px 0; overflow: hidden;
}
.code-fold-summary {
  display: flex; align-items: center; justify-content: space-between;
  padding: 8px 12px; cursor: pointer; list-style: none;
  background: var(--bg-elevated); border-bottom: 1px solid var(--border-subtle); font-size: 12px;
}
.code-fold-summary::-webkit-details-marker { display: none; }
.code-fold-summary .lang {
  font-family: 'JetBrains Mono', monospace; color: var(--accent);
  font-size: 11px; letter-spacing: 0.04em; text-transform: uppercase;
}
.code-fold-summary .meta { color: var(--text-muted); font-size: 11px; }
.code-fold pre { padding: 14px 16px; overflow-x: auto; max-height: 480px; overflow-y: auto; }
details:not([open]) > .code-fold-summary { border-bottom: none; }
```

```html
<details class="code-fold">
  <summary class="code-fold-summary">
    <span class="lang">{{LANG}}</span>
    <span class="meta">{{LINE_COUNT}} 行 · 點擊展開</span>
  </summary>
  <pre><code class="language-{{LANG}}">{{CODE_ESCAPED}}</code></pre>
</details>
```

Code default folded for blocks > 20 lines; expanded for shorter snippets (use your judgment).

### 5.10 Commit block

```css
.commit-block {
  background: linear-gradient(90deg, rgba(77, 196, 168, 0.05), transparent);
  border-left: 2px solid var(--accent);
  padding: 8px 12px; margin: 10px 0; border-radius: 0 4px 4px 0;
}
.commit-block-label {
  font-size: 10px; color: var(--accent); letter-spacing: 0.06em;
  text-transform: uppercase; margin-bottom: 4px; font-weight: 600;
}
.commit-block code {
  font-size: 12px; color: var(--text-primary);
  background: transparent !important; padding: 0 !important;
}
```

```html
<div class="commit-block">
  <div class="commit-block-label">Commit</div>
  <code>{{COMMIT_MESSAGE}}</code>
</div>
```

---

## 6. Type B component patterns

These components render Type B (architecture spec) kinds. First-version sketches — refine when used against real Type B specs.

### 6.1 Goals vs Non-Goals (兩欄)

```css
.goals-grid {
  display: grid; grid-template-columns: 1fr 1fr; gap: 16px; margin-bottom: 24px;
}
.goals-col {
  background: var(--bg-card); border: 1px solid var(--border);
  border-radius: var(--radius); padding: 16px 20px;
}
.goals-col.do { border-left: 3px solid var(--create); }
.goals-col.dont { border-left: 3px solid var(--delete); }
.goals-col h3 { font-size: 12px; color: var(--text-muted); text-transform: uppercase; letter-spacing: 0.08em; margin-bottom: 10px; }
.goals-col ul { list-style: none; }
.goals-col li { padding: 4px 0; font-size: 13px; color: var(--text-secondary); }
.goals-col li::before { content: "•"; margin-right: 8px; color: var(--accent); }
```

```html
<div class="goals-grid">
  <div class="goals-col do">
    <h3>目標</h3>
    <ul><li>{{GOAL_ZH}}</li></ul>
  </div>
  <div class="goals-col dont">
    <h3>非目標</h3>
    <ul><li>{{NON_GOAL_ZH}}</li></ul>
  </div>
</div>
```

### 6.2 API Endpoint card

```css
.api-card {
  background: var(--bg-card); border: 1px solid var(--border);
  border-radius: var(--radius); padding: 16px 20px; margin-bottom: 16px;
}
.api-header { display: flex; align-items: center; gap: 10px; margin-bottom: 12px; }
.api-method {
  font-family: 'JetBrains Mono', monospace; font-weight: 700; font-size: 11px;
  padding: 3px 8px; border-radius: 4px;
}
.api-method.get { background: rgba(74, 222, 128, 0.15); color: var(--create); }
.api-method.post { background: rgba(77, 196, 168, 0.15); color: var(--accent); }
.api-method.put { background: rgba(251, 191, 36, 0.15); color: var(--modify); }
.api-method.delete { background: rgba(248, 113, 113, 0.15); color: var(--delete); }
.api-path { font-family: 'JetBrains Mono', monospace; font-size: 14px; color: var(--text-primary); }
.api-auth { margin-left: auto; font-size: 11px; color: var(--text-muted); padding: 2px 8px; background: var(--bg-elevated); border-radius: 4px; }
.api-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 14px; }
.api-section h4 { font-size: 11px; color: var(--text-muted); text-transform: uppercase; letter-spacing: 0.08em; margin-bottom: 6px; }
.api-section pre { background: var(--bg-primary); padding: 10px; border-radius: var(--radius-sm); font-size: 12px; overflow-x: auto; }
```

```html
<article class="api-card" id="api-{{NAME}}">
  <div class="api-header">
    <span class="api-method {{METHOD_LOWER}}">{{METHOD}}</span>
    <span class="api-path">{{PATH}}</span>
    <span class="api-auth">{{AUTH}}</span>
  </div>
  <p class="task-goal">{{DESCRIPTION_ZH}}</p>
  <div class="api-grid">
    <div class="api-section">
      <h4>請求 Body</h4>
      <pre><code class="language-json">{{BODY_SCHEMA}}</code></pre>
    </div>
    <div class="api-section">
      <h4>回應</h4>
      <pre><code class="language-json">{{RESPONSE_SCHEMA}}</code></pre>
    </div>
  </div>
</article>
```

### 6.3 Data Model (Mermaid ER)

Use Mermaid `erDiagram` directly:

```html
<section class="data-model-section">
  <h3 class="section-title">資料模型</h3>
  <pre class="mermaid">
erDiagram
    {{ENTITY_A}} ||--o{ {{ENTITY_B}} : "{{RELATION}}"
    {{ENTITY_A}} {
      {{TYPE}} {{FIELD}} "{{COMMENT_ZH}}"
    }
  </pre>
</section>
```

### 6.4 Risk matrix

```css
.risk-matrix {
  display: grid; grid-template-columns: 80px repeat(3, 1fr); gap: 4px;
  margin-bottom: 24px; background: var(--bg-card); border: 1px solid var(--border);
  border-radius: var(--radius); padding: 14px;
}
.risk-axis { font-size: 11px; color: var(--text-muted); padding: 8px; text-align: center; }
.risk-cell {
  padding: 12px; border-radius: var(--radius-sm); min-height: 80px;
  background: var(--bg-elevated); font-size: 12px; color: var(--text-secondary);
}
.risk-cell.high { background: rgba(248, 113, 113, 0.1); border-left: 3px solid var(--risk-high); }
.risk-cell.medium { background: rgba(251, 191, 36, 0.1); border-left: 3px solid var(--risk-medium); }
.risk-cell.low { background: rgba(74, 222, 128, 0.1); border-left: 3px solid var(--risk-low); }
.risk-item { display: block; padding: 4px 0; }
```

```html
<section class="risk-matrix">
  <div></div>
  <div class="risk-axis">低機率</div>
  <div class="risk-axis">中機率</div>
  <div class="risk-axis">高機率</div>

  <div class="risk-axis">高影響</div>
  <div class="risk-cell medium"><span class="risk-item">{{RISK_ZH}}</span></div>
  <div class="risk-cell high"><span class="risk-item">{{RISK_ZH}}</span></div>
  <div class="risk-cell high"><span class="risk-item">{{RISK_ZH}}</span></div>

  <div class="risk-axis">中影響</div>
  <div class="risk-cell low"></div>
  <div class="risk-cell medium"></div>
  <div class="risk-cell high"></div>

  <div class="risk-axis">低影響</div>
  <div class="risk-cell low"></div>
  <div class="risk-cell low"></div>
  <div class="risk-cell medium"></div>
</section>
```

### 6.5 Decision log timeline

```css
.decision-log { background: var(--bg-card); border: 1px solid var(--border); border-radius: var(--radius); padding: 20px; margin-bottom: 24px; }
.decision { padding: 12px 14px; border-left: 2px solid var(--accent); margin-bottom: 10px; background: var(--bg-elevated); border-radius: 0 var(--radius-sm) var(--radius-sm) 0; }
.decision-title { font-size: 13px; font-weight: 600; color: var(--text-primary); margin-bottom: 4px; }
.decision-rationale { font-size: 12px; color: var(--text-secondary); margin-bottom: 6px; }
.decision-alt { font-size: 11px; color: var(--text-muted); }
.decision-alt::before { content: "其他選項:"; color: var(--accent); }
```

```html
<section class="decision-log">
  <h3 class="section-title">決策記錄</h3>
  <div class="decision">
    <div class="decision-title">{{DECISION_ZH}}</div>
    <div class="decision-rationale">{{RATIONALE_ZH}}</div>
    <div class="decision-alt">{{ALTERNATIVES_ZH}}</div>
  </div>
</section>
```

### 6.6 Acceptance criteria checklist

```css
.ac-list { background: var(--bg-card); border: 1px solid var(--border); border-radius: var(--radius); padding: 16px 20px; margin-bottom: 16px; }
.ac-list h3 { font-size: 13px; margin-bottom: 10px; }
.ac-list label { display: flex; gap: 10px; padding: 6px 0; font-size: 13px; color: var(--text-secondary); cursor: pointer; }
.ac-list input[type="checkbox"] { accent-color: var(--accent); }
```

```html
<section class="ac-list">
  <h3>驗收標準</h3>
  <label><input type="checkbox" data-ac="{{ID}}"> {{AC_ZH}}</label>
</section>
```

### 6.7 User journey (Mermaid journey)

```html
<section class="journey-section">
  <h3 class="section-title">使用者旅程</h3>
  <pre class="mermaid">
journey
    title {{JOURNEY_TITLE_ZH}}
    section {{SECTION_ZH}}
      {{STEP_ZH}}: {{SCORE}}: {{ACTOR}}
  </pre>
</section>
```

### 6.8 Persona card

```css
.persona-card { background: var(--bg-card); border: 1px solid var(--border); border-radius: var(--radius); padding: 16px 20px; margin-bottom: 12px; display: flex; gap: 16px; }
.persona-avatar { font-size: 32px; }
.persona-body h4 { font-size: 14px; margin-bottom: 4px; }
.persona-meta { font-size: 11px; color: var(--text-muted); margin-bottom: 8px; }
.persona-quote { font-size: 12px; color: var(--text-secondary); font-style: italic; padding-left: 10px; border-left: 2px solid var(--accent); }
```

```html
<article class="persona-card">
  <div class="persona-avatar">{{EMOJI}}</div>
  <div class="persona-body">
    <h4>{{NAME}}</h4>
    <div class="persona-meta">{{ROLE_ZH}} · {{CONTEXT_ZH}}</div>
    <div class="persona-quote">「{{QUOTE_ZH}}」</div>
  </div>
</article>
```

---

## 7. JS interactions

```js
// hljs init (always)
if (typeof hljs !== 'undefined') hljs.highlightAll();

// Sidebar tab switch
document.querySelectorAll('.tab-switch button').forEach(btn => {
  btn.addEventListener('click', () => {
    document.querySelectorAll('.tab-switch button').forEach(b => b.classList.remove('active'));
    btn.classList.add('active');
    const view = btn.dataset.view;
    document.getElementById('toc-phases')?.classList.toggle('hidden', view !== 'phases');
    document.getElementById('toc-files')?.classList.toggle('hidden', view !== 'files');
  });
});

// File chip expand
document.querySelectorAll('.toc-file').forEach(file => {
  file.addEventListener('click', e => {
    if (e.target.tagName === 'A') return;
    file.classList.toggle('expanded');
  });
});

// Phase pill click → scroll to first task in phase
document.querySelectorAll('.phase-pill').forEach(p => {
  p.addEventListener('click', () => {
    document.querySelectorAll('.phase-pill').forEach(x => x.classList.remove('active'));
    p.classList.add('active');
    const firstTaskId = p.dataset.firstTask;
    if (firstTaskId) document.getElementById(firstTaskId)?.scrollIntoView({ behavior: 'smooth', block: 'start' });
  });
});

// Scroll-spy: highlight TOC link when its task is in view
const observer = new IntersectionObserver(entries => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const id = entry.target.id;
      document.querySelectorAll('.toc-link').forEach(a => {
        a.classList.toggle('active', a.getAttribute('href') === `#${id}`);
      });
    }
  });
}, { rootMargin: '-20% 0px -60% 0px' });
document.querySelectorAll('.task-card, .api-card').forEach(t => observer.observe(t));

// Persist checkbox state to localStorage (per-spec)
const SPEC_KEY = location.pathname.split('/').pop();
function persistChecks() {
  const state = {};
  document.querySelectorAll('.step-checkbox').forEach((cb, i) => {
    state[i] = cb.classList.contains('checked');
  });
  localStorage.setItem('review-html:' + SPEC_KEY, JSON.stringify(state));
}
function restoreChecks() {
  try {
    const state = JSON.parse(localStorage.getItem('review-html:' + SPEC_KEY) || '{}');
    document.querySelectorAll('.step-checkbox').forEach((cb, i) => {
      if (state[i]) cb.classList.add('checked');
    });
  } catch {}
}
restoreChecks();
```

Add `data-first-task="task-N"` to each `.phase-pill` so scroll target works.

---

## 8. Translation glossary (consult while rendering)

UI shell + structural label translations Claude should apply consistently:

| English | 繁中 |
|---|---|
| Phases | 階段 |
| Files | 檔案 |
| Tasks | 任務 |
| Steps | 步驟 |
| Goal | 目標 |
| Architecture | 架構 |
| Tech Stack | 技術堆疊 |
| Testing | 測試策略 |
| Backend | 後端 |
| Frontend (create) | 前端 (新增) |
| Frontend (modify) | 前端 (修改) |
| click to expand | 點擊展開 |
| lines | 行 |
| None modified — research only | 無檔案修改 — 純研究 |
| Goals | 目標 |
| Non-Goals | 非目標 |
| Risks | 風險 |
| Decisions | 決策記錄 |
| Acceptance Criteria | 驗收標準 |
| User Journey | 使用者旅程 |
| Personas | 角色 |
| API | API (do not translate) |
| Commit | Commit (do not translate — preserve user-facing term) |
| Request Body | 請求 Body |
| Response | 回應 |
| High / Medium / Low | 高 / 中 / 低 |

Phase / task names use spec's own descriptive nouns; translate descriptively (e.g. Welcome banner → 歡迎橫幅, Page shell → 頁面骨架).

---

## 9. Output sanity checklist

Before writing the file, verify:

- [ ] HTML is single-file (CSS + JS inline, only CDN external resources)
- [ ] All translation rules applied (shell + prose translated, code untouched)
- [ ] Code blocks default-folded if > 20 lines, expanded if shorter
- [ ] Sidebar TOC has both 階段 and 檔案 views (or only one if spec is Type B without phases)
- [ ] All Mermaid blocks valid (test mentally that syntax parses)
- [ ] No `{{PLACEHOLDER}}` strings left in output
- [ ] Lang attribute is `zh-Hant` on `<html>`
- [ ] highlight.js + Mermaid scripts loaded if needed
