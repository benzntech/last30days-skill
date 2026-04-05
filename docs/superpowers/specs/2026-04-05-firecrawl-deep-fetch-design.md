# Firecrawl Deep Fetch — Design Spec

**Date:** 2026-04-05  
**Status:** Approved  
**Scope:** SKILL.md + variants/open/SKILL.md only (agent-layer change, no Python subprocess changes)

---

## Problem

Web search results from Exa MCP and WebSearch return snippets only (200–500 chars). Reddit threads get full comment enrichment; YouTube gets full transcripts. Web articles are the only source with no depth — synthesis sees teasers, not content.

---

## Solution

Add **Step 2b: Deep Fetch** to SKILL.md. After Step 2 finds URLs, Firecrawl MCP scrapes the full markdown content of the top N results before synthesis. No API key required.

---

## Architecture

### Layer

Agent-layer only (SKILL.md instructions). Firecrawl MCP tools are called by the skill, not inside the Python subprocess. Consistent with the Exa MCP pattern established in the same session.

### Tools added to `allowed-tools`

```
mcp__firecrawl__firecrawl_scrape
mcp__firecrawl__firecrawl_crawl
```

Both `SKILL.md` and `variants/open/SKILL.md` updated.

---

## Depth-Adaptive Fetch Count

| Mode | URLs fetched |
|------|-------------|
| `--quick` | 3 |
| default | 5 |
| `--deep` | 8 |

Mirrors the existing depth profile pattern (quick/default/deep timeouts and volumes).

---

## Step 2b: Placement and Flow

**Placement:** After Step 2 (Exa MCP + WebSearch), before Judge Agent synthesis.

**URL selection:**
1. Collect all URLs from Step 2 results
2. Rank by Exa relevance score descending; fall back to return order if Exa MCP wasn't used
3. Skip excluded domains: `reddit.com`, `x.com`, `twitter.com` (covered by script)
4. Take top N per depth profile

**Fetch execution:**
- Call `mcp__firecrawl__firecrawl_scrape` for each selected URL
- Run in parallel where possible
- Truncate fetched markdown to **1500 characters** per article before passing to synthesis
- On failure or timeout for any URL: skip silently, do not block synthesis

---

## Synthesis Integration

- Firecrawl-enriched articles count as web sources — no separate section
- Full-content articles weighted higher than snippet-only in synthesis (more signal)
- Source names appear on the existing `🌐 Web:` stats line — format unchanged

---

## Flags

| Flag | Behaviour |
|------|-----------|
| `--no-native-web` | Skip Step 2b entirely (consistent: flag means no agent-layer web fetching) |
| `--quick` | Fetch 3 URLs |
| `--deep` | Fetch 8 URLs |

---

## Diagnostics

`--diagnose` output adds:
```
Firecrawl: active (MCP, no key required)
```
Always active — no key dependency.

---

## What Does NOT Change

- `scripts/lib/` — no Python changes
- `exa_search.py` HTTP path — unchanged
- Stats block format — unchanged
- Excluded domains list — unchanged
- `scripts/sync.sh` deploy process — unchanged

---

## Files to Modify

1. `SKILL.md`
   - Add `mcp__firecrawl__firecrawl_scrape`, `mcp__firecrawl__firecrawl_crawl` to `allowed-tools`
   - Add Step 2b section after Step 2
   - Update `--diagnose` description in the transparency section

2. `variants/open/SKILL.md`
   - Add same two tools to `allowed-tools`

---

## Out of Scope

- `firecrawl_search` as a search source (Role B) — explicitly excluded
- `firecrawl_crawl` for recursive site crawling — tool allowed but not used in Step 2b
- Python subprocess integration — architecturally impossible (MCP tools are agent-layer only)
- Firecrawl API key support — not needed, MCP works without one
