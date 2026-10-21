# Texas Early Vote 2026 dashboard — daily refresh runbook

Published artifact (update in place, never create a new one):
**https://claude.ai/artifact/QAi94b3ww9gV74yvBYN43N** — title "Texas Early Vote 2026".

The page is a single self-contained HTML file built by `build.py` from `template.html`, `app.js`, seven JSON
data files and one static geometry file `data/counties.json` (Texas county outlines as SVG paths, made once from
the Census cb_2023_us_county_20m shapefile by `geo/make_geo.py`; it never changes). All source files plus `SPEC.md`,
the merge helpers and `counties.json` are stored in this Project under `claude/texas-ev-dashboard/`.

## Steps (about 20 minutes; the BigQuery queries are the slow part)

1. Make a working folder in the scratchpad, e.g. `texas/` with a `data/` subfolder. Fetch the project docs
   `claude/texas-ev-dashboard/{SPEC.md,build.py,template.html,app.js,merge_county.py,merge_precinct.py,heat_merge.py,counties.json,REFRESH.md}`
   with `project_read` and write each to the working folder under its own file name (`build.py` expects
   `template.html` and `app.js` beside it, and `counties.json` inside `data/` with the query outputs).
2. Re-run the queries in `SPEC.md`. Only 2026 data changes day to day, but `voters` is rebuilt occasionally, so
   re-run everything each time. Run the queries yourself with `mcp__Google_Cloud_BigQuery__execute_sql_readonly`
   (parameter name is `query`, not `sql`). Every result that matters must be SAVED TO A FILE by the harness
   (it does that automatically for large results; add a `REPEAT('x',10000) pad` column to small ones such as
   Query S), then converted with the helper scripts — never retype numbers from inline output:
   - `daily.json`: Query S ×3 years (+ pad) and the totals query → a short Python script (see the 2026-10-20 run)
   - `county_daily_2026.json`: Query C 2026 in one query (+ pad) → `merge_county.py out.json result.txt`
   - `county_daily_2024.json`, `county_daily_2022.json`: Query C in 4 county chunks each → `merge_county.py out.json chunk1 chunk2 ...`
   - `heat_2026.json`, `heat_2022.json`: Query H in 2 chunks each (ASOF = the as-of day; 15 on 2026-10-19) → `heat_merge.py out.json chunk1 chunk2`
   - `precinct_2026.json`: Query P in 6+ chunks → `merge_precinct.py`
   Cross-check: county sums must equal the statewide Query S sums (build.py warns if not); heat `ev_now` total
   equals statewide early votes minus the unmatched rows (722,697 = 731,046 − 8,349 on 2026-10-19).
3. `python3 build.py` → `dashboard.html` and `index.html`. It prints the as-of date (last complete day). If the
   as-of date did not move since the last run, the feed has not updated — republishing is harmless, but say so.
4. Spot-check one number independently, e.g.
   `SELECT COUNTIF(Rejected=0 AND ReturnDate BETWEEN '2026-01-01' AND '<asof>') + COUNTIF(EarlyVoted BETWEEN '2026-01-01' AND '<asof>') FROM abev_2026 WHERE State='TX'`
   must equal the "Early votes cast" tile (731,046 on 2026-10-19; Harris County 128,943).
5. Screenshot once with Playwright (`shot.py`, light, 1200 wide) and look at it; then publish with the Artifact tool:
   `action: publish`, `url: https://claude.ai/artifact/QAi94b3ww9gV74yvBYN43N`, `file_path: <dashboard.html>`,
   `label: "Data through <as-of date>"`, and `files: {"site/index.html": {"from": "<abs path to index.html>", "contentType": "text/html"}}`.
   Read the artifact first if the tool says it has not been read in this conversation. Do not pass `icon`.
   A publish that passes only these keeps the 254 `precincts/*.json` files already on the artifact.
6. If `template.html` or `app.js` had to change, write the changed file back to the Project at the same path
   (and to the Ohio and Iowa kits: the three are identical apart from `build.py`'s STATE block).
7. Report: as-of date, statewide early votes (and change vs 2024 same point), anything odd (rejected rows appearing,
   a change in the county list, a query that hit the 3,000-row cap).

## Data notes that matter
- Texas has no party registration: party = `PartyRollUp` modeled labels (`Modeled Republican` / `Modeled Democrat` /
  `Modeled Independent/Unaffiliated`), from the voter file of each year. The ABEV party flags are all zero in TX.
- Texas's ABEV feed records a mail ballot only when returned: `RequestDate` = `ReturnDate` on every row, so
  requested = returned and there are no outstanding ballots (`STATE['no_requests'] = True` swaps that tile for
  "Mail share of early vote").
- In-person early voting runs 17 to 5 days before Election Day (2026-10-19 to 2026-10-30). The first EV day is
  d=15 in all three years, so same-point comparisons line up.
- `Rejected` is 0 on every Texas row (all years); the exclude-rejected rule is still applied.
- County spelling: the feed and voter file use DE WITT and LASALLE; the Census (and `counties.json`) use DEWITT and
  LA SALLE. `merge_county.py`, `heat_merge.py` and `build.py` all map to the Census spelling.
- Five small counties had no 2026 rows on 2026-10-19 (CONCHO, CROCKETT, DELTA, LOVING, MARTIN); build.py gives them
  empty day lists. Jurisname is mixed case in Texas; the SQL upper-cases it.
- Unmatched ABEV rows (no voter in `voters`): 8,349 of 731,046 in 2026 (1.1%), counted as modeled independent.

## Precinct maps
The county view shows a precinct map. Its geometry lives in 254 files on the artifact (`precincts/<COUNTY>.json`,
spaces as underscores; 6.0 MB total) and the page fetches them on demand. They were built once by
`geo/match_precincts.py` (stored as `claude/texas-ev-dashboard/match_precincts.py`): 2024 general-election precinct
polygons from The New York Times' precinct map (`TX-precincts-with-results.geojson.gz`, 9,718 shapes), matched to the
voter file's `PrecinctCode` by the Census 2020 blocks of each precinct's voters (`voter_contact.CensusBlock2020`, top 8
blocks per precinct via a STRING_AGG query in 4 county chunks; block interior points from TIGER `tl_2020_48_tabblock20.dbf`).
Result on 2026-10-20: 9,385 of 9,727 codes placed; 169 weak (<60% of voters' blocks in the chosen shape); 372 shapes
hold several codes; 408 unplaced, of which 254 are the per-county `99999` placeholder code (not a real area, skipped on
purpose) and 66 have no voter with a census block. Dallas County redrew its precincts after 2024 (737 codes vs 1,230
NYT shapes), so about 500 Dallas shapes show no 2026 data. Rebuild only if Texas precinct codes change materially.

## Public site
No GitHub Pages site yet for Texas. If one is set up (e.g. texas.coldspark.com), follow `claude/early-vote-sites-publish.md`:
the standalone page is published on the artifact as `site/index.html` by step 5 above, and the device-bound task pulls it
from there.
