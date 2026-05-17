---
description: Fetch Shopify changelog entries since the last check, summarize, then update the per-repo last-checked marker.
---

# Changelog Check

Pull recent Shopify changelog entries, summarize what matters, and reset the weekly reminder.

## When to Use

- A changelog reminder (e.g. a per-repo hook) flagged that 7+ days have passed since the last check
- You want a manual on-demand check before that

## Process

### Step 1: Read the current marker

Read `$REPO_ROOT/.claude/changelog-last-checked.txt` (where `$REPO_ROOT` is the current git repo top-level). The file holds a single `YYYY-MM-DD` date. If it does not exist, treat the cutoff as 7 days ago.

### Step 2: Fetch the changelog

Fetch <https://shopify.dev/changelog> using WebFetch. Ask the fetch model to extract every entry **dated on or after the cutoff from Step 1**, returning for each entry: date, title, category/tag, a one-sentence summary, and the **full absolute URL to the entry's detail page** (e.g. `https://shopify.dev/changelog/<slug>`). The URL is mandatory — re-prompt if missing.

If WebFetch returns nothing or the page structure has changed, fall back to `mcp__shopify-dev-mcp__search_docs_chunks` with a query like "changelog recent updates" and surface what comes back (still extract per-entry URLs from the chunks).

### Step 3: Summarize for the user

Present the entries grouped by date (newest first). For each entry give:

- **Title as a clickable markdown link** pointing to the entry URL — `[Title](https://shopify.dev/changelog/...)`. Never present a title without its link.
- A one-line takeaway
- The Shopify-area it touches (Admin API, Liquid, Checkout extensions, etc.)
- A flag if it looks relevant to your project's focus areas (e.g. checkout extensions, webhooks, admin UI, payment APIs — adapt to what your app actually uses)

Keep it short — a quick scannable list (table form is fine), not a full report. If nothing relevant since cutoff, say so explicitly. **Every entry must carry its clickable URL** so the user can jump to the original page.

### Step 4: Update the marker

After presenting the summary, run:

```bash
date +%Y-%m-%d > "$(git rev-parse --show-toplevel)/.claude/changelog-last-checked.txt"
```

This silences the reminder for the next 7 days.

## Edge Cases

**Not in a git repo:** Skip Step 4 and tell the user the marker can't be updated outside a repo.

**Marker file missing:** Run Steps 2–3 with cutoff = today − 7 days, then create the marker in Step 4.

**Network / fetch failure:** Report the failure, don't update the marker. The reminder will fire again tomorrow.

**No new entries since cutoff:** Still update the marker in Step 4 — the user did the check, it just happened to be quiet.
