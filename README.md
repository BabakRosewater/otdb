# Lead + Sold Log Analyzer (Cohort + Re-Engage)

A single-page, **browser-only** analyzer that loads **Leads** + **Sold Log** data (either encrypted BINs or raw CSVs), performs **attribution matching**, calculates KPIs + charts, and produces a prioritized **Unsold Prospects (Re-Engage)** list with “next best action” guidance.

✅ **All processing stays in the browser tab.**  
✅ Decrypts AES-GCM `.bin` files locally using Web Crypto.  
✅ Exports **XLSX** + **Unsold CSV**.

---

## What it does

1. **Load data**
   - **Encrypted Mode:** Fetches `lead.bin` + `sold_log.bin` by URL and decrypts with `.key` files.
   - **CSV Mode:** Upload raw leads + sold CSVs.

2. **Normalize and standardize columns**
   - Auto-detects common column names (lead date, sold date, source, VIN, email, phone, gross, etc.)

3. **Attribution matching (Sold → Lead)**
   - Matches in this order:
     1) AutoLeadID (if present)  
     2) VIN  
     3) Email  
     4) Phone  
     5) Name (weakest)
   - Picks best lead by date proximity to SoldDate and flags ambiguous matches.

4. **Cohort / “Sold Through” logic**
   - Lead cohort is defined by **Lead Start / Lead End** (Lead Recd Date).
   - “Unsold” is evaluated through **Count Sold Through** to prevent false unsold when sales happen after the cohort window.

5. **Unsold Prospects (Re-Engage)**
   - Priority Score = Recency + Contact + Underserved RT + Source
   - Provides a “Next Best Action” suggestion per lead.

6. **Exports**
   - XLSX: Leads (View), Sold (View), Unsold Prospects
   - Unsold CSV: re-engagement list ready for upload/work queues

---

## Tech stack

- **TailwindCSS** (UI)
- **Vanilla JS** (logic)
- **PapaParse** (CSV parsing)
- **Day.js** (date parsing)
- **Chart.js** (visualizations)
- **SheetJS (xlsx)** (exports)
- **Web Crypto API** (AES-GCM decrypt)

---

## Repository layout (recommended)

This app can be a single HTML file. For GitHub + Cloudflare Pages, the cleanest approach is:

/
├─ index.html
└─ README.md


> If you prefer, you can later split into `styles.css` + `app.js`, but it’s not required.

---

## Data requirements

### Leads CSV (common fields)

The app will try to auto-detect these columns (your headers may vary):

- Lead date: `Recd Date`, `Received Date`, `Created Date`, `Lead Date`, etc.
- Source: `Lead Source`, `Source`
- Contact: `Email`, `Phone`
- Optional: `Lead Price`, `VIN`, `Business Response Time (min)`, `Sales Rep`
- Optional vehicle detail: `Vehicle Type`, `Media Type`, `Car Make`, `Car Model`, `Trim`

If Lead Date or Source can’t be found, you’ll see a data-quality alert.

### Sold Log CSV (common fields)

Auto-detected fields include:

- Sold date: `SoldDate`, `Sold Date`, `RDR Date`
- VIN: `VIN`, `VehicleVIN`
- Contact: `Email`, `Phone`
- Gross: `FrontGross`, `BackGross`
- Rep: `SalesRepName`
- Optional: `AutoLeadID`, deal #, stock #

If Sold Date can’t be found, you’ll see an alert.

---

## Encrypted BIN mode

### BIN format

Each `.bin` must be:

- First **12 bytes**: AES-GCM IV
- Remaining bytes: AES-GCM ciphertext

### Key file format

Each `.key` must be:

- Exactly **32 bytes** (raw AES-256 key)

### Typical inputs

- Leads BIN URL: `https://raw.githubusercontent.com/<owner>/<repo>/main/lead.bin`
- Sold BIN URL:  `https://raw.githubusercontent.com/<owner>/<repo>/main/sold_log.bin`

> The app fetches BINs over HTTPS and decrypts them locally. Nothing is uploaded.

---

## How to run locally

### Option 1 — simplest (recommended)
Use a local static server (avoid `file://` due to fetch/CORS limitations):

#### Using Python
```bash
python -m http.server 8080
Then open:

http://localhost:8080

Using Node (http-server)
npx http-server -p 8080
Usage
Encrypted Mode (BINs)
Paste BIN URLs:

Leads Data URL

Sold Log Data URL

Select .key files:

Lead Key

Sold Key

Click Decrypt & Load

Click Run Analysis (auto-runs after decrypt in current build)

CSV Mode
Click Or upload raw CSVs

Upload Leads CSV

Upload Sold Log CSV

Click Run Analysis

Filters
Lead Start / Lead End: defines the lead cohort

Count Sold Through: counts sales up to this date (important for unsold accuracy)

Unsold filters:

text search (name/source/model/phone/etc.)

new/used

age buckets

sort by priority or lead date

Export
XLSX: full workbook

Unsold CSV: re-engagement list

KPI definitions (high level)
Leads (Filtered): leads within selected cohort date range

Sold Records (All): all sold records loaded (or filtered view if you extend it)

Attribution Match: matched sold records / sold view

Speed to Lead SLA: score derived from response-time buckets

Unsold Prospects: cohort leads not sold through cutoff date

Dedup Close Rate: unique customers sold / unique customers leads

Days to Sale (Med / P90): median and 90th percentile lead→sale time

Troubleshooting
“Key must be 32 bytes”
Your .key file is not exactly 32 bytes. It must be a raw 32-byte key.

“BIN too small to contain IV + ciphertext”
Your .bin does not include the 12-byte IV or is not the expected encryption format.

Many unmatched sold records
Make sure Sold Log contains at least one strong identifier (VIN, email, phone) and that Leads contain the same.

False “unsold”
Use Count Sold Through so the app counts sales after the cohort lead window.

Security & privacy
All parsing and matching occurs locally in the browser.

Decryption occurs locally via Web Crypto.

Network calls are limited to:

fetching .bin files by URL (if used)

loading CDN libraries (Tailwind, PapaParse, Day.js, Chart.js, xlsx)

Setup + Deployment (Cloudflare Pages)
Option A — Deploy from GitHub (recommended)
1) Create a GitHub repo
Add:

index.html

README.md (this file)

2) Connect Cloudflare Pages
In Cloudflare Dashboard → Workers & Pages → Pages

Click Create a project

Select Connect to Git

Choose your GitHub repo

Configure build settings:

Framework preset: None

Build command: (leave blank)

Build output directory: / (root)

Deploy

Cloudflare will host your static site at:

https://<project-name>.pages.dev

Notes
Because this is static HTML, there is no build step.

Updates happen automatically on new commits (if you keep auto-deploy enabled).

Option B — Deploy by direct upload (no Git integration)
In Cloudflare Pages → Create a project

Choose Upload assets

Upload:

index.html

(optional) any additional assets

Deploy

Using encrypted BIN URLs on Cloudflare Pages
If your BINs are hosted on GitHub:

Use raw URLs to fetch:

https://raw.githubusercontent.com/<owner>/<repo>/refs/heads/main/lead.bin

https://raw.githubusercontent.com/<owner>/<repo>/refs/heads/main/sold_log.bin

If you see fetch/CORS issues:

Ensure the URL is reachable publicly

Use the raw.githubusercontent.com domain

Avoid redirect URLs or “blob” URLs

(Optional) Recommended repo enhancements
Add a CNAME file if using a custom domain

Add LICENSE (internal use or proprietary)

Split to /assets later if you add images/logos

License / Intended use
Internal dealership operational analytics tool:

attribution insights

speed-to-lead accountability

re-engagement execution

lead source ROI overview

Support / Next enhancements (ideas)
Add UI controls for:

attribution window

ambiguity threshold

Add a “debug attribution” panel:

top unmatched reasons

rejected by window

multi-match collisions

Add source normalization mapping (canonical source groups)


