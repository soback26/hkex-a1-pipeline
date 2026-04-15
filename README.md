# HKEX A1 Pipeline

> **Monthly pre-IPO tracker for life-science companies filing Application Proofs (A1) on the Hong Kong Stock Exchange.**
> A sourcing and screening database for Chapter 18A biotech, Chapter 18C specialist tech, and main-board commercial-stage healthcare names before they list.

---

## Table of Contents

- [Purpose](#purpose)
- [Coverage Snapshot](#coverage-snapshot)
- [Repository Layout](#repository-layout)
- [Tracker Schema](#tracker-schema)
- [Update Workflow](#update-workflow)
- [Data Source](#data-source)
- [Notes](#notes)

---

## Purpose

Hong Kong has become the dominant listing venue for Chinese biotech since the 2018 Chapter 18A rule allowing pre-revenue biotech listings, and the 2023 Chapter 18C rule for specialist tech (including some healthcare-adjacent companies). Every month brings a fresh batch of Application Proof (A1) filings — the first public disclosure in the HKEX listing process — alongside re-filings from companies whose prior applications have lapsed after the 6-month HKEX window.

This tracker exists to systematically monitor that flow and answer one core question:

> **Which HKEX A1 filers are in scope for an investment-committee-ready screening — and which need to move up our priority queue because they just re-filed with updated financials or hit a new inflection point?**

Every row in this database is a life-science / healthcare company at the A1 stage (not yet priced, not yet listed). The 14-column schema captures what a generalist healthcare PM needs at a glance: corporate structure, stage, sector, sponsors, headline financials, lead asset, and an analyst highlights note that flags the one or two things that matter for the investment thesis.

---

## Coverage Snapshot

As of the most recent update (see `tracker/a1_pipeline_tracker.xlsx` row 1 metadata):

| Metric | Value |
|---|---|
| **Companies tracked** | ~150 |
| **Date range** | 2024 – 2026 |
| **Primary use case** | Pre-IPO screening, re-filing refresh, sponsor/sector mapping |
| **Sort order** | `filing_date` descending (newest A1 at top) |

**Sector mix** (approximate, as of the current snapshot):

| Sector | Share |
|---|---|
| Pharma / Biotech (incl. -B Ch.18A) | ~60% |
| MedTech / Diagnostics / Tools | ~15% |
| Services (TCM, healthcare delivery, CRO) | ~10% |
| Consumer Health (nutrition, supplements) | ~5% |
| CDMO | ~5% |
| Healthcare Tech / Specialist Tech (-P Ch.18C) | ~5% |

**Stage mix** (approximate):

| Stage | Share |
|---|---|
| Commercialized (profitable, A-share dual, TCM brands) | ~35% |
| Commercial-stage (early revenue, sub-scale) | ~25% |
| Clinical-stage (pre-revenue Ch.18A biotech, P1-P3) | ~30% |
| NDA-stage / Pre-commercial | ~10% |

---

## Repository Layout

```
hkex-a1-pipeline/
├── README.md                        # This file — project overview
├── CLAUDE.md                        # Update rules and field conventions
├── LICENSE                          # MIT
├── tracker/
│   └── a1_pipeline_tracker.xlsx     # Master tracker — 14 columns, DESC sort
├── archive/
│   └── A1 pipeline_<Mon> <Year>_update_v<N>.xlsx  # Historical monthly versions
├── raw/                             # (Reserved for monthly input files)
└── scripts/                         # (Reserved for future automation)
```

The authoritative workflow lives in the `/a1-pipeline-update` skill at [`soback26/lifesci-methodology`](https://github.com/soback26/lifesci-methodology/blob/main/skills/a1-pipeline-update/SKILL.md). This repo is the **working data warehouse** — the skill is the **methodology**.

---

## Tracker Schema

Single sheet, 14 columns:

| # | Column | What it captures |
|---|---|---|
| A | **Timestamp** | `"Updated as of <Mon D, YYYY>"` — row 1 only, refreshed each save |
| B | **Status** | Free-text row marker (e.g., `Expiring in 1 month`). Replaces the legacy highlighted-row scheme. |
| C | **Latest A1 Filing** | Date of the most recent Application Proof (1st submission). Format: `dd/mm/yyyy`, strictly from the HKEX JSON feed. |
| D | **Company Name** | English. Suffix `-B` = Chapter 18A biotech; `-P` = Chapter 18C specialist tech; no suffix = main board / commercial. |
| E | **Chinese Name** | Traditional or simplified. |
| F | **Shareholder Structure** | One of: `H-share` \| `Red Chip` \| `VIE` \| `Cayman holdco` \| `BVI holdco`. Sourced from the prospectus corporate-structure section. Zero blanks allowed. |
| G | **Business Model / Clinical Stage** | Format: `<stage>; <one-line description>`. Stage ∈ {`Commercial-stage`, `Clinical-stage`, `Pre-clinical`, `Commercialized`}. Zero blanks allowed. |
| H | **Sector** | `Pharma / Biotech`, `MedTech`, `Services`, `Diagnostics / Tools`, `Consumer Health`, `CDMO`, `Healthcare Tech` |
| I | **Sponsor** | Joint or sole sponsors (truncated to top 3 + `et al.` if 4+). |
| J | **Latest FY Revenue (RMB m)** | Full year, target FY first (default FY25). Pre-revenue: `None` or `-`. |
| K | **Latest FY Net Income (RMB m)** | Positive for profitable; negative for losses. |
| L | **Latest FYE Cash (RMB m)** | Year-end cash balance. |
| M | **Lead Asset / Business** | One-line core product or business description. |
| N | **Highlights / Updates** | Investment-memo style analyst note; FY year flagged if not target FY. Font: Arial 8 black. |

### Formatting invariants (enforced on every save)

- **Entire region `rows 1..last_data_row × cols A..N`**: Arial 8 black, `fill_type=None`. This includes blank cells — the skill applies a "self-healing" format so any future write inherits Arial 8 automatically.
- **Col C**: `number_format = 'dd/mm/yyyy'`, `datetime` type (never string).
- **Sort**: `filing_date` DESC (newest at top), secondary key = company name ASC.
- **Dedup**: normalized name across `-B`/`-P` suffix strip + corp-suffix normalization. Duplicates resolved by keeping the row with the most recent `filing_date`; data richness as tiebreak.
- **Row 1 metadata**: `Updated as of <today>` refreshed each save.

---

## Update Workflow

The tracker is refreshed **monthly** via the `/a1-pipeline-update` skill. Two modes:

### Mode A — User supplies a pre-typed monthly Excel

Drop a new Excel into `raw/` (or point the skill at it directly) with new company rows at the top containing only `D` (name) and `C` (filing date). The skill enriches the other 12 columns via prospectus + web search, classifies each candidate as `TRULY NEW` / `DATA RECOVERY` (copied from prior month) / `DUPLICATE` (re-filing), and saves a new version.

### Mode B — Phase 0 HKEX auto-scraper

Trigger phrase: *"scan HKEX for new A1s"* / *"抓一下 HKEX"*. The skill reaches into the HKEX JSON feed (`app_{YYYY}_sehk_{e|c}.json`), pulls every Application Proof record, applies a keyword + `-B`/`-P` suffix pre-filter to surface life-science candidates, and matches each candidate against the existing tracker. A **Gate 1** approval checkpoint shows the NEW / REFRESH / SKIP buckets before any PDF is downloaded. Approved candidates get their SUMMARY / BUSINESS / FINANCIAL INFORMATION chapters fetched from HKEX Multi-Files pages, parsed with `pdfplumber`, and written through the standard 5-phase pipeline.

Both modes converge at the same **Phase 4 Gate 2 diff preview** (mandatory checkpoint before any write), then run the Phase 5 save sequence:

1. Verify dates against HKEX feed (feed is authoritative)
2. Dedup by normalized name
3. Sort `filing_date` DESC
4. Force `dd/mm/yyyy` on col C
5. Force Arial 8 black on entire region + zero fills
6. Update row 1 metadata
7. Readback QC

### Commit convention

```
update: <YYYY-MM-DD> | +N new, +R refreshed, -D deduped | <one-line summary>
```

Previous monthly versions move to `archive/` with their original `A1 pipeline_<Mon> <Year>_update_v<N>.xlsx` filename for audit trail. `tracker/a1_pipeline_tracker.xlsx` is always overwritten with the latest.

---

## Data Source

**Primary**: HKEX Application Proof feed — `https://www1.hkexnews.hk/ncms/json/eds/app_{YYYY}_sehk_{e|c}.json`. Filing dates, application IDs, and document URLs come directly from this feed. The scraper treats the feed as authoritative: when the master disagrees with the feed, the master is wrong and gets overwritten (with user approval via Gate 2).

**Secondary** (for field enrichment when Phase 0 can't extract from PDFs):
1. HKEX Multi-Files prospectus pages (SUMMARY / BUSINESS / FINANCIAL INFORMATION chapters)
2. Company corporate website
3. Chinese financial news (sina finance, cls.cn, phirda, 医药魔方)
4. English databases (PitchBook, BioCentury) as last resort

**Prospectus chapter extraction**: `pdfplumber` on FINANCIAL for J/K/L; pdfplumber + regex heuristics on SUMMARY/BUSINESS for F/G/H/I/M; assembled N with FY marker. Chapters only — the skill never downloads the full prospectus (30-60 MB each) — just the three relevant sections (~5-10 MB total per candidate).

---

## Notes

- This is a **working repository**. Tracker contents reflect curated analyst judgment on companies at the A1 stage and are based on public HKEX filings.
- All company data is sourced from public HKEX documents and news sources; **no MNPI** is captured here.
- Highlights / Updates notes (col N) are **internal opinions** for screening purposes only and do not constitute investment recommendations.
- Col F taxonomy, Col G format, and the 12-item QC checklist are subject to periodic revision — see `CLAUDE.md` for the current canonical version, and the [`/a1-pipeline-update` skill](https://github.com/soback26/lifesci-methodology/blob/main/skills/a1-pipeline-update/SKILL.md) for the authoritative workflow.
- Related repositories:
  - [`soback26/biotech-bd-tracker`](https://github.com/soback26/biotech-bd-tracker) — cross-border out-licensing deal tracker (GC/JP/KR → Western pharma)
  - [`soback26/lifesci-methodology`](https://github.com/soback26/lifesci-methodology) — skills, frameworks, and reference data for life-science investing
