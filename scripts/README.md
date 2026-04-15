# `scripts/hkex_scraper.py` — Python API

A standalone Python module for scraping HKEX Application Proof (A1) filings and extracting structured fields from prospectus chapters. Designed for financial analysts tracking Hong Kong pre-IPO life-science pipelines, but generic enough for any A1-filer workflow.

## What it does

1. **Feed ingestion** — pulls the HKEX Application Proof JSON feed (`app_{YYYY}_sehk_{e|c}.json`) for a given calendar year and unifies EN + CN records by `id`.
2. **Life-science pre-filter** — matches EN/CN keywords and `-B` (Ch.18A biotech) / `-P` (Ch.18C specialist tech) suffixes to surface healthcare candidates.
3. **Multi-Files TOC parsing** — resolves each candidate's prospectus into its chapter PDF URLs (`SUMMARY`, `BUSINESS`, `FINANCIAL INFORMATION`).
4. **Chapter download** — fetches only the three relevant chapters (~5 MB per candidate), not the full 30-60 MB prospectus.
5. **Hybrid field extraction**:
   - **`pdfplumber` table parser** for deterministic financial numbers (FY revenue, net income, cash) from the FINANCIAL chapter
   - **Regex on pdfplumber SUMMARY text** for sponsor bank names (templated text, regex is reliable)
   - **Firecrawl MCP (`mcp__firecrawl__scrape`)** with a JSON schema for narrative fields (shareholder structure, business model, sector, lead asset) — LLM-backed extraction is more accurate than regex for prose-heavy semantic fields
6. **Classification** — buckets candidates into `new` / `refresh` / `skip` against an existing tracker Excel by normalized company name.

## Requirements

**Python 3.9+** (3.9 compatibility is strict — no `X | Y` type unions, no `match/case`, no lowercase generics).

```bash
pip install -r requirements.txt
```

Installs `requests`, `beautifulsoup4`, `pdfplumber`, `openpyxl`.

## Firecrawl MCP setup (optional but recommended)

Narrative field extraction (shareholder structure / business model / sector / lead asset) uses Firecrawl's schema-backed scrape endpoint. Without it, those four fields come back as `None` and must be filled manually — financial numbers and sponsor names still work.

1. Register at [firecrawl.dev](https://firecrawl.dev) (free tier: 500 credits/month, no credit card).
2. Copy your API key from the dashboard (`fc-...`).
3. Register the MCP server at user scope in Claude Code:

   ```bash
   claude mcp add firecrawl -s user --env FIRECRAWL_API_KEY=fc-your-key-here -- npx -y firecrawl-mcp
   ```

4. Restart Claude Code. `claude mcp list` should show `firecrawl: ✓ Connected`.

Typical monthly usage (12 candidates × 1 scrape each) costs ~60 credits — well within the free tier.

## Python API

### Canonical flow

```python
import sys
sys.path.insert(0, "path/to/hkex-a1-pipeline/scripts")
import hkex_scraper as hs

# 1. Pull all HKEX A1 records for the year, pre-filter to life-science
cands = hs.fetch_lifesci_candidates(year=2026)
passed, dropped = hs.filter_candidates(cands)

# 2. Classify against an existing tracker (optional — skip if starting fresh)
master = hs.load_master_tracker("tracker/a1_pipeline_tracker.xlsx")
buckets = hs.classify_candidates(passed, master)

# 3. For each NEW/REFRESH candidate, download chapters + extract deterministic fields
cache_dir = hs.create_cache_dir()
staging = []
for cand in buckets["new"] + buckets["refresh"]:
    hs.fetch_targeted_chapters(cand, cache_dir)
    row = hs.extract_fields_from_chapters(cand, target_fy="FY25")
    staging.append(row)

# 4. (Optional) Call Firecrawl MCP to fill narrative fields F/G/H/M
#    This step runs in the Claude Code agent tool layer, not in Python:
#
#    for row in staging:
#        summary_url = row["candidate"]["chapter_urls"].get("summary")
#        if summary_url:
#            fc = mcp__firecrawl__scrape(
#                url=summary_url,
#                formats=["json"],
#                jsonOptions={
#                    "schema": hs.FIRECRAWL_NARRATIVE_SCHEMA,
#                    "prompt": hs.FIRECRAWL_NARRATIVE_PROMPT,
#                },
#                onlyMainContent=True,
#            )
#            fc_data = (fc.get("data") or {}).get("json") or fc.get("json") or {}
#            hs.apply_firecrawl_narrative(row, fc_data)

# 5. On success, clean up the transient PDF cache
hs.cleanup_cache_dir(cache_dir, had_failures=False)
```

### Pure-Python mode (no Firecrawl)

If you skip step 4, `staging` rows come back with `F/G/H/M` as `None` and `_qc_flags` containing `firecrawl_pending_col_F/G/H/M`. Financial numbers (J/K/L), sponsor (I), filing date / names (C/D/E), and highlights (N) are all fully populated. You can fill F/G/H/M manually or plug in your own extraction.

### Standalone CLI

For quick smoke testing without Claude Code:

```bash
python hkex_scraper.py 2026                          # feed + filter, no master
python hkex_scraper.py 2026 /path/to/tracker.xlsx    # feed + filter + classify
```

## Exported API

| Name | Purpose |
|---|---|
| `fetch_hkex_feed(year, board='sehk', lang='en')` | Raw JSON feed fetch with retry / rate limit |
| `fetch_lifesci_candidates(year, since=None, include_gem=False)` | Unified EN+CN candidate records with TOC URLs |
| `filter_candidates(candidates)` | Partition into `(passed, dropped)` via EN kw + CN kw + `-B`/`-P` suffix |
| `load_master_tracker(xlsx_path)` | Read-only snapshot of an existing tracker for classification lookup |
| `classify_candidates(candidates, master)` | Bucket candidates into `{new, refresh, skip}` by normalized name |
| `fetch_targeted_chapters(cand, cache_dir)` | Parse Multi-Files TOC, download SUMMARY/BUSINESS/FINANCIAL chapter PDFs, populate `cand['chapter_urls']` |
| `extract_fields_from_chapters(cand, target_fy='FY25')` | Build `row_draft` dict with C/D/E/I/J/K/L/N filled; leaves F/G/H/M = None with pending flags for Firecrawl |
| `apply_firecrawl_narrative(staging_row, fc_data)` | Merge a Firecrawl scrape result into staging row F/G/H/M with off-enum guard for sector |
| `FIRECRAWL_NARRATIVE_SCHEMA` | Module-level JSON schema dict: 4 narrative fields with enum constraints |
| `FIRECRAWL_NARRATIVE_PROMPT` | Module-level extraction prompt (instructs LLM on VIE detection, main-business sector) |
| `create_cache_dir(run_date=None)` / `cleanup_cache_dir(...)` | Manage transient per-run PDF cache under `checkpoints/` (gitignored) |

## Configuration

| Env var | Default | Purpose |
|---|---|---|
| `A1_CHECKPOINT_DIR` | `hkex-a1-pipeline/checkpoints/` (sibling of `scripts/`) | Where to put transient chapter PDF downloads. The repo's `.gitignore` excludes `checkpoints/`, so downloaded PDFs are never committed. |
| `FIRECRAWL_API_KEY` | (none) | Required for Firecrawl MCP. Set via `claude mcp add firecrawl --env FIRECRAWL_API_KEY=...` — NOT via shell env or a committed file. |

## Rate limits and retry

All HTTP calls respect an internal 0.8-second minimum gap between HKEX requests, with exponential backoff (3 attempts, starting at 2 seconds) on 429/5xx responses. The module maintains a `warnings_log` list you can inspect after a run for any soft failures.

## Not included

This module is the scraping / extraction layer only. The full monthly workflow (Gate 1 candidate approval, Gate 2 diff preview, Phase 5 save invariants — Arial 8 black font, zero fills, dd/mm/yyyy date format, dedup, DESC sort) lives in a sibling Claude Code skill at [`../.claude/skills/a1-pipeline-update/SKILL.md`](../.claude/skills/a1-pipeline-update/SKILL.md), shipped in this same repo. Open the repo in [Claude Code](https://claude.com/claude-code) and the skill auto-activates — see the repo's top-level [`README.md → Setup & Usage`](../README.md#setup--usage) for the end-to-end run walkthrough. The Python API documented above is the layer the skill calls into; you can also plug in your own workflow wrapper on top if you prefer.

## License

MIT — see repository LICENSE.
