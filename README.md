````md
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
````

---

## MTD Default Logic (Important)

On first load, the dashboard sets the date filter to **MTD of the current month in the dataset**:

1. Finds the **latest `Created` date** present in the CSV (`maxD`)
2. Sets:

   * `StartDate = first day of maxD’s month`
   * `EndDate   = maxD’s day`

This is safer than using “today” because your export may not include today’s rows yet.

---

## KPI Definitions (Glossary)

### Counts

* **Total Leads** = all rows in the filtered set
* **Bad** = rows where status type is Bad
* **OTDB Leads** = `Total − Bad`
* **Sold / Lost / Active** = based on status type

### Close Rates

* **OTDB Close %** = `Sold ÷ OTDB`
* **Close (Closed)** = `Sold ÷ (Sold + Lost)`

### Hygiene (tracked separately)

* **Bad %** = `Bad ÷ Total`
* **Duplicate % (Total)** = `Dup ÷ Total`
* **Dup% of Bad** = `Dup ÷ Bad`

> Duplicate rows are detected as: `Bad` + the status name contains `duplicate` (case-insensitive).

### Execution (Speed-to-lead)

All execution metrics use **OTDB** rows (non-bad):

* **Attempt Coverage %** = `OTDB with a first attempt timestamp ÷ OTDB`
* **Median Speed-to-Lead (min)** = median minutes between Created → First Attempt
* **SLA ≤15m %** = `count(minsToFirst ≤ 15) ÷ OTDB`
* **SLA ≤60m %** = `count(minsToFirst ≤ 60) ÷ OTDB`

### Aging

On **OTDB Active** rows:

* **Active >7d** = count of active OTDB rows older than 7 days
* Also bucket counts: `0–2`, `3–7`, `8–14`, `15+`

### Trade Capture

* **Trade Capture %** = `OTDB rows with TradeMake or TradeModel ÷ OTDB`

### Gross (optional)

If your CSV contains front/back gross:

* **Avg Front / Back / Total** across Sold rows

---

## Required / Supported Columns

This project **auto-detects** columns with a mapping function, so the exact header names can vary.

### Key fields (best case)

* Created timestamp:

  * `Created`, `LeadCreatedUTC`, `LeadCreated`, `CreatedUTC`
* First attempt timestamp:

  * `FirstAttemptedContactUTC`, `FirstAttemptedContact`
* Rep name (preferred):

  * `UserFirstName`, `UserLastName`
* Lead source/type:

  * `LeadSourceName`, `LeadTypeName`
* Status:

  * `LeadStatusTypeName` (Bad, Sold, Lost, Active)
  * `LeadStatusName` (reason, includes “Duplicate” often)

### Optional fields

* `FrontGross`, `BackGross`, `SellingPrice`, `DealNumber`
* `TradeMake`, `TradeModel`

---

## Why “Unassigned” reps happen

Salesperson is built from:

1. `UserFirstName + UserLastName`
2. fallback to other rep columns (AssignedTo/Owner/etc.)
3. if missing → `"(Unassigned)"`

If you see lots of Unassigned:

* those rows likely have blank rep name fields in the export
* the dashboard includes an **Unassigned diagnostics** panel to quantify missing first/last/both

---

## Features

### Filters

* Date range (Start / End)
* Salesperson multi-select
* Source multi-select
* Min volume threshold (for “low n” confidence)
* Now override (for aging calculations)

### Charts

* OTDB Close % by Salesperson
* OTDB Leads + Sold by Salesperson
* OTDB Close % by Source (top 30)
* Duplicate Count by Source (top 30)

### Tables

* Salesperson KPI table (sortable)
* Lead Source KPI table (sortable)
* Lost Reasons (OTDB only)
* Bad Reasons (hygiene only)
* Raw filtered data (search + paging)

### Exports

* Export filtered rows as CSV
* Export Rep KPI table as CSV
* Export Source KPI table as CSV

---

## Local Use

This is a **static HTML** file. You can run it:

* from a local file (may be subject to browser CORS rules)
* or via a simple local server:

### Option A — Python

```bash
python -m http.server 8080
```

Open:

* `http://localhost:8080`

### Option B — Node

```bash
npx serve
```

---

## Deployment Options

* GitHub Pages (recommended)
* Netlify
* Render (Static site)

If using GitHub Pages:

1. Put the HTML file in repo root (or `/docs`)
2. Enable Pages in repo settings
3. Confirm the `DEFAULT_URL` points to your raw `otdb.csv`

---

## Updating the dataset

Replace/commit the CSV here:

* `otdb.csv` in this repo:

  * `BabakRosewater/otdb`

Then reload the dashboard.

> Pro tip: keep the filename the same (`otdb.csv`) so the dashboard never needs code updates.

---

## Customization Guide

### Change the default dataset URL

Edit:

```js
const DEFAULT_URL = "https://raw.githubusercontent.com/BabakRosewater/otdb/main/otdb.csv";
```

### Add a KPI

1. Add calculation in `computeKpis(rows)`
2. Add columns in KPI table headers in `renderKpiTable(...)`
3. Add to exports in `exportKpiTable(...)`
4. (Optional) add a tile in `renderTiles(...)`

### Add new column mapping

Add to `inferColumnMap(cols)` using either:

* exact aliases list
* regex include patterns

---

## Known Notes / Design Decisions

* **Bad leads do not count against OTDB closing performance**
* **Duplicates are treated as hygiene**
* MTD is based on the dataset’s newest date (not system date)
* SLA calculations use OTDB denominator so missing attempts count as misses
* Charts are destroyed before redraw to prevent stacking

---

## Roadmap Ideas

* Appointment funnel metrics (Set / Shown / Sold) if OTDB export includes those fields
* Channel segmentation (Internet vs Phone vs Walk-in)
* Lead status analysis (pipeline health) if a true “Lead Status” field exists
* Rep leaderboard view + trend over time (requires historical CSVs)

---

## Support / Troubleshooting

### Data won’t load

* Ensure the URL is **raw** GitHub:

  * `raw.githubusercontent.com/...`
* Confirm the CSV has headers and content
* Open devtools console to see fetch or parsing errors

### Dates missing / wrong

* Confirm the export has a usable “Created” column format
* If needed, extend `parseDateAny()` to support your exact timestamp format

### Too many Unassigned reps

* Confirm the export includes `UserFirstName` and `UserLastName`
* Use the Unassigned diagnostics panel to confirm what’s missing

---

```

If you want, I can also include a **“Screenshots”** section scaffold and a **sample CSV header snippet** that matches your current `otdb.csv` so your team knows what “good” looks like.
::contentReference[oaicite:0]{index=0}
```
