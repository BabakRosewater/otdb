OTDB Dashboard — Development Documentation
1) Overview

Project name: OTDB Dashboard (MTD) — Rep KPIs + Hygiene
Purpose: A single-page web dashboard that loads an OTDB export (otdb.csv) from GitHub, defaults to Month-To-Date (MTD), and produces:

Whole-store OTDB performance metrics

Salesperson performance table + charts

Lead source performance table + charts

Execution metrics (speed-to-lead, SLA coverage)

Hygiene metrics (Bad/Duplicate tracking) separated from performance

Raw filtered table with search + paging

CSV export of filtered rows and aggregated KPI tables

Key principle:
Performance denominators use OTDB = Total − Bad.
Bad leads (including duplicates) are excluded from OTDB performance denominators but still tracked as hygiene.

2) Data Source & Assumptions
2.1 Default data location

The app loads a CSV from a raw GitHub URL:

DEFAULT_URL = https://raw.githubusercontent.com/BabakRosewater/otdb/main/otdb.csv

The UI allows changing the URL at runtime.

2.2 Expected CSV structure

The code is resilient to column name differences using auto column mapping.

Primary fields used (best-case):

Created or other “created” variant

FirstAttemptedContactUTC (or similar)

UserFirstName, UserLastName (preferred rep identity)

Lead source/type (e.g., LeadSourceName, LeadTypeName)

Status type/name:

LeadStatusTypeName (e.g., Active, Sold, Lost, Bad)

LeadStatusName (reason string; duplicates often appear here)

Optional gross:

FrontGross, BackGross

Optional trade capture:

TradeMake, TradeModel

2.3 Column mapping strategy

inferColumnMap() uses:

Exact aliases (preferred)

Pattern matching (fallback via regex)

If rep name fields are blank, rep becomes "(Unassigned)".

3) Runtime Flow (How the App Works)
3.1 Boot sequence

Set URL input to DEFAULT_URL

Fetch CSV text (fetch(url, { cache: "no-store" }))

Parse CSV with PapaParse (header:true)

Infer column map

Normalize rows into typed objects

Populate filter dropdowns (rep/source)

Set default date filter to Month-To-Date

Apply filters and render all components

3.2 MTD default logic (important)

setDefaultDatesToMTDFromRows(rows):

Finds newest Created date in the file (maxD)

Sets:

StartDate = first day of maxD’s month

EndDate = maxD’s date

This avoids problems when:

“Today” is ahead of the export

The file only contains partial days

3.3 Rendering pipeline

applyFilters() → updates state.filtered → renderAll():

KPI tiles (whole store)

Rep KPI table

Source KPI table

Charts (rep close %, rep volume, source close %, duplicates by source)

Lost reason + Bad reason tables

Raw filtered table with paging/search

Unassigned diagnostics panel

Status chips (row counts, range, mapping summary)

4) Data Model
4.1 State object
state = {
  rawRows: [],
  cols: [],
  colMap: {},
  rows: [],       // normalized objects
  filtered: [],   // filtered normalized objects
  sort: { rep:{...}, src:{...} },
  raw: { page, pageSize, search },
  charts: {}
}

4.2 Normalized row shape

Each CSV row becomes:

createdTs (Date | null)

firstAttemptTs (Date | null)

rep (string; uses UserFirstName/UserLastName then fallback)

source (string)

statusType, statusName

booleans: isBad, isSold, isLost, isActive, isDuplicate

optional metrics: frontGross, backGross, totalGross, hasTrade

raw (original CSV row object)

rep name fragments: repFirst, repLast

5) KPI Definitions (Truth Table)
5.1 Core counts

Total Leads = rows.length

Bad = count(isBad)

OTDB = Total − Bad

Sold = count(isSold)

Lost = count(isLost)

Active = count(isActive)

Duplicate = count(isDuplicate) where duplicate means:

isBad === true AND statusName contains “duplicate” (case-insensitive)

5.2 Performance close rates

OTDB Close % = Sold ÷ OTDB

Close (Closed) = Sold ÷ (Sold + Lost)

5.3 Hygiene rates

Bad % = Bad ÷ Total

Duplicate % (Total) = Dup ÷ Total

Dup% of Bad = Dup ÷ Bad

5.4 Execution (speed-to-lead)

Computed on OTDB rows only (non-bad):

Attempt Coverage % = OTDB with FirstAttempt / OTDB

minsToFirst = (firstAttemptTs − createdTs) / 60000, only if both exist and non-negative

Median Speed (min) = median(minsToFirst)

SLA ≤15m % = count(minsToFirst ≤ 15) ÷ OTDB

SLA ≤60m % = count(minsToFirst ≤ 60) ÷ OTDB

Note: SLA uses OTDB as denominator, so missing attempts count as misses.

5.5 Aging metrics

On OTDB Active rows with createdTs:

ageDays = (now − createdTs) / 86400000

Active >7d = count(ageDays > 7)

Bucket counts: 0–2, 3–7, 8–14, 15+

5.6 Trade capture

On OTDB rows:

hasTrade = TradeMake || TradeModel

Trade Capture % = count(hasTrade) ÷ OTDB

5.7 Gross metrics (optional)

On Sold rows:

totalGross = frontGross + backGross if either exists

Avg Front / Back / Total = averages across sold rows

6) Filters & UX Rules
6.1 Date filtering

If either start or end date is set:

Rows missing createdTs are excluded

Otherwise, rows without created date can still exist in the set

6.2 Multi-select filters

Rep filter uses normalized rep

Source filter uses normalized source

Ctrl/Cmd + click for multiple selections

6.3 Reset behavior

Reset clears rep/source selection, search, overrides, min volume, and returns date filters to MTD.

7) Charts

Uses Chart.js with safe teardown to prevent stacking:

destroyChart(key) destroys existing chart instance

Then recreates new chart with updated data

Charts included:

Rep OTDB Close% (bar)

Rep OTDB Leads + Sold (bar, 2 datasets)

Source OTDB Close% (top 30)

Duplicate count by source (top 30)

8) Exports
8.1 Export filtered rows

Exports the exact CSV rows that are currently filtered:

Filename: otdb_filtered_rows.csv

8.2 Export Rep/Source KPI tables

Exports aggregated KPI rows:

otdb_rep_kpis.csv

otdb_source_kpis.csv

Export uses:

A simple CSV serializer with quote escaping

downloadText() with Blob + hidden anchor

9) Extending the Project (Common Enhancements)
9.1 Add new KPIs

Add calculations to computeKpis(rows)

Add display tiles in renderTiles(summary)

Add columns in renderKpiTable(...) and exports

9.2 Add “Appointments Set/Shown/Sold” logic

If your OTDB extract has appointment fields:

Map columns in inferColumnMap()

Parse to ints in normalizeRows()

Track rates per rep/source in computeKpis()

9.3 Add “Floor vs Internet” segmentation

If lead source contains usable categories:

Create a derived field channel in normalize

Add a filter dropdown + chart breakdown

10) Deployment & Hosting
10.1 Local / static hosting

This is a pure static page.

Works from local file if CORS allows fetch to GitHub raw

Best hosted:

GitHub Pages

Netlify

Render static site

10.2 Updating the dataset

Replace otdb.csv in the repo (BabakRosewater/otdb)

App reloads and recalculates immediately

11) Troubleshooting

Dashboard loads but shows “0 rows”

Confirm the raw GitHub URL is correct (must be raw.githubusercontent.com/...)

Confirm CSV has headers and rows

Check console for PapaParse errors

Dates look wrong / empty

Confirm which “Created” column exists and if it’s parseable (YYYY-MM-DD, ISO, or US date formats)

If H-shaped formats appear, extend parseDateAny()

Reps show as “Unassigned”

CSV rows missing UserFirstName/UserLastName

Turn on “Explain Unassigned” for counts

Charts keep stacking

Ensure destroyChart() is called before re-render (already implemented)

12) Change Log Pattern (Recommended)

Maintain a simple CHANGELOG.md with:

Date

Dataset changes (new columns, mapping changes)

KPI changes (definitions updated)

UI changes (filters, charts)

If you want, I can also produce a README.md version ready to drop into the GitHub repo (with screenshots sections, install/deploy steps, and a KPI glossary).

You said:
Yes lets do the readme virsion for git hub 
ChatGPT said:
# OTDB Dashboard (MTD) — Rep KPIs + Hygiene

A lightweight, single-page dashboard that loads your **OTDB CSV** from GitHub, defaults to **Month-To-Date (MTD)**, and produces **salesperson + source KPIs** while separating **performance** from **hygiene** (Bad/Duplicate).

---

## What this does

- ✅ Pulls OTDB data from a **raw GitHub CSV** (`otdb.csv`)
- ✅ Auto-detects common column names (Created, Status, Lead Source, Rep name, etc.)
- ✅ Defaults the date filter to **MTD of the dataset’s latest date**
- ✅ **Excludes Bad leads from OTDB denominators** (so spam/duplicate routing doesn’t punish performance)
- ✅ Tracks **Duplicates as hygiene**, not performance
- ✅ Builds:
  - KPI tiles (whole store)
  - Salesperson KPI table + charts
  - Lead Source KPI table + charts
  - Lost reasons + Bad reasons tables
  - Raw filtered data with search + paging
  - CSV exports (filtered rows, rep KPIs, source KPIs)

---

## Live Data Source

Default CSV URL used by the app:

- `https://raw.githubusercontent.com/BabakRosewater/otdb/main/otdb.csv`

You can change it in the UI at runtime via the **Data URL (CSV)** input.

### Where to change it in code
Look for:

```js
const DEFAULT_URL = "https://raw.githubusercontent.com/BabakRosewater/otdb/main/otdb.csv";

MTD Default Logic (Important)

On first load, the dashboard sets the date filter to MTD of the current month in the dataset:

Finds the latest Created date present in the CSV (maxD)

Sets:

StartDate = first day of maxD’s month

EndDate = maxD’s day

This is safer than using “today” because your export may not include today’s rows yet.

KPI Definitions (Glossary)
Counts

Total Leads = all rows in the filtered set

Bad = rows where status type is Bad

OTDB Leads = Total − Bad

Sold / Lost / Active = based on status type

Close Rates

OTDB Close % = Sold ÷ OTDB

Close (Closed) = Sold ÷ (Sold + Lost)

Hygiene (tracked separately)

Bad % = Bad ÷ Total

Duplicate % (Total) = Dup ÷ Total

Dup% of Bad = Dup ÷ Bad

Duplicate rows are detected as: Bad + the status name contains duplicate (case-insensitive).

Execution (Speed-to-lead)

All execution metrics use OTDB rows (non-bad):

Attempt Coverage % = OTDB with a first attempt timestamp ÷ OTDB

Median Speed-to-Lead (min) = median minutes between Created → First Attempt

SLA ≤15m % = count(minsToFirst ≤ 15) ÷ OTDB

SLA ≤60m % = count(minsToFirst ≤ 60) ÷ OTDB

Aging

On OTDB Active rows:

Active >7d = count of active OTDB rows older than 7 days

Also bucket counts: 0–2, 3–7, 8–14, 15+

Trade Capture

Trade Capture % = OTDB rows with TradeMake or TradeModel ÷ OTDB

Gross (optional)

If your CSV contains front/back gross:

Avg Front / Back / Total across Sold rows

Required / Supported Columns

This project auto-detects columns with a mapping function, so the exact header names can vary.

Key fields (best case)

Created timestamp:

Created, LeadCreatedUTC, LeadCreated, CreatedUTC

First attempt timestamp:

FirstAttemptedContactUTC, FirstAttemptedContact

Rep name (preferred):

UserFirstName, UserLastName

Lead source/type:

LeadSourceName, LeadTypeName

Status:

LeadStatusTypeName (Bad, Sold, Lost, Active)

LeadStatusName (reason, includes “Duplicate” often)

Optional fields

FrontGross, BackGross, SellingPrice, DealNumber

TradeMake, TradeModel

Why “Unassigned” reps happen

Salesperson is built from:

UserFirstName + UserLastName

fallback to other rep columns (AssignedTo/Owner/etc.)

if missing → "(Unassigned)"

If you see lots of Unassigned:

those rows likely have blank rep name fields in the export

the dashboard includes an Unassigned diagnostics panel to quantify missing first/last/both

Features
Filters

Date range (Start / End)

Salesperson multi-select

Source multi-select

Min volume threshold (for “low n” confidence)

Now override (for aging calculations)

Charts

OTDB Close % by Salesperson

OTDB Leads + Sold by Salesperson

OTDB Close % by Source (top 30)

Duplicate Count by Source (top 30)

Tables

Salesperson KPI table (sortable)

Lead Source KPI table (sortable)

Lost Reasons (OTDB only)

Bad Reasons (hygiene only)

Raw filtered data (search + paging)

Exports

Export filtered rows as CSV

Export Rep KPI table as CSV

Export Source KPI table as CSV

Local Use

This is a static HTML file. You can run it:

from a local file (may be subject to browser CORS rules)

or via a simple local server:

Option A — Python
python -m http.server 8080


Open:

http://localhost:8080

Option B — Node
npx serve

Deployment Options

GitHub Pages (recommended)

Netlify

Render (Static site)

If using GitHub Pages:

Put the HTML file in repo root (or /docs)

Enable Pages in repo settings

Confirm the DEFAULT_URL points to your raw otdb.csv

Updating the dataset

Replace/commit the CSV here:

otdb.csv in this repo:

BabakRosewater/otdb

Then reload the dashboard.

Pro tip: keep the filename the same (otdb.csv) so the dashboard never needs code updates.

Customization Guide
Change the default dataset URL

Edit:

const DEFAULT_URL = "https://raw.githubusercontent.com/BabakRosewater/otdb/main/otdb.csv";

Add a KPI

Add calculation in computeKpis(rows)

Add columns in KPI table headers in renderKpiTable(...)

Add to exports in exportKpiTable(...)

(Optional) add a tile in renderTiles(...)

Add new column mapping

Add to inferColumnMap(cols) using either:

exact aliases list

regex include patterns

Known Notes / Design Decisions

Bad leads do not count against OTDB closing performance

Duplicates are treated as hygiene

MTD is based on the dataset’s newest date (not system date)

SLA calculations use OTDB denominator so missing attempts count as misses

Charts are destroyed before redraw to prevent stacking

Roadmap Ideas

Appointment funnel metrics (Set / Shown / Sold) if OTDB export includes those fields

Channel segmentation (Internet vs Phone vs Walk-in)

Lead status analysis (pipeline health) if a true “Lead Status” field exists

Rep leaderboard view + trend over time (requires historical CSVs)

Support / Troubleshooting
Data won’t load

Ensure the URL is raw GitHub:

raw.githubusercontent.com/...

Confirm the CSV has headers and content

Open devtools console to see fetch or parsing errors

Dates missing / wrong

Confirm the export has a usable “Created” column format

If needed, extend parseDateAny() to support your exact timestamp format

Too many Unassigned reps

Confirm the export includes UserFirstName and UserLastName

Use the Unassigned diagnostics panel to confirm what’s missing


If you want, I can also include a **“Screenshots”** section scaffold and a **sample CSV header snippet** that matches your current `otdb.csv` so your team knows what “good” looks like.
::contentReference[oaicite:0]{index=0}
