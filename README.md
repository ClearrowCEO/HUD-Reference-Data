# HUD Reference Data

Public mirror of HUD reference files used by [Broker's Portfolio](https://github.com/ClearrowCEO/BrokersPortfolio)'s
ingestion pipelines. `huduser.gov` sits behind an AWS WAF bot-challenge that blocks automated
fetching (confirmed by extensive testing), so this repo exists purely so Supabase's ingestion
functions have a plain, unauthenticated URL to pull from — Postgres/edge functions have real
internet access, they just can't get past HUD's own bot-check.

## Naming convention

Every file here is named **`<job-description>_<period>.<ext>`**, so it's obvious which ingestion
job consumes it and which vintage it is, without opening the file:

- `<job-description>` — the Supabase `*_ingest_sources` `source_key` (or a close, readable
  version of it) the file feeds. Today: `hud-zip-county-crosswalk` (source_key `zip_county` in
  `hud_ingest_sources`, consumed by the `hud-crosswalk-ingest` edge function).
- `<period>` — the vintage the file covers, `YYYY-MM` (e.g. `2026-06`). HUD republishes this
  crosswalk quarterly; use the month HUD itself labels the file with.
- `<ext>` — actual file format (`.csv`, `.xlsx`, `.zip`, ...).

## Layout

- **Top level** — the file each ingestion job's `override_url` actually points to. Prefer `.csv`
  over `.xlsx` here when possible: the ingestion functions stream-parse CSV with near-zero memory
  overhead, while `.xlsx` has to be fully parsed into memory (SheetJS) first and can hit Supabase
  Edge Functions' memory limit even on files that aren't especially large.
- **`raw/`** — the original file exactly as downloaded/received, kept for audit/reference. Not
  fetched by any ingestion job directly.

## Updating for a new quarter

1. Get the new HUD ZIP-to-COUNTY crosswalk file (huduser.gov's bot-check means this currently
   requires a real browser session to click through).
2. Add it to `raw/` as `hud-zip-county-crosswalk_<new-period>.<ext>`.
3. If it's not already a plain CSV, convert it to one with headers `ZIP, COUNTY,
   USPS_ZIP_PREF_STATE, RES_RATIO, BUS_RATIO, OTH_RATIO, TOT_RATIO` (these are the column names
   `hud-crosswalk-ingest` looks for) and add that as `hud-zip-county-crosswalk_<new-period>.csv`
   at the top level.
4. Update `hud_ingest_sources.override_url` (table in the Broker's Portfolio Supabase project) to
   the new file's raw GitHub URL, and re-run the `hud-crosswalk-ingest` function.
