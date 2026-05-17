---
description: Memory 系統健康度 read-only 診斷（不動 file，輸出 structured health report）
---

## Prerequisites

This is a framework template, not drop-in. To use this command you need:

- `~/.claude/projects/{{SCOPE}}/memory/MEMORY.md` with 4-section cluster schema (User / Feedback / Reference / Project) — adapt to your schema
- `_index_*.md` cluster files and `_orphan_registry.md` — self-build
- The audit logic is hardcoded for a specific MEMORY.md schema; reader with different memory layout needs to rewrite the audit rules

**This is a framework, not a drop-in tool.**

# Memory Audit

跑 read-only 健康度檢查。**不動任何 file**，輸出結構化 health report。

⚠️ **Multi-session safety**: 跨 session 平行編輯 memory 是常態。本 command 永遠讀**當下 file 系統實際狀態**（Bash / Read tool），**不依賴** CC inject 的 MEMORY.md snapshot（那是 session start 時的，不會即時 update）。看到 mismatch 先確認是當下狀態還是某個 session 的 mid-process。

## 1. 抓基本資訊

```bash
PROJECT_HASH=$(pwd | sed 's|^/|-|; s|/|-|g')
MEMORY_DIR="$HOME/.claude/projects/$PROJECT_HASH/memory"
[[ -d "$MEMORY_DIR" ]] || { echo "no memory dir for current project"; exit 0; }
ls "$MEMORY_DIR"/_index_*.md 2>/dev/null   # 動態 audit cluster 結構（不要記固定 list）
```

## 2. Audit 7 個維度

### A. Size + lines vs CC 25 KB / 200 lines hard limit

```bash
size=$(wc -c < "$MEMORY_DIR/MEMORY.md")
lines=$(wc -l < "$MEMORY_DIR/MEMORY.md")
entries=$(grep -c '^- \[' "$MEMORY_DIR/MEMORY.md")
```

判定：
- ❌ `size >= 25000` 或 `lines >= 200` → over limit，需 trim
- ⚠️ `size >= 22500` (90%) → 接近 limit
- ✅ otherwise

### B. 結構 check

驗證 MEMORY.md 含：
- 4 sections: `## User`, `## Feedback`, `## Reference`, `## Project`
- Reference 段含 `### Cluster indexes` + `### Standalone reference` sub-sections

判定：❌ section 缺 / ✅ 完整

### C. Cluster integrity

對每個 `_index_*.md`:
1. Frontmatter `(N entries)` 跟 `## Entries` 段內實際 `- [` 行數對齊？
2. Cluster 內每個 entry 指向的 source file 存在嗎？

判定：❌ count mismatch → 修 frontmatter；❌ broken link → 處理孤兒指向；✅ otherwise

### D. Orphan + double-ref

- `all_topics` = `*.md` 排除 `MEMORY.md` + `_index_*` + `_orphan_registry.md`
- `ref_in_memory` = MEMORY.md 內 link target（排除 cluster pointer，即 `_index_*` 開頭）
- `ref_in_cluster` = 各 cluster file 內 link target
- `intentional_orphans` = `_orphan_registry.md` 內列出的 file（grep `^| \`(.+\.md)\`` 抓 filename）
- **Orphan** = topic file 在 dir 但無 reference
- **Intentional orphan** = orphan 命中 `intentional_orphans` set（E pass / 25KB cap 時 prune 但不刪檔）
- **Unregistered orphan** = orphan 且不在 registry 內 — 真的「忘記補 index」case
- **Double-ref** = 同時在 MEMORY.md standalone + 某 cluster file（違反 cluster maintenance rule）

判定：
- ❌ `Unregistered orphan` 非 0 → 立刻修（加進 cluster / MEMORY.md / 刪檔 / 確定故意 prune → append 到 `_orphan_registry.md`）
- ❌ `Double-ref` 非 0 → 立刻修
- ⚠️ `Intentional orphan` 非 0 → suggestion 區列出（標 registry hit、可審視是否仍合理）
- ✅ `Unregistered orphan` + `Double-ref` 皆 0

### E. Cluster pointer alignment

- Dir 內 `_index_*.md` 數量 vs MEMORY.md `### Cluster indexes` 段 cluster pointer 數量
- 兩集合元素對齊？

判定：❌ dir 有 cluster file 但 MEMORY.md 沒 pointer / ❌ MEMORY.md 有 pointer 但 dir 沒 file / ✅ 完全對齊

### F. Cluster maturity（read-only suggestion only）

對每個 cluster:
- Entry count
- Working tree mtime（最舊 / 最新；用 `stat -f %Sm -t %F` 拿 entries 的 mtime）
- Git log 最近 commit（若 git 可用）

標籤：
- **mature**: `>= 3 entries` + 最新 mtime `> 7 天前` + 看 description 偏 entity-centric → 提示「考慮 `/wiki-promote`」
- **活躍**: 最新 mtime `< 7 天` → 內容還在演進，wait until stable
- **太薄**: `< 3 entries` → 觀察是否該 dissolve 或等累積

判定：純 ⚠️ suggestion，不算 ❌

### G. Standalone clustering opportunities

掃 `### Standalone reference` 段 + `## Project` 段 entry filename + description，找：
- 同 keyword / 同 topic 累積 ≥3 條（且不在現有 cluster 內）→ 提示「考慮建新 cluster」

判定：純 ⚠️ suggestion

## 3. Output 格式

固定結構，分 ✅ / ⚠️ / ❌ 三層：

```
=== Memory Audit Report ===
Memory dir: <MEMORY_DIR>
Audit time: <UTC timestamp>

✅ Healthy
  - Size: X.X KB / N lines (X% of limit)
  - Cluster integrity: N/N OK
  - Unregistered orphan: 0 / Double-ref: 0
  - Cluster pointer alignment: N/N OK

⚠️ Suggestions
  - Intentional orphans (registry hit, N 條): <列出 filename + reason 摘要>
  - Cluster `<name>`: mature（≥7 天 stable, N entries）→ consider `/wiki-promote`
  - Standalone reference 主題分布: keyword `<x>` 累積 N → consider new cluster
  - ...
  (若無 suggestion 就 list "(none)")

❌ Must fix
  - Unregistered orphan (N) → 加進 cluster / MEMORY.md / 刪檔 / 確定故意 prune → append 到 `_orphan_registry.md`
  - ...
  (若無 must-fix 就完全跳過 ❌ 段，不要列 "(none)")
```

## 4. 不要做的事

- **不動任何 file** — read-only diagnostic
- **不 ls scan 把 cluster file 當「孤兒」索引進 MEMORY.md** — `_index_*` 開頭永遠排除
- **不依賴 CC inject 的 MEMORY.md snapshot** — 用 Bash / Read tool 拿當下狀態
- **不釘死 cluster 數量 / 名單** — 每次跑 `ls _index_*.md` 動態抓

## 5. 完成後

簡短 summary：`✅ N / ⚠️ N / ❌ N`。

如果有 ❌：提示「執行 `/memory-checkpoint` 或手動修對應 file」。
如果只有 ⚠️：使用者自行決定要不要動。
全 ✅：簡短報「memory 健康」即可。

## 6. Implementation hint

跑 audit 用 Python script 比 awk/sed 可靠（中文 unicode + frontmatter parsing）。一個 monolithic script 處理 A-G 7 個維度然後 print structured output 最乾淨。

## 7. 設計 cross-reference

- `feedback_memory_cluster_maintenance.md` — cluster 維護規則 + E pass / intentional orphan 規則（audit 是它的 health check 配套）
- `_orphan_registry.md` — intentional orphan 白名單（E pass prune 後仍留檔的 entry）；audit 對照本檔 demote intentional orphan 為 Suggestions 不是 Must-fix
- `feedback_prompt_principle_over_hardcoded_list.md` — 動態 ls 不釘死 cluster list 原則
- `reference_cc_auto_memory_limits.md` — 25 KB / 200 行 limit 來源
- `/memory-checkpoint` — 寫入時 cluster check（互補：write-time vs ad-hoc audit）
- `/wiki-promote` — promotion candidate 細節 audit + drafting（本 command 只 hint，不 draft）
