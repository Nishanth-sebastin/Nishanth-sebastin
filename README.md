# Job Fetcher

Fetches the **last 24 hours** of job postings for a role across many sources and
appends them to an Excel sheet (or Google Sheet) with filter dropdowns.

## Sources

| Type | Sources | Reliability |
|------|---------|-------------|
| Free APIs | Adzuna*, Jooble*, Remotive, RemoteOK, Arbeitnow, The Muse, Jobicy | Stable |
| Company feeds (ATS) | Greenhouse, Lever, Ashby | Stable, no keys |
| Browser scrapers | **LinkedIn, Naukri, Wellfound, Indeed** | Fragile (see note) |

`*` need a free API key. Toggle any source on/off in `config.yaml`.

> **Scraper note:** LinkedIn & Indeed scrape reliably in testing; **Naukri &
> Wellfound are often Cloudflare/login-gated and return 0** — that's expected,
> not a bug. A broken scraper never crashes the run; it just yields nothing.

## What it does

1. Run it → it asks `Enter role:` → type a role + Enter.
2. Queries every enabled source across **all your cities** (see `locations`),
   keeping only postings from the last 24h.
3. Writes to a workbook with **two tabs**:
   - **`Jobs`** — active leads. New jobs append below previous ones. Existing
     not-applied rows are **never deleted**; their **`Age`** column refreshes
     each run (a job from yesterday shows "2 days ago" today).
   - **`Applied`** — when you set a row's **`Applied`** column to `Yes`, the
     **next run moves it here** (stamped with `Applied At`) and removes it from
     `Jobs`. It won't come back as a new lead.
4. Filter dropdowns on every column → filter by Platform, Role, City, Posted
   date, Age, Applied status, right inside Excel/Sheets.

### The applied workflow
- Open `jobs.xlsx`, browse the `Jobs` tab.
- Applied to one? Type `Yes` in its `Applied` column, save.
- Next run: it moves to the `Applied` tab; everything you didn't apply to stays
  put (with refreshed ages); new jobs get added below.

## Setup

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/python -m playwright install chromium   # for the browser scrapers
```

## Run

```bash
.venv/bin/python jobfetch.py                      # prompts for the role
.venv/bin/python jobfetch.py --role "Data Analyst"
.venv/bin/python jobfetch.py --days 3             # widen the window
.venv/bin/python jobfetch.py --remote-only
```

## Config (`config.yaml`)

- **`locations:`** — the cities to search. LinkedIn, Indeed and Adzuna query
  each city; Jooble/remote APIs use the broad `location`. Edit this list freely.
- **`sources:`** — turn each source on/off. Disable `naukri`/`wellfound` if their
  0-results noise bothers you.
- **`adzuna` / `jooble`** — paste free API keys for strong India coverage:
  - Adzuna: https://developer.adzuna.com  (best India API)
  - Jooble: https://jooble.org/api/about
- **`ats:`** — list company "board tokens" (the slug in their careers URL) for
  Greenhouse/Lever/Ashby. Add the startups you care about.
- **`defaults:`** — `days` (window), `location`, `remote_only`, per-source cap.

## Output: Excel (default) or Google Sheets

Choose the output in **`config.local.yaml`** → `output.backend`:
`excel` (local file) or `gsheet` (cloud). Secrets/keys also live in
`config.local.yaml`, which is git-ignored and never pushed.

### Cloud Google Sheets — one-time setup (~10 min)

1. Go to https://console.cloud.google.com → create a project (any name).
2. **APIs & Services → Library** → enable **Google Sheets API** and
   **Google Drive API** (search each, click Enable).
3. **APIs & Services → Credentials → Create credentials → Service account**.
   Give it a name, Create, Done.
4. Click the new service account → **Keys → Add key → Create new key → JSON**.
   A `.json` file downloads. Rename it to **`service_account.json`** and put it
   in this folder (it's git-ignored — stays private).
5. Open that JSON, copy the `"client_email"` value
   (looks like `name@project.iam.gserviceaccount.com`).
6. Create a Google Sheet (e.g. name it **"Job Tracker"**). Click **Share**,
   paste that email, give it **Editor**, Send.
7. In `config.local.yaml` set `output.gsheet.spreadsheet` to the sheet's exact
   name (`"Job Tracker"`), and make sure `output.backend: gsheet`.

Now `findjobs` writes straight to that cloud sheet — open it on any device. It
creates the **Jobs** and **Applied** tabs automatically on the first run.

## Tips

- `Applied` defaults to `No`; edit it to `Yes` — reruns won't overwrite (matched by URL).
- Delete `jobs.xlsx` to start a fresh sheet.
- Schedule a daily top-up with `cron`:
  `0 9 * * * cd /path/to/jobapply && .venv/bin/python jobfetch.py --role "Python Developer"`
