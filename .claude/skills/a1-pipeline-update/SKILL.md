---
name: a1-pipeline-update
description: >
  Monthly update workflow for the HKEX A1 pre-IPO pipeline tracker
  (life sciences companies that have filed an A1 listing application
  on the Hong Kong Stock Exchange). Given a new monthly tracker Excel
  with incomplete rows at the top, this skill fills in Chinese name,
  shareholder structure, sector, sponsor, FY financials, lead asset
  and highlights via HKEX prospectus lookups and web research, handles
  re-filed companies via data recovery from the prior month's file,
  and preserves Bloomberg-style cell formatting on save. Narrative
  fields (shareholder structure, business model, sector, lead asset)
  are extracted via Firecrawl MCP schema-backed extraction on the
  SUMMARY chapter URL; financial numbers stay on deterministic
  pdfplumber table parsing, and sponsor bank names stay on regex. When
  the user invokes the skill without supplying a pre-typed Excel,
  Phase 0 auto-scrapes the HKEX Application Proof JSON feed,
  pre-filters life-science candidates, and processes the Summary /
  Business / Financial Information chapters per candidate before the
  usual 5-phase enrichment runs. Use whenever the user mentions: "A1
  update", "update A1 tracker", "HKEX A1", "scan HKEX for new A1",
  "pull latest A1 filings", "Phase 0 scrape", "抓一下 HKEX",
  "pre-IPO tracker", "pipeline tracker update", "月度 A1 更新",
  "更新 A1 tracker", "A1 申报", "港股 pre-IPO 跟踪", or provides a
  new monthly A1 filing tracker Excel file with empty detail columns.
---

# HKEX A1 Pipeline Tracker Monthly Update

## When to Use
When user asks to update the HKEX A1 pipeline tracker, mentions "A1 update", or provides a new monthly A1 filing tracker Excel.

## Overview
Monthly workflow to update a pre-IPO A1 filing tracker for HKEX-listed life science companies. The user provides a new Excel with recently filed companies (rows with company name + filing date but empty details), and this skill fills in the missing data.

## Prerequisites

- **Invoke this skill from the root of the `hkex-a1-pipeline` repo** so that all relative paths (`tracker/`, `archive/`, `raw/`, `scripts/`) resolve correctly.
- **Python 3.9+** with `requests`, `beautifulsoup4`, `pdfplumber`, `openpyxl` — see [`scripts/requirements.txt`](../../../scripts/requirements.txt). Install with `pip install -r scripts/requirements.txt` from the repo root. Needed for Phase 0 (HKEX JSON feed scrape + chapter PDF parse) and Phase 5 (openpyxl writes).
- **Firecrawl MCP** configured at user scope (`claude mcp add firecrawl ...`) — required for Mode B to extract narrative fields F/G/H/M from SUMMARY chapter URLs via LLM-backed schema extraction. Without it, Mode B still runs but F/G/H/M land as `None` with `firecrawl_pending_col_X` QC flags for manual fill in Gate 2.

## Repo file layout (all paths relative to repo root)

| Purpose | Path |
|---|---|
| Live master tracker (overwrite on every update, never version-suffix) | `tracker/a1_pipeline_tracker.xlsx` |
| Historical monthly snapshots | `archive/A1 pipeline_<Mon> <YYYY>_update_v<N>.xlsx` |
| Mode A input drop (user-provided monthly Excel) | `raw/<filename>.xlsx` |
| Canonical Python scraper (1500 LOC, single source of truth) | `scripts/hkex_scraper.py` |
| Phase 0 transient PDF cache | `scripts/checkpoints/YYYY-MM-DD_a1_hkex_pdfs/` (auto-created, `.gitignore`d; override via env var `A1_CHECKPOINT_DIR`) |
| Repo-local operational rules (authoritative) | `CLAUDE.md` |

**File discipline** — the tracker is a **single live file** at `tracker/a1_pipeline_tracker.xlsx`. Never version-suffix the live file. On every successful save, drop a snapshot into `archive/` with the next `A1 pipeline_<Mon> <YYYY>_update_v<N>.xlsx` suffix, then overwrite the live file. Full commit-discipline rules are in [`CLAUDE.md`](../../../CLAUDE.md) → `## Update Workflow → What to commit`.

## Inputs

The skill runs in one of two modes, depending on what the user supplies. Both modes converge at Phase 4 (Gate 2 diff preview) and Phase 5 (write + QC + archive + commit).

**Mode A — User provides a monthly Excel** (manual / legacy path):
- The user drops a `raw/<filename>.xlsx` with new rows at the top populated in col D (company name) + col C (filing date) only — all other cells blank.
- Skill reads `tracker/a1_pipeline_tracker.xlsx` as the "old file" for dedup/recovery lookup, and **skips Phase 0** — jump straight to Phase 1.

**Mode B — No Excel, full auto-scrape** (default for routine monthly runs):
- User invokes the skill with phrasing like `scan HKEX for new A1s`, `update A1 tracker`, `pull latest A1 filings`, `A1 update`, or `抓一下 HKEX` — no input file supplied.
- Skill enters **Phase 0** (described below), which pulls the HKEX Application Proof JSON feed, classifies candidates, runs Gate 1 for approval, downloads SUMMARY / BUSINESS / FINANCIAL chapter PDFs, and extracts field values via a hybrid `pdfplumber` + regex + Firecrawl MCP pipeline. The resulting staging rows feed directly into Phase 1.

**Target FY**: default `FY25` for calendar-year 2026 runs; confirm or override in Phase 1 before processing financial numbers.

## Phase 0 — HKEX Auto-Scrape (optional, user-triggered)

**Run Phase 0** when the user says "scan HKEX", "pull new A1s", "抓一下 HKEX", or asks for a fresh run without providing an Excel with pre-typed new rows. **Skip Phase 0 entirely** when the user supplies a new file with new rows already typed — jump straight to Phase 1.

Phase 0 produces an in-memory list of `staging_row` dicts that Phase 1 consumes in place of "candidate rows from the new Excel". Phase 0 itself writes nothing to disk except transient PDF chapter cache under `checkpoints/YYYY-MM-DD_a1_hkex_pdfs/`.

### Phase 0 inputs
- `year`: `int`, default = current calendar year
- `since`: `datetime.date`, default = most recent filing_date in master + 1 day (scan only new filings)
- `include_gem`: `bool`, default `False`
- `target_fy`: inherited from skill-level default (FY25)

### Phase 0 workflow (10 steps)

1. **Sanity import** — `import requests, bs4, pdfplumber, openpyxl, pandas`. On `ImportError` print missing lib and stop.
2. **Fetch JSON feeds** — `fetch_hkex_feed(year, lang="en")` and `..."c"` from `https://www1.hkexnews.hk/ncms/json/eds/app_{YYYY}_sehk_{e|c}.json`. Keep records whose `ls` has `nF` starting with `"Application Proof"`. Join EN+CN by `id`.
3. **Keyword pre-filter** — `filter_candidates(...)`: EN keyword OR CN keyword OR `-B`/`-P` suffix match. Typical hit rate ~22% (58/265 on 2026 SEHK).
4. **Load master tracker** — read-only via `openpyxl.load_workbook(..., data_only=True)`. Handles mixed `datetime`/`str` dates in col C; `†` lapsed markers; drifted sector vocab.
5. **Classify candidates** — `classify_candidates(...)` buckets into NEW / REFRESH / SKIP. Duplicate master rows (e.g., Sirius r6 + r92) pick the most recent and annotate `multi_row_match` QC flag.
6. **GATE 1** — render the candidate list below and WAIT for user approval. No PDFs fetched until approval.
7. **Create cache dir** — `create_cache_dir()` under `checkpoints/YYYY-MM-DD_a1_hkex_pdfs/` (with `_2`/`_3` suffix on collision from aborted prior run).
8. **Per-approved-candidate extraction loop** — for each candidate, run the following three sub-steps in order:
   - **(8a)** `fetch_targeted_chapters(cand, cache_dir)` — parse Multi-Files TOC, match SUMMARY/BUSINESS/FINANCIAL, download the three chapter PDFs (~5 MB total per candidate). Also populates `cand['chapter_urls']` with the remote URLs for each slot — Firecrawl needs these in step 8c.
   - **(8b)** `staging_row = extract_fields_from_chapters(cand, target_fy=target_fy)` — pdfplumber table parser on FINANCIAL for J/K/L, regex on pdfplumber SUMMARY text for I (sponsor), assembled col N with FY marker. **Leaves F/G/H/M as `None`** and tags the row with `firecrawl_pending_col_F/G/H/M` QC flags. This step is deterministic, offline, and cheap — if Firecrawl is unavailable for any reason, the skill can still proceed to Phase 4 and let the user fill F/G/H/M manually.
   - **(8c)** **Firecrawl narrative extraction for F/G/H/M** — invoke `mcp__firecrawl__scrape` on the candidate's SUMMARY chapter URL (from `cand['chapter_urls']['summary']`) with `formats=["json"]` and `jsonOptions={"schema": hs.FIRECRAWL_NARRATIVE_SCHEMA, "prompt": hs.FIRECRAWL_NARRATIVE_PROMPT}`. The MCP call returns a dict under `.json` with keys `shareholder_structure / business_model / sector / lead_asset`. Pass that dict to `hs.apply_firecrawl_narrative(staging_row, fc_data)` which maps it onto F/G/H/M, sets provenance to `firecrawl:SUMMARY`, sets confidence to `high`, and clears the pending QC flags for any field that came back non-null. Fields Firecrawl returned null for remain as `None` + pending flag — Phase 4 will surface them for manual fill. See [Firecrawl MCP Integration](#firecrawl-mcp-integration-cols-fghm) for schema/prompt details, cost, and fallback behavior.
   - **(8d)** **Robustness tag classification** — after `apply_firecrawl_narrative()` returns, call `hs.auto_classify_fg_robustness(staging_row)` to assign a Phase-0-determinable F+G robustness tag (one of `verified_fg_prospectus` / `single_source_prospectus_fg` / `not_found_fg` — see [Robustness Tag Vocabulary](#robustness-tag-vocabulary) below). The function reads `_provenance` / `_confidence` / `row_draft` for cols F and G and mutates the staging row in place, appending the chosen tag to `_qc_flags` and the canonical suffix to `row_draft["N"]`. The web-fallback tags (`web_cross_checked_fg` / `single_source_family_fg` / `conflicting_fg`) cannot be auto-determined from Python state — they're set by the agent driver via `hs.apply_fg_robustness_tag(staging_row, tag)` after Phase 3 web fallback completes. Phase 4's robustness counter aggregates over whichever tag is current when Gate 2 runs.
9. **Hand off to Phase 1** — pass `staging_rows` list in memory. Phase 1 treats each entry as a candidate row; NEW rows get appended, REFRESH rows follow `fields_to_refresh`.
10. **Cache cleanup** — on successful Phase 5 write, `cleanup_cache_dir(cache_dir, had_failures=False)`. On abort or mid-run failure, preserve cache and print path.

### GATE 1 — Candidate list approval format

```
===================================================================
HKEX Phase 0 — Candidate List (Gate 1)
-------------------------------------------------------------------
Source feed   : https://www1.hkexnews.hk/ncms/json/eds/app_2026_sehk_e.json
Feed uDate    : 15/04/2026
Master file   : A1 pipeline_Apr 2026_update_v2.xlsx  (163 rows)
Master cutoff : latest filing_date = 02/04/2026
Target FY     : FY25
Scan window   : filings after 02/04/2026
Pre-filter    : 31 → 12 life-sci (EN kw=9, CN kw=5, suffix=7, union=12)
-------------------------------------------------------------------

=== NEW (<n> rows — not in master) ===
  1. <dd/mm/yyyy>  id=<id>  <Company Name - B>
                   reason: EN kw+suffix; CN: <中文名>
                   TOC: yes  |  chapters: SUMMARY, BUSINESS, FIN
  ...

=== REFRESH (<r> rows — already in master) ===
  1. <dd/mm/yyyy>  id=<id>  <Company>
                   master row <N> (existing date <old_date>)
                   delta       +<N> days (re-filed)
                   refresh     C, J, K, L, N   (fy_refresh_needed=True)
                   keep        D, E, F, G, H, I, M

=== SKIP (<s> rows — already current) ===
  1. <Company> (master r<N>, same date)
  ...

=== LLM DOWNGRADED (optional — pre-filter hit but SUMMARY says non-LS) ===
  1. <Company> — <reason>; override with `keep 1` if you disagree

=== DROPPED BY PRE-FILTER (<d> rows) ===
  (first 5 shown; `show dropped` for full list)
  ...

-------------------------------------------------------------------
Estimated PDF work: <N> candidates × 3 chapters = <M> files (~<X> MB)

Proceed?
  yes                 — start PDF extraction
  drop <row>          — remove a NEW/REFRESH entry
  add <hkex_id>       — force-include a DROPPED entry
  keep <llm_row>      — override an LLM downgrade
  show dropped        — list all dropped
  abort               — exit cleanly, no cache
===================================================================
```

**Wait for explicit user response**. Accepted replies: `yes` / `drop N` / `add <id>` / `keep <N>` / `show dropped` / `abort`. Never proceed without user input.

### Phase 0 staging contract with Phase 1

Phase 0 hands Phase 1 a list of `staging_row` dicts (in-memory only, no intermediate Excel file):

```python
staging_row = {
    "target_bucket": "new" | "refresh",
    "master_row_idx": Optional[int],           # filled for refresh
    "fields_to_refresh": List[str],            # e.g., ["C"] or ["C","J","K","L","N"]
    "row_draft": Dict[str, Any],               # keyed by col letter C..N
    "candidate": Dict,                          # raw HKEX record, for audit trail
    "pdf_status": str,                          # ok / no_toc / partial / failed
    "_provenance": Dict[str, str],              # {"J": "pdfplumber:FINANCIAL", ...}
    "_confidence": Dict[str, str],              # {"H": "medium", ...}
    "_qc_flags": List[str],                     # surfaced in Phase 4 diff
}
```

**Phase 1 behavior on staging rows**:
- **NEW** rows → append above the existing block (top of data region, same as normal "candidate rows" handling).
- **REFRESH** rows → locate `master_row_idx` and plan in-place updates only for the columns listed in `fields_to_refresh`. **Never overwrite non-empty master cells** with a staging value whose `_confidence == "low"`. Preserve lead asset / sector / highlights from the master row unless they are in `fields_to_refresh` AND master had them empty.
- Phase 2's "Map: Classify Each Candidate" bucketing is unchanged but operates on the staging list instead of "read empty rows from new Excel".

## Column Structure (A-N)
| Col | Field | Format |
|-----|-------|--------|
| A | Timestamp metadata | "Updated as of [Date]" |
| B | Status | Text |
| C | Latest A1 Filing | datetime (dd/mm/yyyy display) |
| D | Company Name | English, with suffix: -B (Ch.18A biotech), -P (Ch.18C tech) |
| E | Chinese Name | Traditional or Simplified |
| F | Shareholder Structure | H-share / Red Chip / VIE |
| G | Business Model / Clinical Stage | e.g., "Commercial-stage; 5 approved products" |
| H | Sector | Pharma/Biotech, MedTech, Services, Diagnostics/Tools, Consumer Health, CDMO |
| I | Sponsor | Bank name(s) |
| J | Latest FY Revenue (RMB m) | Number; "-" if pre-revenue; None if N/A |
| K | Latest FY Net Income (RMB m) | Number (negative for losses) |
| L | Latest FYE Cash (RMB m) | Number |
| M | Lead Asset / Business | One-liner on core product/pipeline |
| N | Highlights / Updates | Key investment points, FY data notes |

## The 5-phase pipeline

This is the only way to update the tracker. **Never skip Phase 4.** Never write to disk before the user confirms.

### Phase 1 — Inspection (read-only)

1. Load new Excel via `openpyxl.load_workbook()`.
2. Scan rows from top until reaching rows with complete data (cols E-M filled) — these are the candidates for this month.
3. Load old file (previous month's completed tracker) read-only for lookup.
4. Report:

```
New file: <path>
Old file: <path>
Target FY: FY25
  Candidate rows (E-M empty): N
  Existing rows (E-M filled): M
  Latest A1 filing date in new file: dd/mm/yyyy
```

### Phase 2 — Map: Classify Each Candidate (in-memory)

For each incomplete row, classify into one of three buckets:

```
For each candidate row:
  1. Search new file existing rows (E-M filled) for same company name.
     → match: DUPLICATE (company re-filed this month; old row to be deleted)
  2. Search old file for same company name.
     → match: DATA RECOVERY (re-filing from prior month; copy E-N forward)
  3. No match in either file.
     → TRULY NEW (needs web research in Phase 3)
```

**Name matching**: normalize by stripping `-B` / `-P` suffix, lowercasing, and collapsing whitespace before comparing. A match on normalized name is sufficient — don't require exact case.

Report classification counts:

```
  DUPLICATE (re-filed, delete old row):   d rows
  DATA RECOVERY (copy from old file):      r rows
  TRULY NEW (web research needed):         n rows
```

### Phase 3 — Assign: Fill In Data (in-memory)

Process each bucket:

**DUPLICATE**: Copy cols E-N from the matching existing row → new row. Mark the old duplicate row for deletion (apply deletes in **reverse row order** at write time to avoid index shifting). Check if financial data needs refreshing to target FY.

**DATA RECOVERY**: Copy cols E-N from the old file's matching row → new row. Re-verify FY financials are still target FY; if not, flag for refresh in Phase 3's web research step.

**TRULY NEW**: Fill cols E-N per the Column Structure table above. Tool priority:

1. **`mcp__firecrawl__scrape`** — first choice for company website, Chinese financial news (sina finance, cls.cn, phirda.com), press releases, and IR pages. Handles JS-rendered pages and reCAPTCHA-lite walls that break `WebFetch`. Use `formats=["markdown"]` for general reading, or `formats=["json"]` with a targeted schema when you know exactly which fields you need.
2. **`mcp__firecrawl__search`** — when the company has no known URL, use Firecrawl's search endpoint instead of `WebSearch` + `WebFetch` round-trips. Returns ranked hits with clean markdown in one call.
3. **`WebFetch`** — fallback only. Use it when Firecrawl credits are low, the target URL is a simple static page (PDF, plain HTML), or Firecrawl returns an error. `WebFetch` is free but 403s on most Chinese financial sites.
4. **`WebSearch`** — general discovery when you don't yet know which domain has the answer. Pipe promising hits into `mcp__firecrawl__scrape`, not `WebFetch`.

Sources in priority order: `hkexnews.hk` (prospectus PDFs — via Firecrawl scrape on the chapter URL with `FIRECRAWL_NARRATIVE_SCHEMA`) → company website → Chinese financial news → PitchBook / BioCentury. For the prospectus itself, you can skip the Phase 0 pdfplumber path entirely and call Firecrawl directly on the chapter URL; for TRULY NEW rows that arrive from the new Excel without Phase 0 having run, this is the fastest path to a filled F/G/H/M.

**Robustness tag classification (after extraction completes for each TRULY NEW row)**: same hook as Phase 0 step 8d — once F/G/H/M are filled (via prospectus, web fallback, or both), classify the row's F+G robustness per the [Robustness Tag Vocabulary](#robustness-tag-vocabulary) (§ Phase 4 below) and append the tag to `_qc_flags` + col N suffix. The tag drives Phase 4's robustness counter and may block Phase 5 (e.g., `not_found_fg` blocks until user `accept blank <row>`; `conflicting_fg` blocks until `keep prospectus <row>` / `keep <secondary> <row>`).

**Bounded-effort discipline** (`/web-research` Rule 10 borrowing): for any TRULY NEW row where the prospectus path fails, cap web-fallback at **3 distinct Tier-1 attempts** (e.g., firecrawl_search top-3 hits + IR page; OR HKEX archive index + cls.cn + Caixin). After 3 failures, stop, tag `not_found_fg`, and surface in Gate 2. Do not silently downgrade to lower-tier sources without the tag, and do not recurse into PitchBook / BioCentury / proprietary databases without explicit user authorization.

Key conventions:
- **Col F decision rule**: "股份有限公司" (domestic joint stock) → H-share; offshore Cayman/BVI holdco → Red Chip; VIE structure → VIE.
- **Col D suffix**: `-B` = Ch.18A biotech; `-P` = Ch.18C specialist tech; none = Main Board / commercial.
- **Cols J/K/L**: target latest full year (FY25 > FY24), RMB millions only. Pre-revenue: Revenue=`None`, note in col G. Interim/partial FY: flag in col N (e.g., "FY24 data; FY25 not yet available").

**Cols F and G — mandatory verification (no blanks allowed)**:
- **Every row in every save** must have cols F (Shareholder Structure) AND G (Business Model / Clinical Stage of Core Asset) populated — this applies to NEW rows, REFRESH rows, AND any pre-existing row that happens to be blank when the skill runs.
- **Verification rule**: Col F and G values must be sourced from the prospectus (first choice — Tier 1) or two independent public sources (second choice, only when prospectus is unavailable). Never infer col F from the company name alone; always check the corporate structure section of the prospectus.
- **Independence definition** (only relevant when falling back to web sources because prospectus is paywalled / 404 / Firecrawl returned null on F or G): "two independent public sources" means **different issuer entity AND different document class**. The following all count as ONE source family and do **NOT** satisfy the rule on their own:
  - Same issuer, different docs — company website + same-issuer press release; HKEXnews announcement + same-issuer cited in news
  - **Parent + majority-owned subsidiary** — 集团公告 + 子公司公告; Cayman holdco filing + onshore PRC subsidiary filing; A-share parent + H-share child of the same group
  - Same trade-media family — different sections of the same outlet (e.g., 财联社 主页 + 财联社 IPO 板块; sina finance + sina news re-cite)

  To satisfy ≥2 independent sources, pair the issuer family with at least one of: **regulator filing** (HKEXnews / SAMR / NMPA / 巨潮资讯网 / 港交所披露易), **independent trade-media** (Caixin 财新 / 21财经 / Endpoints / FierceBiotech / BioCentury), or **PitchBook / S&P Capital IQ**. Single-family-only sources → tag `[Single source family — verify]` per the Robustness Tag Vocabulary (§ Phase 4 below) — do NOT silently treat as verified.
- **Col F taxonomy** (use exactly one of): `H-share` | `Red Chip` | `VIE` — `H-share` for PRC-domiciled joint stock companies (股份有限公司 incorporated in PRC); `Red Chip` for **any** offshore-holdco structure (Cayman / BVI / Bermuda) **without** VIE — regardless of where operating business sits; `VIE` when a variable interest entity is in the chain. The specific incorporation jurisdiction (e.g., `Cayman-incorporated` / `BVI-incorporated`) goes in col N as a footnote, not in col F. Col F is intentionally narrow at three values to avoid the Red Chip / Cayman holdco synonymy that the older 5-option enum invited (Cayman holdco *was* effectively a subset of Red Chip — same legal structure, different naming convention).
- **Col G format**: `<stage>; <one-line description of business model or core asset status>`. Stage = one of `Commercial-stage` | `Clinical-stage` | `Pre-clinical` | `Commercialized` (for commercial-stage non-biotech). Example: "Clinical-stage; 5 assets in Phase II/III oncology pipeline, pre-revenue" or "Commercialized; #2 goat milk formula brand in China (14% share)".
- **If both prospectus and web search (≥3 distinct Tier-1 attempts) fail to yield F or G**, do not guess — apply the `[NOT FOUND — searched: <list>]` robustness tag, leave the cell blank, and surface in Gate 2 for user override. (Bounded-effort discipline borrowed from `/web-research` Rule 10.)
- **When refreshing a row**, re-verify F and G against the latest prospectus even if they already have values — business models and structures do change (e.g., a 2024 Reorganization may move an entity from a Cayman/BVI holdco structure to a PRC joint stock structure, flipping F from `Red Chip` to `H-share`; Alebund Pharmaceuticals' May 2026 re-filing is a recent example).

**Pause and ask the user when**: prospectus contradicts web search; FY financials disclosed in multiple currencies with no clear RMB figure; sector is genuinely ambiguous (e.g., dx company with therapeutic pipeline); col F or G cannot be determined from prospectus + web.

### Phase 4 — Diff Preview ⚠️ MANDATORY CHECKPOINT (Gate 2)

This is **Gate 2**. Gate 1 was the Phase 0 candidate approval (only when Phase 0 ran). Both gates must pass before any write. **Never skip Phase 4**, even when Phase 0 looked fine.

When a staging row has `_confidence["X"] == "low"` OR `pdf_status != "ok"`, surface it in the diff with a QC flag line like `"PHASE0 LOW-CONF: col H confidence=low"` or `"PHASE0 pdf_status=partial (missing chapter: financial)"` so the user can decide whether to accept, correct, or drop.

**Firecrawl-specific flags to surface in Phase 4**:
- Any `firecrawl_pending_col_X` in `_qc_flags` → Firecrawl did not fill that column. Either Firecrawl was skipped, returned null, or was unavailable. Display as `"FIRECRAWL PENDING: col X blank — fill manually or retry firecrawl"`. Block Phase 5 save until the user either fills the cell or explicitly accepts a blank.
- Any `firecrawl_off_enum_H: <value>` in `_qc_flags` → Firecrawl returned a sector string not in `CANONICAL_SECTORS`. Surface the raw value, ask the user to map it or override. The row_draft["H"] remains `None` until resolved.
- Per-field provenance now includes `firecrawl:SUMMARY` for rescued cols — include the provenance tag in the diff so the reviewer knows which cells came from the LLM vs. the HKEX feed vs. pdfplumber.

#### Robustness Tag Vocabulary

> Borrowed from `/web-research` Tag Vocabulary + Rule 4 (independent sources) + Rule 10 (bounded effort). Applies to **F + G robustness only** — these are the two narrative fields where source robustness varies row-by-row. (H sector is enum-validated; M lead asset is informational; J/K/L are pdfplumber-deterministic; I is regex-templated; C/D/E come from the authoritative HKEX feed.)

Every NEW or REFRESH row gets exactly **one** F+G robustness tag, assigned at Phase 0 step 8d (or at Phase 3 web-fallback step for non-Phase-0 rows). The tag drives Phase 4's robustness counter aggregation and the col N suffix.

| Trigger condition | `_qc_flags` entry | Col N suffix |
|---|---|---|
| Phase 0 path: prospectus extraction succeeded; F+G+H+M all non-null + in-enum + `_confidence == "high"` | `verified_fg_prospectus` | `[VERIFIED F+G — prospectus]` |
| Phase 0 path: F or G non-null but `_confidence` includes `"low"` or `"medium"` (e.g., Firecrawl extracted but the prospectus section was ambiguous) | `single_source_prospectus_fg` | `[Single source — prospectus only]` |
| Phase 3 fallback: prospectus 404 / Firecrawl null → ≥2 **independent** web sources (per the Independence definition in Phase 3) agreed on F and G | `web_cross_checked_fg` | `[Web cross-checked F+G]` |
| Phase 3 fallback: prospectus 404 / Firecrawl null → only one source family found, no independent confirmation | `single_source_family_fg` | `[Single source family — verify]` |
| **All Tier-1 attempts failed** (prospectus + 3 distinct firecrawl_search top hits + IR page) — bounded-effort discipline | `not_found_fg` | `[NOT FOUND — searched: prospectus, firecrawl_search top-3, IR]` |
| Prospectus extraction conflicts with a secondary source (e.g., prospectus says "Red Chip" (offshore holdco, no VIE), IR page says "VIE in chain") | `conflicting_fg` | `[Conflicting — used prospectus]` (footnote which source was discarded in col N) |

**Mutual exclusion**: pick the dominant tag per row. A row that's prospectus-anchored AND web cross-checked still tags as `verified_fg_prospectus` (the prospectus is Tier 1 — additional cross-check is bonus, not separate state). A row tagged `not_found_fg` cannot also be `conflicting_fg`.

**Phase 5 write blocking**:
- `not_found_fg` → block Phase 5 unless user fills cells in Gate 2 OR explicitly accepts blank with `accept blank <row>`
- `conflicting_fg` → block Phase 5 unless user confirms which source to use with `keep prospectus <row>` / `keep <secondary> <row>`
- All other tags pass through

**Implementation note**: tag classification logic lives in the skill driver (not in `hkex_scraper.py`), invoked after `apply_firecrawl_narrative()` returns or after web-fallback completes. The Python module exposes the building blocks (`_provenance` / `_confidence` / off-enum detection); the driver synthesizes the appropriate tag per row.

**Before writing anything**, present to the user:

```
Projected tracker state:
  Existing rows:     M → M' (+n new, -d duplicate-deletes)
  Candidate rows:    N → 0 (all filled)

--- F+G robustness summary (NEW + REFRESH rows being written) ---
  [VERIFIED F+G — prospectus]:        X / Y rows  (Tier-1 prospectus, high-confidence)
  [Single source — prospectus only]:  S rows      (prospectus extracted but low/medium confidence)
  [Web cross-checked F+G]:            W rows      (prospectus failed → ≥2 independent web sources)
  [Single source family — verify]:    K rows      (prospectus failed → only one source family — analyst review needed)
  [Conflicting — used prospectus]:    P rows      (prospectus vs secondary disagreement — discarded source footnoted)
  [NOT FOUND — searched: ...]:        L rows      (bounded-effort fail — manual fill or accept blank required)
  Stale FY data (older than target):  T rows      (col N flagged with FY note)

  → IC-grade target:  [VERIFIED] + [Web cross-checked] ≥ 80% of (Y-S-K-L) row pool
  → Block-Phase-5 row count (not_found_fg + conflicting_fg unresolved): <count>

--- TRULY NEW (n rows) ---
  1. <CompanyName-B> (<Chinese>) | <structure> | <sector> | <sponsor>  [<F+G tag>]
     FY25 rev <X>m / NI <-Y>m / cash <Z>m
     Lead: <one-liner>
     Highlights: <first-sentence>
     Provenance: F=<src> G=<src> H=<src> M=<src>     (e.g., F=firecrawl:SUMMARY, G=firecrawl:SUMMARY, H=firecrawl:SUMMARY, M=firecrawl:SUMMARY)

--- DATA RECOVERY (r rows) ---
  1. <Company> — recovered from old file row <N>  [<F+G tag carried over OR re-verified>]
     FY refresh: FY24 → FY25 (needed / not needed)

--- DUPLICATE → DELETE (d rows) ---
  1. <Company> | old row <N> deleted, new row keeps latest filing date

--- QC FLAGS (c rows) ---
  1. <Company> — <reason>      (Firecrawl pending / off-enum / pdf_status partial / etc.)

Output path: <new_file with version suffix v+1>

Confirm to write? (yes / fix <row X> / accept blank <row X> / keep prospectus <row X> / dry run / stop)
```

**Reading the robustness summary**:
- High `[VERIFIED F+G — prospectus]` percentage = clean Phase 0 run; little manual review needed.
- Non-zero `[Single source family — verify]` or `[NOT FOUND]` = analyst attention required before Phase 5.
- Any `[Conflicting]` row blocks Phase 5 until you pick `keep prospectus <row>` or `keep <secondary> <row>` (Independence definition in Phase 3 — discarded source goes in col N as footnote).

**Wait for explicit user confirmation** (`yes` / `go` / `OK`) before Phase 5. If user says `dry run`, exit cleanly without any file changes. New per-row overrides accepted in Gate 2:
- `accept blank <row>` — accept a `not_found_fg` row with F or G blank (overrides Phase 5 block)
- `keep prospectus <row>` / `keep <secondary> <row>` — resolve a `conflicting_fg` row by picking the winning source
- `fix <row>` — pause for manual cell fill before re-presenting Gate 2

### Phase 5 — Write + QC (only after confirmation)

1. **Fix data-quality issues first** (in-memory, before write):
   - **Date consistency**: convert all col C string dates to `datetime.datetime`
   - **Date parsing**: check for dd/mm swap (month > 12 means it was misinterpreted)
   - **Verify dates against HKEX feed**: for every row whose normalized name matches the HKEX JSON feed (`fetch_hkex_feed(year, board='sehk', lang='en')`, `app[*].ls[*]` where `nF == "Application Proof (1st submission)"`), overwrite col C with the feed's `d` value. The HKEX feed is the authoritative source — if the master disagrees, the master is wrong. Surface every fix in the Phase 4 diff so the user can override.
   - **Structure verification**: all "股份有限公司" → H-share; offshore holding → Red Chip
   - **Deduplicate by normalized name**: group rows by normalized English name (strip `-B`/`-P` suffix, `†` marker, common corp suffixes, lowercase, collapse whitespace). For each group with >1 row, keep the row with the most recent filing_date; break ties by data richness (count of non-null cells in D..N). Drop the losers. Report removed rows in Phase 4 diff.
2. **Sort by filing date DESC** — **REQUIRED every save**:
   - After all data fixes and dedup, sort all data rows by col C filing_date **descending** (newest at top, oldest at bottom). Secondary key: company name ascending (for stable ties).
   - Rows with unparseable / None dates sort to the bottom.
   - Move each row's values + per-cell styles + row height together — never split a row's content from its formatting during the sort.
3. **Force dd/mm/yyyy format on col C** — apply `number_format = 'dd/mm/yyyy'` to every populated col C cell in the data region. The HKEX website displays filing dates as `dd/mm/yyyy` and the tracker must match. Do not preserve the original per-row format for col C; uniformize.

**3a. Force Arial size 8 BLACK font on the ENTIRE region — not just col N, not just populated cells** — apply `Font(name='Arial', size=8, color='FF000000', ...)` to **every cell** in rows 1 through last_data_row, cols A through (at least) N. Include BLANK cells too: if you only touch populated cells, then later editing a previously-blank cell will reveal Excel's default Calibri 11 (the exact bug this rule prevents). Preserve bold/italic/underline/strike but force name + size + color:

```python
from openpyxl.styles import Font, PatternFill, Color

BLACK = Color(rgb="FF000000")
NO_FILL = PatternFill(fill_type=None)

last_data_row = max(r for r in range(3, ws.max_row+1) if ws.cell(row=r, column=4).value)

for r in range(1, last_data_row + 1):       # includes row 1 metadata + row 2 header + all data
    for c in range(1, 15):                    # cols A..N; extend to 21 if you use cols O..T
        cell = ws.cell(row=r, column=c)
        existing = cell.font
        cell.font = Font(
            name='Arial',
            size=8,
            bold=existing.bold or False,
            italic=existing.italic or False,
            underline=existing.underline,
            strike=existing.strike or False,
            color=BLACK,
        )
        if cell.fill.patternType:
            cell.fill = NO_FILL               # strip highlight/solid fills — no colored backgrounds
```

**Why both populated and blank cells**: openpyxl stores per-cell style objects, and a "blank" cell that was never written still carries the workbook default style (Calibri 11). When the next skill run (or a human user) types a value into that cell, it inherits Calibri 11 — breaking the uniform-format invariant. Setting Arial 8 black on every cell in the region makes the format self-healing: any future write lands in an Arial-8-black cell by default.

**Rationale for no fills**: historical tracker versions used highlighted rows as an ad-hoc "watch list" marker. That semantics is unreliable (highlights drift on sort, are inconsistent across versions, and don't survive dedup). The tracker standard is **no fills anywhere**; use col B (Status) for any row-level markers instead.
4. **Write via openpyxl**:
   - Use `openpyxl.load_workbook()` on the new file to preserve all existing formatting.
   - Only write to cells that need updating (plus the sort-shuffled rows).
   - Copy cell styles from a nearby complete row for newly filled cells (see Formatting Reference below).
   - **Never overwrite** formatting of rows you didn't modify.
   - When sorting, clear the old data region first, then write the sorted list back starting at the first data row — this avoids row-index collisions from partial shuffling.
   - Apply duplicate-row deletes in **reverse row order** to avoid index shifting (only relevant if not using the full clear-and-rewrite approach above).
5. **Update row 1 metadata**: set `ws.cell(row=1, column=1).value = "Updated as of <Mon D, YYYY>"` with today's date.
6. **Save** with incremented version suffix (e.g., `v1` → `v2`, or append `_vN` if no version in filename).
7. **Readback**: re-open the saved file and verify:
   - Top 5 rows are the newest filings (DESC sort).
   - 3-5 sample cells match expected values (one row each from NEW / RECOVERY / DUPLICATE / REFRESH batches when applicable).
   - All col C cells use `dd/mm/yyyy` format.
   - No duplicate company names remain post-dedup.
8. Run the QC Checklist below before reporting done.

## Python 3.9 Constraints
- Use `from typing import Optional, List, Dict` (not `X | Y`)
- Use `datetime.datetime` for dates
- Use `from copy import copy` for openpyxl style copying

## Formatting Reference
```python
from copy import copy

# Copy style from reference row to target
src_cell = ws.cell(row=ref_row, column=col)
tgt_cell = ws.cell(row=target_row, column=col)
tgt_cell.font = copy(src_cell.font)
tgt_cell.alignment = copy(src_cell.alignment)
tgt_cell.border = copy(src_cell.border)
tgt_cell.number_format = src_cell.number_format
```

## QC Checklist (Run Before Saving)
- [ ] All updated rows have cols E-M filled (at minimum E, F, G, H, M)
- [ ] Shareholder structure populated for every row
- [ ] Financial data annotated with FY year if not target FY
- [ ] All dates in col C are datetime type
- [ ] **All col C cells use `dd/mm/yyyy` number_format** (uniform, no mixed formats)
- [ ] **Data rows sorted by filing_date DESC** — first data row = newest filing, last data row = oldest. Verify by reading top 5 and bottom 3 rows after save.
- [ ] **No duplicate companies** — dedup by normalized name complete; every `norm_name` appears in exactly one row.
- [ ] **Row 1 metadata** updated to `Updated as of <today>`.
- [ ] **Col F (Shareholder Structure) — ZERO blanks**: every data row has a non-null col F value sourced from prospectus or ≥2 public sources. Run `sum(1 for r in data_rows if not ws.cell(r, 6).value) == 0` before save.
- [ ] **Col G (Business Model / Clinical Stage) — ZERO blanks**: every data row has a non-null col G value matching the `<stage>; <description>` format. Run `sum(1 for r in data_rows if not ws.cell(r, 7).value) == 0` before save.
- [ ] **Col N font = Arial size 8**: every populated col N cell has `font.name == 'Arial'` and `font.size == 8`. Run `all(ws.cell(r,14).font.name == 'Arial' and ws.cell(r,14).font.size == 8 for r in data_rows if ws.cell(r,14).value)` before save.
- [ ] **ENTIRE region uniform Arial 8 black — populated AND blank cells**: every cell in rows 1 through last_data_row × cols A through N has `font.name == 'Arial'`, `font.size == 8`, and `font.color.rgb in ('00000000', 'FF000000')`. Run the following and expect `{('Arial', 8.0): <n>}` as the only entry:
```python
from collections import Counter
fonts = Counter((ws.cell(r,c).font.name, ws.cell(r,c).font.size) for r in range(1, last_data_row+1) for c in range(1, 15))
assert set(fonts.keys()) == {('Arial', 8.0)}, f"Non-uniform fonts: {fonts}"
```
- [ ] **No fills anywhere in data region**: every cell in rows 1 through last_data_row × cols A through N has `fill.patternType is None`. Run `assert all(ws.cell(r,c).fill.patternType is None for r in range(1, last_data_row+1) for c in range(1, 15))` before save.
- [ ] No unintended duplicate companies
- [ ] Core product descriptions verified against latest prospectus
- [ ] Auto-discovered file paths were echoed back to user and confirmed before Phase 1
- [ ] **No unresolved `firecrawl_pending_col_X` flags on rows being written**: every staging row that survived Phase 4 either has F/G/H/M populated (via Firecrawl or manual fill) or has been explicitly accepted as blank by the user during Gate 2. Run `all("firecrawl_pending" not in f for row in staging_to_write for f in row.get("_qc_flags", []))` before save.
- [ ] **No `firecrawl_off_enum_H` flags**: any off-enum sector returns from Firecrawl have been mapped into `CANONICAL_SECTORS` and the flag cleared.
- [ ] **Firecrawl provenance is recorded**: for rows where F/G/H/M came from Firecrawl, `_provenance["F"/"G"/"H"/"M"]` starts with `firecrawl:` — useful for later audit.
- [ ] **Every NEW + REFRESH row has exactly ONE F+G robustness tag** in `_qc_flags`: from `{verified_fg_prospectus, single_source_prospectus_fg, web_cross_checked_fg, single_source_family_fg, conflicting_fg, not_found_fg}`. Run:
  ```python
  ROBUSTNESS_TAGS = {"verified_fg_prospectus", "single_source_prospectus_fg",
                     "web_cross_checked_fg", "single_source_family_fg",
                     "conflicting_fg", "not_found_fg"}
  for row in new_or_refresh_rows:
      tags = [f for f in row.get("_qc_flags", []) if f in ROBUSTNESS_TAGS]
      assert len(tags) == 1, f"Row {row} has {len(tags)} robustness tags (expected 1): {tags}"
  ```
- [ ] **No unresolved `not_found_fg` rows**: every `not_found_fg` row has either had F/G manually filled in Gate 2 (clears the tag) OR has been explicitly `accept blank <row>` by user. Run `all("not_found_fg" not in row.get("_qc_flags", []) or row.get("_user_accept_blank") for row in staging_to_write)` before save.
- [ ] **No unresolved `conflicting_fg` rows**: every `conflicting_fg` row has been resolved via `keep prospectus <row>` / `keep <secondary> <row>` (which sets `_user_resolved_conflict` and rewrites col N footnote with the discarded source).
- [ ] **Col N suffix matches `_qc_flags` robustness tag** for every NEW + REFRESH row. Sanity check: if `_qc_flags` contains `verified_fg_prospectus`, col N must end with `[VERIFIED F+G — prospectus]` (or analogous for other tags).

## Edge Cases — When Auto-Discovery or the Pipeline Breaks

| Situation | What to do |
|---|---|
| **No new file found in default dir** | Tell user the dir is empty / has no matching pattern. Ask for explicit path. Do NOT guess. |
| **Multiple files with same (year, month)** — e.g., user has `_v1` and `_v2` for current month | Pick highest version for NEW. If unsure whether `_v2` is "in progress" vs "completed", **ask the user** which to treat as the working file. |
| **Old file is from same month as new file** | This means user is iterating within the same month (e.g., `Apr_v1` → `Apr_v2`). Treat the older version as "old file" and recover any rows that were filled in the v1 but missing in v2. |
| **No prior file exists** (very first month, or default dir was just created) | Skip the DATA RECOVERY bucket entirely. All candidates are TRULY NEW. Tell the user explicitly: *"No prior file found — all N candidates will be web-researched as TRULY NEW."* |
| **Phase 1 finds zero candidate rows** (all rows already complete) | Do NOT silently exit. Report: *"All M rows in `<filename>` are already complete (cols E–M filled). Nothing to do. Was this expected? (yes / re-scan / specific row needs refresh)"* |
| **Year skip in file naming** (e.g., NEW = `Jan 2026`, OLD candidates only in `2025`) | Auto-discovery should still work — most recent prior file regardless of year boundary. Just confirm with user since calendar transitions are common error sources. |
| **Sheet name varies across files** (e.g., one file uses `Sheet1`, another uses `A1 Pipeline`) | Always read the **active sheet** via `wb.active`, not by hardcoded name. If multiple sheets exist, ask user which to update. |
| **openpyxl `read_only=True` strips formatting on save** | Never use `read_only=True` for the NEW file. Read-only is fine for the OLD file (lookup only, never written back). |
| **Chinese characters render as `_x0000_` placeholders** | This is an openpyxl encoding issue with rich-text or shared-string artifacts. Re-open with `keep_vba=False, data_only=False`. If still broken, fall back to copying the cell value via `cell.value = str(val).strip()` and let the user re-verify. |
| **Date col C has mixed types** (some `datetime`, some `str`, some `int` Excel serials) | Coerce all to `datetime.datetime` in Phase 5 before write. For string dates, use `dateutil.parser.parse(s, dayfirst=True)`. For Excel serials, use `openpyxl.utils.datetime.from_excel(s)`. Surface all coercions in the Phase 4 diff. |
| **Prospectus PDF is paywalled or 404** (HKEX archives) | Apply bounded-effort discipline (Phase 3 web-fallback): try **3 distinct Tier-1 attempts** — `mcp__firecrawl__search` with `<company> HKEX A1 prospectus` (top-3 hits) + company IR page. Each hit found → pipe URL through `mcp__firecrawl__scrape` with `FIRECRAWL_NARRATIVE_SCHEMA`. If ≥2 independent web sources agree per the Independence definition (Phase 3) → tag `web_cross_checked_fg`. If only one source family found → tag `single_source_family_fg`. After 3 failures → tag `not_found_fg` + col N = `[NOT FOUND — searched: prospectus, firecrawl_search top-3, IR]`. Block Phase 5 until user `accept blank <row>` in Gate 2. Do not fabricate financials. |
| **Firecrawl API unavailable or credits exhausted** | `extract_fields_from_chapters` has already returned a staging row with F/G/H/M as `None` and `firecrawl_pending_col_X` QC flags attached. Skip the `mcp__firecrawl__scrape` call, proceed to Phase 4, and let the user fill F/G/H/M manually during the Gate 2 review. Tag the row `single_source_family_fg` if the user fills from a single web source, or `not_found_fg` if they `accept blank`. **Do NOT fabricate values or fall back to the deleted regex extractors.** I (sponsor) and J/K/L (financials) are unaffected because they never depended on Firecrawl. |
| **Firecrawl returns null for F/G/H/M** (schema satisfied but LLM couldn't find the field) | `apply_firecrawl_narrative` keeps the cell as `None` and the `firecrawl_pending_col_X` flag stays. Trigger Phase 3 web-fallback for that row (bounded-effort 3 Tier-1 attempts), then re-classify the F+G robustness tag accordingly. Surface in Phase 4 diff with the resulting tag (`web_cross_checked_fg` / `single_source_family_fg` / `not_found_fg`); user chooses: fill manually, retry Firecrawl with a wider page range, or accept blank. |
| **Firecrawl returns an off-enum sector for col H** | `apply_firecrawl_narrative` adds `firecrawl_off_enum_H: <raw>` to `_qc_flags` and leaves col H as `None`. Surface in Phase 4 diff with the raw value so the user can map it into `CANONICAL_SECTORS`. (This is a col H issue, not an F+G robustness issue — the row's F+G tag is independent.) |
| **Prospectus and secondary source disagree on F or G** (e.g., prospectus says "Red Chip" (offshore holdco, no VIE), IR page says "VIE in chain") | Tag the row `conflicting_fg`, set col N suffix to `[Conflicting — used prospectus]` with footnote naming the discarded source, and **block Phase 5** until user resolves with `keep prospectus <row>` (default — Tier 1 wins per source hierarchy) OR `keep <secondary> <row>` + brief justification (e.g., "prospectus is stale, IR page is post-restructuring"). Update col N footnote to reflect the resolution. |
| **Sponsor list contains 4+ banks** (rare jumbo deals) | Truncate to first 3 in col I + append "et al." Full list goes in col N. |
| **Same company appears under both English and Chinese name in different rows** | Phase 2 normalize step handles English `-B`/`-P` suffix, but if the new file has one row with English name and another with Chinese name, treat as DUPLICATE and merge — keeping the row with the more recent filing date. Surface in Phase 4 diff. |

## Phase 0 Implementation Reference

All Phase 0 logic lives in `hkex_scraper.py` — **canonical location**: [`hkex-a1-pipeline/scripts/hkex_scraper.py`](https://github.com/soback26/hkex-a1-pipeline/blob/main/scripts/hkex_scraper.py) (1515 LOC, single module, pure Python, no side effects outside the explicit cache dir). The scraper is open-source in the public `hkex-a1-pipeline` repo so that the scraping-tool layer is reproducible for external users; this private `lifesci-methodology` repo only contains the 5-phase investment-workflow wrapper. The skill driver imports and calls these functions in order:

| Function | Purpose |
|---|---|
| `fetch_lifesci_candidates(year, since=None, include_gem=False)` | Returns unified EN+CN candidate records with resolved Multi-Files TOC URLs. |
| `filter_candidates(candidates)` | Partitions into `(passed, dropped)` via EN kw + CN kw + `-B`/`-P` suffix. Each passed record gets a `reason` field. |
| `load_master_tracker(xlsx_path)` | Read-only snapshot of the master tracker with normalized names, parsed filing dates (datetime/str tolerated), and J/K/L values. |
| `classify_candidates(candidates, master)` | Buckets into `{new, refresh, skip}`. Multi-row master matches annotate `multi_row_match` QC flag. |
| `create_cache_dir(run_date=None)` | Creates per-run cache dir under `checkpoints/YYYY-MM-DD_a1_hkex_pdfs/`, with `_2`/`_3` suffix on collision. |
| `fetch_targeted_chapters(candidate, cache_dir)` | Parses Multi-Files TOC, picks SUMMARY/BUSINESS/FINANCIAL via `CHAPTER_VARIANTS`, downloads each chapter PDF. Also populates `candidate['chapter_urls']` with the remote URLs per slot (used by Firecrawl in step 8c). Sets `pdf_status` to ok/partial/no_toc/failed. |
| `extract_fields_from_chapters(candidate, target_fy="FY25")` | Builds a `row_draft` dict keyed by col letter C..N + `_provenance` + `_confidence`. Deterministic: C/D/E from HKEX feed, J/K/L from pdfplumber FINANCIAL table, I from regex on pdfplumber SUMMARY text, N assembled from FY label + pdf_status. **F/G/H/M are left as None** with `firecrawl_pending_col_X` QC flags — these are expected to be filled by a downstream `apply_firecrawl_narrative` call from the skill driver. |
| `apply_firecrawl_narrative(staging_row, fc_data, source_label="firecrawl:SUMMARY")` | Merges a Firecrawl `/scrape` JSON-format result into a staging row. Maps `shareholder_structure`/`business_model`/`sector`/`lead_asset` onto `row_draft["F"/"G"/"H"/"M"]`, updates provenance + confidence, and clears pending QC flags. Guards col H against off-enum sector values. |
| `auto_classify_fg_robustness(staging_row)` | Phase 0 step 8d helper. Reads `_provenance` + `_confidence` + `row_draft` for cols F+G and assigns a Phase-0-determinable F+G robustness tag (`verified_fg_prospectus` / `single_source_prospectus_fg` / `not_found_fg`). Mutates row's `_qc_flags` + col N suffix in place. Returns the applied tag string. |
| `apply_fg_robustness_tag(staging_row, tag, suffix_override=None)` | Manually apply / override a F+G robustness tag (used by the agent driver after Phase 3 web fallback for `web_cross_checked_fg` / `single_source_family_fg` / `conflicting_fg` / overrides of an auto-classified tag). Strips any pre-existing robustness tag from `_qc_flags` and any robustness suffix from col N before applying the new one. `suffix_override` lets the caller customize the col N suffix (e.g., for `not_found_fg` with a custom searched-attempts list, or `conflicting_fg` with a discarded-source footnote). Raises `ValueError` if `tag` is not in `FG_ROBUSTNESS_TAGS`. |
| `FIRECRAWL_NARRATIVE_SCHEMA` / `FIRECRAWL_NARRATIVE_PROMPT` | Module-level constants the skill driver passes verbatim to `mcp__firecrawl__scrape` under `jsonOptions`. Defines the four narrative fields + their enum constraints + extraction instructions. |
| `FG_ROBUSTNESS_TAGS` / `FG_ROBUSTNESS_COL_N_SUFFIX` | Module-level constants. The 6-tuple of allowed robustness tag strings + dict mapping each to its canonical col N suffix. Phase 4's robustness counter iterates over `FG_ROBUSTNESS_TAGS` to aggregate counts per tag. |
| `cleanup_cache_dir(cache_dir, had_failures)` | `rmtree` cache on success; preserve on failure. Refuses to touch paths outside the `checkpoints/` root. |

### Skill-driver run pattern (REPL + agent)

The pattern is **hybrid**: Python handles deterministic work (feed, PDF download, financial numbers, sponsor regex), and the agent handles Firecrawl MCP tool calls for F/G/H/M. The agent orchestrates the loop, not a single Python script, because MCP tool calls happen at the Claude Code tool layer, not inside Python.

```python
# Python side (runs in a REPL started via start_process("python3 -i"))
# Assumes cwd = repo root (hkex-a1-pipeline/)
import sys, datetime
sys.path.insert(0, "scripts")
import hkex_scraper as hs

# Step 0a-0e: scrape + filter + classify
year = datetime.date.today().year
master_path = "tracker/a1_pipeline_tracker.xlsx"
cands = hs.fetch_lifesci_candidates(year)
passed, dropped = hs.filter_candidates(cands)
master = hs.load_master_tracker(master_path)
buckets = hs.classify_candidates(passed, master)

# Render Gate 1 to the user, wait for approval, then:
cache_dir = hs.create_cache_dir()
staging = []
for cand in buckets["new"] + buckets["refresh"]:
    hs.fetch_targeted_chapters(cand, cache_dir)
    row = hs.extract_fields_from_chapters(cand, target_fy="FY25")
    staging.append(row)
# At this point every staging row has F/G/H/M = None with pending flags.
```

```
# Agent side (for each staging row, invoke Firecrawl MCP tool and merge)
For each row in staging where row["candidate"]["chapter_urls"]["summary"] is truthy:
    fc = mcp__firecrawl__scrape(
        url=row["candidate"]["chapter_urls"]["summary"],
        formats=["json"],
        jsonOptions={
            "schema": hs.FIRECRAWL_NARRATIVE_SCHEMA,
            "prompt": hs.FIRECRAWL_NARRATIVE_PROMPT,
        },
        onlyMainContent=True,
    )
    fc_data = (fc.get("data") or {}).get("json") or fc.get("json") or {}
    # Back to Python REPL:
    hs.apply_firecrawl_narrative(row, fc_data, source_label="firecrawl:SUMMARY")
    # Phase 0 step 8d -- assign initial F+G robustness tag based on extraction state:
    hs.auto_classify_fg_robustness(row)
```

```python
# Python side again: hand `staging` to Phase 1 (in-memory); on success of Phase 5:
hs.cleanup_cache_dir(cache_dir, had_failures=False)
```

**Degraded path — Firecrawl unavailable**: skip the agent-side Firecrawl loop entirely (no `apply_firecrawl_narrative` call). `staging` rows still have F/G/H/M as None with `firecrawl_pending_col_X` QC flags. **Still call `hs.auto_classify_fg_robustness(row)` on each row** — it will tag them as `not_found_fg` (F or G is None), so Phase 4 surfaces the gap correctly. The user fills manually during Gate 2 review (which clears the tag — agent then calls `hs.apply_fg_robustness_tag(row, "single_source_family_fg")` or whichever tag matches the manual-fill source). The skill never crashes on Firecrawl failure.

**Phase 3 web fallback (TRULY NEW path or any post-Phase-0 row that needs F/G)**: after the agent driver completes web fallback (≥3 Tier-1 attempts per `/web-research` Rule 10), it calls `hs.apply_fg_robustness_tag(row, tag)` directly with one of `web_cross_checked_fg` / `single_source_family_fg` / `conflicting_fg` / `not_found_fg` based on the fallback outcome — overriding whatever auto-classification set in step 8d.

### Python 3.9 compatibility

`hkex_scraper.py` is strict Python 3.9 — no PEP 604 unions (`X | Y`), no `match/case`, no lowercase generics (`list[dict]`). Uses `Optional[X]` / `List[Dict]` / `Tuple[X, Y]` from `typing`. All HTTP calls respect a 0.8s `REQ_DELAY` rate limit with 3x exponential backoff retries.

### Hard external dependency

The scraper depends on the HKEX static JSON feed at `https://www1.hkexnews.hk/ncms/json/eds/app_{YYYY}_sehk_{e|c}.json`. If this URL pattern changes, `fetch_hkex_feed` will raise `HkexFetchError` and the driver should surface a clear message to the user rather than silently falling back. An HTML-scrape fallback on `appindex.html` is a future enhancement — currently the scraper hard-fails on JSON feed errors so the user can notice and manually escalate.

## Firecrawl MCP Integration (Cols F/G/H/M) {#firecrawl-mcp-integration-cols-fghm}

**Scope**: cols **F (Shareholder Structure)**, **G (Business Model)**, **H (Sector)**, and **M (Lead Asset)**. These four fields are extracted via LLM-backed schema extraction, not regex.

**Not in scope**: col I (sponsor) stays on regex (`_extract_col_I_sponsor`) — sponsor-bank text in prospectus summaries is templated and regex is both deterministic and reliable. Cols J/K/L (financial numbers) stay on `pdfplumber` table parsing — numbers need exact deterministic extraction and LLMs hallucinate.

### Why Firecrawl for F/G/H/M

The previous implementation used ~350 lines of regex + keyword heuristics in `_extract_col_F_structure`, `_extract_col_G_business_model`, `_extract_col_H_sector`, and `_extract_col_M_lead_asset`. Those heuristics had persistent failure modes:

- Col F (structure): couldn't reliably detect VIE structures (offshore holdco with VIE in the chain) from plain offshore holdco; the older keyword approach defaulted to "Red Chip" / "Cayman holdco" too aggressively on Chapter 18A signals. The 3-option enum (`H-share` | `Red Chip` | `VIE`) makes the VIE-vs-no-VIE distinction the only thing the LLM has to call.
- Col G (business model): required both stage and therapy area to pattern-match in one pass; often captured only one or the other.
- Col H (sector): 70 lines of keyword rules still mis-classified biotechs that mentioned CDMO suppliers as "CDMO" themselves.
- Col M (lead asset): regex patterns like `SRSD\d+` only caught alphanumeric drug codes; missed narrative names ("Sirubase", "Neoferet").

Firecrawl's `/scrape` endpoint with `formats=["json"]` and a JSON schema runs an LLM over the fetched page (PDF or HTML) and returns a structured dict. This is much more accurate for prose-heavy semantic fields than regex.

### How the skill driver calls Firecrawl

Inside the Phase 0 per-candidate loop (step 8c), after `extract_fields_from_chapters` has returned a staging row with F/G/H/M as `None`:

```python
import hkex_scraper as hs

# (step 8a-8b above already ran; staging_row has F/G/H/M = None)
summary_url = cand["chapter_urls"].get("summary")
if summary_url:
    # Agent-side tool invocation (runs in the Claude Code tool layer,
    # not inside the Python module):
    fc_response = mcp__firecrawl__scrape(
        url=summary_url,
        formats=["json"],
        jsonOptions={
            "schema": hs.FIRECRAWL_NARRATIVE_SCHEMA,
            "prompt": hs.FIRECRAWL_NARRATIVE_PROMPT,
        },
        onlyMainContent=True,
    )
    fc_data = fc_response.get("data", {}).get("json") or fc_response.get("json") or {}
    hs.apply_firecrawl_narrative(staging_row, fc_data, source_label="firecrawl:SUMMARY")
```

**What `apply_firecrawl_narrative` does**:
1. Reads `fc_data["shareholder_structure" | "business_model" | "sector" | "lead_asset"]`
2. For non-null values, writes them to `row_draft["F" | "G" | "H" | "M"]`, sets `_provenance[X] = "firecrawl:SUMMARY"`, `_confidence[X] = "high"`, and clears `firecrawl_pending_col_X` from `_qc_flags`
3. Guards col H against off-enum values: if Firecrawl returns a sector not in `CANONICAL_SECTORS`, col H stays `None` and a `firecrawl_off_enum_H: <raw>` QC flag is added instead
4. Leaves unchanged any field Firecrawl returned null for — the pending QC flag persists and Phase 4 surfaces it for manual fill

### Cost and rate limits

- Free tier: **500 credits/month** (renews monthly, no credit card required)
- `/v1/scrape` with `formats=["json"]` and a schema: **~5 credits per call**
- Typical monthly run: 12 candidates × 1 Firecrawl call each = **~60 credits/month**
- Safety margin: **~8× under the free limit**
- **Do NOT batch-call Firecrawl on the full 150-row master tracker.** The master rows have already been verified; re-extracting them would burn 750 credits (1.5× free tier) with zero incremental value.
- If credits get low, the `extract_fields_from_chapters` → `None + pending flag` pipeline degrades gracefully. Skip step 8c, proceed to Phase 4, fill manually.

### Schema and prompt (exported from `hkex_scraper.py`)

The scraper exports two module-level constants, and the skill driver must pass them verbatim to the Firecrawl call — do NOT redefine inline:

- **`hs.FIRECRAWL_NARRATIVE_SCHEMA`** — a JSON schema dict with four properties: `shareholder_structure` (enum of 5 structures + null), `business_model` (free string matching `<stage>; <desc>` format), `sector` (enum of `CANONICAL_SECTORS` + null), `lead_asset` (free string, one-line). All four are required but nullable — the LLM must return every key but may return null when the field is not clearly stated.
- **`hs.FIRECRAWL_NARRATIVE_PROMPT`** — one-paragraph instruction telling the LLM to: return null rather than guess; read the Corporate Structure section carefully for VIE detection; and match the issuer's main business for sector (ignoring incidental vendor references).

Both constants live in the Python module so that schema changes are version-controlled alongside the field definitions. If you change a sector vocab item or add a structure type, update both the scraper constant and the Column Structure table in this SKILL.md.

### When NOT to use Firecrawl

| Column | Extraction method | Why not Firecrawl |
|---|---|---|
| **J / K / L** (financial numbers in RMB m) | `extract_financial_tables_pdfplumber` | Numbers need deterministic positional table parsing. LLMs occasionally off-by-1000x on unit detection and occasionally hallucinate values entirely. pdfplumber + the existing unit-pattern heuristics are strictly safer for numeric fields. |
| **I** (sponsor bank names) | `_extract_col_I_sponsor` regex | Sponsor text is highly templated ("Sole Sponsor: XYZ Bank" / "Joint Sponsors: A, B and C"). Regex captures 99%+ and is deterministic across runs — no drift. Firecrawl would cost credits with zero accuracy gain. |
| **C / D / E** (filing date, English name, Chinese name) | HKEX JSON feed | Authoritative machine-readable source. Never extract these from the prospectus. |
| **N** (highlights) | Assembled from FY label + pdf_status | Synthesized from other row_draft state, not extracted from source text. |

### Testing Firecrawl integration

A lightweight smoke test: pick one candidate from a prior month's Phase 0 cache (or a live HKEX filing), call `mcp__firecrawl__scrape` with the SUMMARY chapter URL and the exported schema, and verify:

- All four fields come back non-null on a typical biotech (e.g., any Chapter 18A `-B` filer)
- `sector` is one of `CANONICAL_SECTORS` (not an off-enum string)
- `business_model` starts with one of the four canonical stages
- `shareholder_structure` is one of the five enum values

If any of these fails, the prompt needs tuning — edit `FIRECRAWL_NARRATIVE_PROMPT` in the scraper, not inline in the driver.
