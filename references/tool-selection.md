# Research Tool Selection Guide

Tool selection follows a pragmatic decision tree. Use the simplest **available** tool that answers the question. `web_search` / `web_fetch` here mean whatever search and page-fetch tools the current harness exposes.

1. **Discovery and reads (default):** host `web_search` / `web_fetch` — any available search or fetch.
2. **Mapping, crawl, schema, JS-browser (optional):** Firecrawl CLI or equivalent — only if already installed and authenticated, and only when host fetch cannot do the job.
3. **Multi-source research:** host search for discovery, host fetch for deep-reads. Specialized scrape/map only if needed and present.

Never block research for Firecrawl setup. Missing Firecrawl is not a failure of this skill.

If Firecrawl commands are used, set `FIRECRAWL_OUTPUT_DIR` to the active research session first:

```bash
export FIRECRAWL_OUTPUT_DIR=".researches/<timestamp>/firecrawl/raw"
```

## Decision Tree

```text
What do you need?
|
|-- Quick fact or discovery
|   |-- Use: host web_search
|       |-- Need deeper content from a result?
|           |-- host web_fetch the URL
|       |-- Need dual representation AND a scrape backend is already available?
|           |-- optional Firecrawl scrape --only-main-content
|
|-- Find information (don't know exact URLs)
|   |-- Use: host web_search, then host web_fetch on 1-2 URLs
|   |   |-- Do not switch search backends with the same query
|   |-- Need structured extraction AND a schema backend is already available?
|       |-- optional firecrawl agent with --schema; else fetch and parse
|
|-- Explore a website structure
|   |-- Host fetch has no map equivalent
|   |-- If Firecrawl (or equivalent) is already available: firecrawl map
|       |-- Need content from discovered pages? crawl or scrape
|   |-- Else: search `site:<domain> <topic>` and fetch listed URLs; note the gap
|
|-- Extract from known URLs
|   |-- Single or few URLs
|   |   |-- Use: host web_fetch
|   |       |-- Optional Firecrawl scrape if already available and host fetch is noisy
|   |       |-- Optional firecrawl agent --schema if structured JSON is required
|   |
|   |-- Many URLs from same site
|       |-- Host fetch each selected URL, or optional firecrawl crawl if present
|
|-- Complex multi-step research
    |-- Chain host searches first
    |-- Optional firecrawl agent --wait / browser only if that backend is present
```

## Host tools (default)

| Goal | Primary tool |
| --- | --- |
| Search web | Host `web_search` (or equivalent) |
| Read a known page | Host `web_fetch` (or equivalent) |
| Persist artifacts | `save-web-research.ts` |
| Consolidate | `consolidate-research.ts --auto-session` |

## Optional Firecrawl commands

Only when that CLI is already available. See SKILL.md for when Phase 2 is useful.

| Goal | Primary Command | Typical Flags |
| --- | --- | --- |
| Alternate search | `firecrawl search "<query>"` | `--limit`, `--sources`, `--tbs`, `--scrape` |
| Scrape known page | `firecrawl scrape "<url>"` | `--format markdown,html`, `--only-main-content`, `--wait-for` |
| Discover site URLs | `firecrawl map "<url>"` | `--search`, `--limit`, `--sitemap` |
| Crawl section | `firecrawl crawl "<url>"` | `--wait`, `--limit`, `--max-depth`, `--include-paths` |
| Structured extraction | `firecrawl agent "<prompt>"` | `--schema`, `--schema-file`, `--urls`, `--wait` |
| Interactive browsing | `firecrawl browser ...` | `launch-session`, `execute`, `list`, `close` |
| GitHub repo docs | `archive-github-repo-docs.ts` | Keep markdown from local/raw, HTML from blob page |

## Patterns

### 1. Search First (Default Host Pattern)

Start with host `web_search` for discovery, then host `web_fetch` for deep-reads:

```
# Phase 1: Discovery
web_search: "site:docs.example.com auth"
→ identify 2-5 most relevant URLs from results
→ web_fetch the 1-2 best URLs

# Phase 2: Optional specialized extraction (only if host fetch is insufficient
# and Firecrawl is already available)
firecrawl scrape "<best-url>" --only-main-content -o "$FIRECRAWL_OUTPUT_DIR/auth.md"
```

Optional Firecrawl search when that CLI is present and host search is not enough:

```bash
firecrawl search "site:docs.example.com auth" --limit 10 --json \
  -o "$FIRECRAWL_OUTPUT_DIR/search.json"
```

**Key efficiency tip:** When `web_search` returns 5 results, scan titles and snippets first.
Do not read all results in full. Select only the 1-2 most relevant URLs and
`web_fetch` those. Save full discovery results with `save-web-research.ts` for
reproducibility, even if you only deep-read a subset.

```text
❌ web_search → read all 5 full results → try to synthesize from noisy pages
✅ web_search → scan titles + snippets → select 1-2 best → web_fetch → synthesize
```

### 2. Map then Scrape

```bash
firecrawl map "https://docs.example.com" --search "webhook" --json \
  -o "$FIRECRAWL_OUTPUT_DIR/map.json"
firecrawl scrape "https://docs.example.com/reference/webhooks" --only-main-content \
  -o "$FIRECRAWL_OUTPUT_DIR/webhooks.md"
```

### 3. Controlled Crawl

```bash
firecrawl crawl "https://docs.example.com" --include-paths /docs --exclude-paths /blog \
  --limit 100 --max-depth 2 --wait --json -o "$FIRECRAWL_OUTPUT_DIR/crawl.json"
```

### 4. Structured Agent Extraction

```bash
firecrawl agent "Extract pricing tiers and limits" --schema-file schema.json --wait --json \
  -o "$FIRECRAWL_OUTPUT_DIR/pricing.json"
```

### 5. Browser Escalation for Dynamic Pages

```bash
firecrawl browser "open https://example.com"
firecrawl browser "snapshot -i"
firecrawl browser "click @e5"
firecrawl browser "scrape" -o "$FIRECRAWL_OUTPUT_DIR/dynamic.md"
firecrawl browser close
```

### 6. Verbatim Documentation Capture

```bash
firecrawl scrape "https://docs.example.com/reference/auth" --format markdown \
  -o ".researches/<timestamp>/documentation/markdown/auth.md"
firecrawl scrape "https://docs.example.com/reference/auth" --format html \
  -o ".researches/<timestamp>/documentation/html/auth.html"
```

Screenshot evidence:

```bash
firecrawl scrape "https://docs.example.com/reference/auth" --format screenshot --json \
  -o ".researches/<timestamp>/firecrawl/raw/auth-screenshot.json"
curl -L "$(jq -r '.screenshot' .researches/<timestamp>/firecrawl/raw/auth-screenshot.json)" \
  -o ".researches/<timestamp>/documentation/screenshots/auth.png"
```

If the installed CLI lacks a direct full-page screenshot flag, escalate to `firecrawl browser ...` after confirming the command surface with `firecrawl browser --help`.

### 7. GitHub Repository Docs (Hybrid Pattern)

When documentation lives as tracked files in a GitHub repository:

- Use the local clone or `raw.githubusercontent.com` as the canonical markdown source.
- Save rendered HTML from the GitHub blob page for source fidelity.
- Use Firecrawl screenshot capture on the blob page only when visual evidence is useful.
- Skip screenshots by default for source code files; keep them for markdown/docs pages.
- Do not treat blob-page `firecrawl scrape` markdown as verbatim file content.

```bash
npx tsx .agents/skills/research-online/scripts/archive-github-repo-docs.ts \
  --session-dir ".researches/<timestamp>" \
  --github-repo "owner/repo" \
  --branch "main" \
  --repo-dir "/abs/path/to/clone" \
  --screenshot-mode docs-only \
  --file "README.md" \
  --file "docs/tool-reference.md"
```

If browser fallback is required for GitHub pages, prefer explicit `firecrawl browser execute --node ...` or `--python ...`. Avoid assuming the agent-browser shorthand or `launch-session --json` will behave consistently across CLI versions.

## Cost and Reliability Guardrails

1. Start with host `web_search` for discovery and host `web_fetch` for known URLs.
2. Use a specialized backend only when you need mapping, crawl, schema extraction, or JS-browser work that host fetch cannot do — and only if it is already available.
3. If Firecrawl is in use: `search`/`scrape`/`map` before `agent`.
4. Add explicit limits (`--limit`, `--max-depth`) for crawls.
5. Use `--max-age` for cache reuse during iteration (specialized backends).
6. Save output to files (`-o` or `save-web-research.ts`) and inspect incrementally.
7. Use `agent` only when deterministic commands cannot complete the task.
8. For GitHub repo docs, do not spend specialized-backend credits on canonical markdown capture; use raw URLs or the archival helper.
9. Host `web_search` + `web_fetch` is a complete default path, not a fallback.
10. Multi-source research: host search for discovery, host fetch for the best sources; Phase 2 only if needed.
11. Formulate specific, disambiguated queries before searching — see SKILL.md > Research Methodology > Query Formulation for heuristics.
12. Evaluate source credibility before deep-reading — prioritize results matching 3+ strong signals (official docs, recent, code examples, first-party, canonical). See SKILL.md > Research Methodology > Source Evaluation.
