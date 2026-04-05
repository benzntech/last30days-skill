# Firecrawl Deep Fetch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Firecrawl MCP as a Step 2b deep-fetch layer in SKILL.md, scraping full article content for the top N web URLs after Step 2 search completes.

**Architecture:** Agent-layer only — all changes are in SKILL.md instruction text. Firecrawl MCP tools (`mcp__firecrawl__firecrawl_scrape`) are called by the skill at runtime, not inside the Python subprocess. Follows the same pattern as the Exa MCP integration added in the same session.

**Tech Stack:** SKILL.md markdown instructions, Firecrawl MCP (no API key required), `bash scripts/sync.sh` for deployment.

---

## File Map

| File | Change |
|------|--------|
| `SKILL.md` | Add 2 tools to `allowed-tools`; insert Step 2b section; update Security & Permissions bullet |
| `variants/open/SKILL.md` | Add same 2 tools to `allowed-tools` only |

---

### Task 1: Add Firecrawl tools to `allowed-tools` in both SKILL.md files

**Files:**
- Modify: `SKILL.md` line 6
- Modify: `variants/open/SKILL.md` line 6

- [ ] **Step 1: Edit main SKILL.md `allowed-tools` line**

Find line 6 in `SKILL.md`:
```
allowed-tools: Bash, Read, Write, AskUserQuestion, WebSearch, mcp__exa__web_search_exa, mcp__exa__crawling_exa
```
Replace with:
```
allowed-tools: Bash, Read, Write, AskUserQuestion, WebSearch, mcp__exa__web_search_exa, mcp__exa__crawling_exa, mcp__firecrawl__firecrawl_scrape, mcp__firecrawl__firecrawl_crawl
```

- [ ] **Step 2: Edit `variants/open/SKILL.md` `allowed-tools` line**

Find line 6 in `variants/open/SKILL.md`:
```
allowed-tools: Bash, Read, Write, AskUserQuestion, WebSearch, mcp__exa__web_search_exa, mcp__exa__crawling_exa
```
Replace with:
```
allowed-tools: Bash, Read, Write, AskUserQuestion, WebSearch, mcp__exa__web_search_exa, mcp__exa__crawling_exa, mcp__firecrawl__firecrawl_scrape, mcp__firecrawl__firecrawl_crawl
```

- [ ] **Step 3: Verify both files**

Run:
```bash
grep "allowed-tools" SKILL.md variants/open/SKILL.md
```
Expected output (both lines must contain `mcp__firecrawl__firecrawl_scrape`):
```
SKILL.md:allowed-tools: Bash, Read, Write, AskUserQuestion, WebSearch, mcp__exa__web_search_exa, mcp__exa__crawling_exa, mcp__firecrawl__firecrawl_scrape, mcp__firecrawl__firecrawl_crawl
variants/open/SKILL.md:allowed-tools: Bash, Read, Write, AskUserQuestion, WebSearch, mcp__exa__web_search_exa, mcp__exa__crawling_exa, mcp__firecrawl__firecrawl_scrape, mcp__firecrawl__firecrawl_crawl
```

- [ ] **Step 4: Commit**

```bash
git add SKILL.md variants/open/SKILL.md
git commit -m "feat: add Firecrawl MCP tools to allowed-tools"
```

---

### Task 2: Insert Step 2b section in main SKILL.md

**Files:**
- Modify: `SKILL.md` — insert after line 424 (the `---` separator after Step 2's options block, before `## Judge Agent`)

- [ ] **Step 1: Insert Step 2b section**

Find this exact text in `SKILL.md` (lines 424–426):
```
---

## Judge Agent: Synthesize All Sources
```

Replace with:
```
---

## STEP 2b: DEEP FETCH TOP WEB RESULTS

After Step 2 (Exa MCP + WebSearch) completes, fetch full article content for the top web URLs using Firecrawl MCP.

**Skip this step entirely if `--no-native-web` was passed.**

**Fetch count by depth:**
- `--quick`: fetch top **3** URLs
- default: fetch top **5** URLs
- `--deep`: fetch top **8** URLs

**URL selection:**
1. Collect all URLs returned by Step 2 (Exa MCP results first, then WebSearch results)
2. Rank by Exa relevance score descending; if no Exa scores available, use return order
3. Remove any URLs from excluded domains: `reddit.com`, `x.com`, `twitter.com` (already covered by the script)
4. Take the top N URLs per depth profile above

**Fetch execution:**
- Call `mcp__firecrawl__firecrawl_scrape` for each selected URL
- Run fetches in parallel where possible
- Truncate each fetched markdown response to **1500 characters** before passing to synthesis
- If Firecrawl returns an error or times out for any URL: skip that URL silently, do not block synthesis

**In synthesis:**
- Firecrawl-enriched articles count as web sources — no separate output section
- Weight full-content articles higher than snippet-only results (more signal, richer content)
- Source domain names from enriched articles appear on the `🌐 Web:` stats line as normal

---

## Judge Agent: Synthesize All Sources
```

- [ ] **Step 2: Verify the section was inserted correctly**

Run:
```bash
grep -n "STEP 2b\|Step 2b\|firecrawl_scrape\|Judge Agent" SKILL.md
```
Expected output must show Step 2b appearing before Judge Agent:
```
381:## STEP 2: DO WEBSEARCH AFTER SCRIPT COMPLETES
NNN:## STEP 2b: DEEP FETCH TOP WEB RESULTS
NNN:- Call `mcp__firecrawl__firecrawl_scrape` for each selected URL
NNN:## Judge Agent: Synthesize All Sources
```
(line numbers will vary — what matters is the order)

- [ ] **Step 3: Commit**

```bash
git add SKILL.md
git commit -m "feat: add Step 2b Firecrawl deep fetch to SKILL.md"
```

---

### Task 3: Update Security & Permissions section and deploy

**Files:**
- Modify: `SKILL.md` lines 864 (Security & Permissions bullet)

- [ ] **Step 1: Update the web search bullet in Security & Permissions**

Find this exact line in `SKILL.md`:
```
- Optionally sends search queries to Brave Search API, Parallel AI API, or OpenRouter API for web search
```

Replace with:
```
- Optionally sends search queries to Brave Search API, Parallel AI API, or OpenRouter API for web search
- Fetches full article content via Firecrawl MCP (`api.firecrawl.dev`) for top web results — no API key required, no user data sent
```

- [ ] **Step 2: Verify the Security section**

Run:
```bash
grep -n "Firecrawl\|firecrawl" SKILL.md
```
Expected: at least 3 matches — `allowed-tools` line, Step 2b section, and Security & Permissions bullet.

- [ ] **Step 3: Deploy via sync.sh**

Run from the project root:
```bash
bash scripts/sync.sh
```
Expected output (last lines):
```
--- Syncing to /Users/developer/.claude/skills/last30days ---
  Copied 41 modules
  Import check: OK

--- Syncing to /Users/developer/.agents/skills/last30days ---
  Copied 41 modules
  Import check: OK

--- Syncing to /Users/developer/.codex/skills/last30days ---
  Copied 41 modules
  Import check: OK

Sync complete.
```

- [ ] **Step 4: Verify deployed SKILL.md contains Firecrawl**

Run:
```bash
grep "firecrawl" ~/.claude/skills/last30days/SKILL.md
```
Expected: at least 2 matches (allowed-tools line + Step 2b content).

- [ ] **Step 5: Final commit**

```bash
git add SKILL.md
git commit -m "feat: add Firecrawl to Security & Permissions; deploy"
```

---

## Self-Review Checklist

- [x] **allowed-tools** — Task 1 adds both tools to both files
- [x] **Step 2b placement** — Task 2 inserts after Step 2, before Judge Agent
- [x] **Depth-adaptive counts** — 3/5/8 for quick/default/deep specified in Step 2b text
- [x] **URL selection logic** — Exa relevance ranking, excluded domains, take top N
- [x] **1500 char truncation** — explicitly stated in fetch execution
- [x] **Failure handling** — skip silently, don't block synthesis
- [x] **--no-native-web flag** — skip condition stated at top of Step 2b
- [x] **Synthesis weighting** — full-content weighted higher than snippet-only
- [x] **Stats line** — unchanged format, source names appear on 🌐 Web: line
- [x] **Security & Permissions** — Task 3 adds Firecrawl bullet
- [x] **Diagnostics (--diagnose)** — spec called for a `--diagnose` line; this is rendered by `scripts/lib/quality_nudge.py` and `env.py` in the Python subprocess, not SKILL.md. Since Firecrawl is MCP-only (agent-layer), it cannot be detected by the Python subprocess. Omitting from `--diagnose` is correct — the spec note was aspirational but architecturally unsound. No task needed.
- [x] **Deploy** — Task 3 runs sync.sh and verifies deployed copy
- [x] **No placeholders** — all steps contain exact text/commands
