# HKEX A1 Pipeline — Update Rules

## Project Overview

A **HKEX A1 pre-IPO life-science tracker**. Monitors every Application Proof filing by Chinese biotech / medtech / healthcare services companies on the Hong Kong Stock Exchange from 2024 onward, including re-filings after 6-month lapses. Output is a single 14-column Excel tracker sorted by filing date descending.

The canonical Python scraper (`scripts/hkex_scraper.py`, 1500 LOC) and its standalone usage docs (`scripts/README.md`) live in this repo and are the source of truth for HKEX feed fetching, TOC parsing, chapter download, pdfplumber financial extraction, regex sponsor extraction, and the Firecrawl MCP narrative-field schema. The full 5-phase investment workflow that wraps the scraper (Gate 1 candidate approval, Gate 2 diff preview, Phase 5 save invariants) ships with this repo as a **project-level Claude Code skill** at [`.claude/skills/a1-pipeline-update/SKILL.md`](./.claude/skills/a1-pipeline-update/SKILL.md) — open the repo in Claude Code and the skill auto-activates. This CLAUDE.md covers file layout, commit conventions, and save-time invariants; the skill covers Phase 0 scraping orchestration, Gate 1/Gate 2 prompts, and the hybrid extraction pipeline. See [`README.md → Setup & Usage`](./README.md#setup--usage) for end-to-end run instructions.

---

## File Layout

```
hkex-a1-pipeline/
├── README.md                        # Project intro (English, public-safe)
├── CLAUDE.md                        # This file — repo-local operational rules
├── LICENSE                          # MIT
├── tracker/
│   └── a1_pipeline_tracker.xlsx     # Master tracker — single sheet, 14 columns
├── archive/
│   └── A1 pipeline_<Mon> <Year>_update_v<N>.xlsx  # Historical monthly snapshots
├── raw/                             # Reserved for monthly input files (Mode A)
└── scripts/
    ├── hkex_scraper.py              # Canonical Python scraper (1500 LOC)
    ├── requirements.txt             # requests / bs4 / pdfplumber / openpyxl
    └── README.md                    # Python API docs + Firecrawl MCP setup
```

**File discipline**:
- `tracker/a1_pipeline_tracker.xlsx` is the **single live file**. Overwrite on every update; never version-suffix it.
- `archive/` holds historical snapshots with their original version-suffixed names. Each monthly update drops a copy of the new tracker into `archive/` with the format `A1 pipeline_<Mon> <YYYY>_update_v<N>.xlsx` before overwriting the live file.
- `raw/` is for Mode A — user drops a monthly Excel with pre-typed new candidate rows (company name + filing date only).
- `scripts/hkex_scraper.py` is the **canonical location** of the Python scraper. Any private workflow that wraps this scraper should `sys.path.insert` into this directory rather than maintaining its own copy. Never duplicate the scraper — single source of truth.
- `scripts/checkpoints/` (auto-created at runtime, `.gitignore`d) holds transient chapter-PDF downloads. Override via env var `A1_CHECKPOINT_DIR` if you want a different location.

---

## Update Workflow

All updates go through the 5-phase skill at [`.claude/skills/a1-pipeline-update/SKILL.md`](./.claude/skills/a1-pipeline-update/SKILL.md), which wraps `scripts/hkex_scraper.py`. When the user says "update A1 tracker", "A1 update", "scan HKEX for new A1s", or any equivalent, Claude Code auto-activates the skill and drives the pipeline. Do not hand-edit the Excel — every save must pass the Phase 5 invariants (HKEX feed verification, dedup, DESC sort, `dd/mm/yyyy`, Arial 8 black, zero fills). Both the scraper module (`scripts/hkex_scraper.py`) and the investment-workflow wrapper (the skill) now ship with this repo — clone it, install deps, configure Firecrawl MCP, and the full pipeline is reproducible end-to-end.

### Two modes

**Mode A — User provides a pre-typed monthly Excel**: new rows at the top with only D (name) + C (filing date) populated. Skill jumps to Phase 1.

**Mode B — Phase 0 HKEX auto-scraper**: no Excel provided. Skill pulls the HKEX JSON feed, classifies candidates, runs Gate 1 for user approval, downloads SUMMARY/BUSINESS/FINANCIAL chapters, then extracts fields via a **hybrid pipeline**: `pdfplumber` table parser for cols J/K/L (FY revenue / net income / cash) and regex for col I (sponsor banks) run locally from `hkex_scraper.py`; **Firecrawl MCP (`mcp__firecrawl__scrape` with a JSON schema)** is called on the SUMMARY chapter URL for cols F/G/H/M (shareholder structure / business model / sector / lead asset) because LLM-backed schema extraction is more accurate than regex for prose-heavy semantic fields. Firecrawl requires a free API key from [firecrawl.dev](https://firecrawl.dev) registered at user scope via `claude mcp add firecrawl ...`. Staging rows then flow into Phase 1.

Both modes converge at Phase 4 Gate 2 (mandatory diff preview) and Phase 5 write.

### Commit message format

```
update: YYYY-MM-DD | +N new, +R refreshed, -D deduped | <one-line summary>
```

Examples:
- `update: 2026-04-15 | +0 new, +3 refreshed, -10 deduped | Apr 7-12 re-filings: Wuhan Ammunition / Good Doctor / Yeeper; 69 date fixes vs HKEX feed; 13 duplicate rows merged`
- `update: 2026-05-03 | +5 new, +2 refreshed, -0 deduped | May 1-2 batch: <companies>`

### What to commit

Always commit exactly:
- `tracker/a1_pipeline_tracker.xlsx` (live file, overwritten)
- `archive/A1 pipeline_<Mon> <YYYY>_update_v<N>.xlsx` (new snapshot)
- `raw/<filename>.xlsx` if a monthly input file was dropped (Mode A only)

Never `git add .` or `git add -A` — enumerate paths explicitly so stray files don't leak in.

---

## Tracker Schema — 14 Columns

| # | Column | Format |
|---|---|---|
| A | **Timestamp** | `"Updated as of <Mon D, YYYY>"` — row 1 only, refreshed on every save |
| B | **Status** | Free-text marker, e.g., `Expiring in 1 month`. Replaces the legacy highlighted-row scheme. |
| C | **Latest A1 Filing** | `datetime` type; `number_format = 'dd/mm/yyyy'`; strictly from HKEX feed |
| D | **Company Name** | English; suffix `-B` = Ch.18A, `-P` = Ch.18C, no suffix = main board |
| E | **Chinese Name** | Traditional or simplified |
| F | **Shareholder Structure** | `H-share` / `Red Chip` / `VIE` / `Cayman holdco` / `BVI holdco` — exactly one, zero blanks |
| G | **Business Model / Clinical Stage** | `<stage>; <one-line description>`; stage ∈ {Commercial-stage, Clinical-stage, Pre-clinical, Commercialized}; zero blanks |
| H | **Sector** | Pharma / Biotech, MedTech, Services, Diagnostics / Tools, Consumer Health, CDMO, Healthcare Tech |
| I | **Sponsor** | Joint or sole sponsors; 4+ truncated to top 3 + `et al.` |
| J | **Latest FY Revenue (RMB m)** | Number; pre-revenue = `None` or `-` |
| K | **Latest FY Net Income (RMB m)** | Number (negative for losses) |
| L | **Latest FYE Cash (RMB m)** | Number |
| M | **Lead Asset / Business** | One-line core product / business description |
| N | **Highlights / Updates** | Analyst note, FY flagged if not target FY; font Arial 8 black |

### Col F decision rules

- Chinese name ends in `股份有限公司` and company is PRC-domiciled → **H-share**
- PRC operating business under an offshore holdco, no VIE structure → **Red Chip**
- Variable interest entity in the corporate chain → **VIE** (even if otherwise Red Chip)
- Non-PRC operating business (e.g., HK-based retail) under a Cayman-registered holdco → **Cayman holdco**
- Non-PRC operating business under a BVI-registered holdco → **BVI holdco**

When in doubt, check the prospectus "HISTORY, DEVELOPMENT AND CORPORATE STRUCTURE" chapter. Search keywords: `incorporated in` / `joint stock company` / `股份有限公司` / `Cayman` / `BVI` / `VIE` / `可变利益实体`.

**Never guess col F or col G**. If the prospectus chapter and a web search both fail to resolve either field, pause and ask the user — do not leave blank, do not make up a value.

### Col G format rules

`<stage>; <one-line description>` where stage is exactly one of `Commercial-stage`, `Clinical-stage`, `Pre-clinical`, `Commercialized`. The description should be specific enough to distinguish the company from its peers. Examples:

- `Clinical-stage; oncolytic HSV-2 virus BS001 Phase II for solid tumors, pre-revenue`
- `Commercialized; #2 in China goat milk formula (14% share), profitable`
- `NDA-stage; Ziresovir RSV F-protein inhibitor (NDA filed) for pediatric RSV`
- `Commercial-stage; A+H dual-listing candidate (A: 688062.SS), 4 marketed biologics + 9MW2821 Nectin-4 ADC (P3 urothelial/cervical)`

---

## Save-Time Invariants (Phase 5 Quality Checks)

Every save of `tracker/a1_pipeline_tracker.xlsx` must satisfy all 12 checkboxes below. If any fail, do not save — fix the in-memory state and re-run. The private workflow wrapper enforces these; this list is here so Claude Code in this repo directory can verify them independently when doing any manual touch-up.

### Data-quality checks

1. ☐ All updated rows have cols E-M filled (at minimum E/F/G/H/M).
2. ☐ **Col F zero blanks**: `sum(1 for r in data_rows if not ws.cell(r, 6).value) == 0`.
3. ☐ **Col G zero blanks**: `sum(1 for r in data_rows if not ws.cell(r, 7).value) == 0`.
4. ☐ Financial data (J/K/L) annotated with FY year in col N if not target FY.
5. ☐ **No duplicate companies** by normalized name.

### Date / sort checks

6. ☐ All col C values are `datetime.datetime` (not string).
7. ☐ All col C cells use `number_format = 'dd/mm/yyyy'`.
8. ☐ Data rows sorted by `filing_date` DESC, ties broken by company name ASC. Verify by reading top 5 and bottom 3 rows.
9. ☐ Col C dates match the HKEX JSON feed's Application Proof `d` field (for every row whose normalized name matches).

### Format checks

10. ☐ **Entire region uniform Arial 8 black** — rows 1 through last_data_row × cols A through N, populated AND blank cells:
    ```python
    from collections import Counter
    fonts = Counter((ws.cell(r,c).font.name, ws.cell(r,c).font.size)
                    for r in range(1, last_data_row+1)
                    for c in range(1, 15))
    assert set(fonts.keys()) == {('Arial', 8.0)}, f"Non-uniform fonts: {fonts}"
    ```
11. ☐ **Zero fills in data region**:
    ```python
    assert all(ws.cell(r,c).fill.patternType is None
               for r in range(1, last_data_row+1)
               for c in range(1, 15))
    ```

### Metadata check

12. ☐ Row 1 col A updated to `Updated as of <today>` in `Mon D, YYYY` format.

---

## Manual Edit Guidance

In general, **do not hand-edit `tracker/a1_pipeline_tracker.xlsx`** outside the skill. The skill enforces 12 invariants on every save; a hand-edit can easily break invariant 10 (font uniformity) or invariant 11 (zero fills) without any visible symptom. If a manual edit is genuinely necessary:

1. Open the file with `openpyxl.load_workbook()` (never `read_only=True`).
2. Make the minimal edit (set `cell.value` only).
3. **Immediately** apply the full Phase 5 uniformization pass (Arial 8 black + zero fills) to the entire region — not just the cells you touched — because openpyxl may have promoted workbook defaults on any cells it serialized.
4. Save with a new timestamp in row 1.
5. Run the 12-item QC checklist against the saved file before committing.

The safer path: invoke the private 5-phase workflow wrapper, let it drive the edit through the pipeline, and accept the Gate 2 diff before write.

---

## Important Context

- **Date source is authoritative**: col C must match the HKEX JSON feed's Application Proof entry (`nF == "Application Proof (1st submission)"`, field `d` in `dd/mm/yyyy`). Manual date entry is a common source of DD/MM/YYYY vs MM/DD/YYYY confusion — the skill always overwrites the master with the feed value.
- **Re-filings create new application IDs**: a company whose first A1 lapsed and is re-filed gets a new HKEX id, so Phase 0 sees the new filing as a fresh record. The Phase 2 dedup logic catches the duplicate by normalized name and keeps the newer row.
- **Listed companies (HKEX status = LP) drop out of Phase 0**: `fetch_lifesci_candidates` filters to active (status = A) applications. If the master has a row for a company that has since successfully listed (like Hangzhou Diagens → HKEX:02526), Phase 0 will not find it in the feed, so a manual `MANUAL_OVERRIDES` entry in the Phase 5 script sets the correct filing date. Alternatively, consider removing the row from the tracker entirely — once a company lists, it's no longer a pre-IPO pipeline item.
- **A+H dual listings**: companies already listed on a China A-share exchange (like Mabwell 688062.SS) filing for a secondary HKEX H-share listing are **H-share** by definition (the A-share listing already requires PRC joint stock status). Note the A-share code in col G.
- **Chapter 18A (-B)** = biotech allowed to list pre-revenue; **Chapter 18C (-P)** = specialist tech (AI, advanced manufacturing, etc. — some healthcare AI companies qualify). No suffix = main board listing, typically commercial-stage with meaningful revenue.
- **The font uniformity invariant (check #10) is self-healing**: by pre-setting every cell in the region (including blank cells) to Arial 8 black, the format survives any future write into a previously-blank cell. This is the fix for the Calibri 11 leakage bug observed when `v3 → v4 → v5` filled previously-blank F/G cells.

---

## Related Repositories

- [`soback26/biotech-bd-tracker`](https://github.com/soback26/biotech-bd-tracker) — cross-border out-licensing deal tracker (GC/JP/KR → Western pharma).
