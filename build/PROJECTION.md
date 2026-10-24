# Projected final electorate (gated panel) — Query M and make_proj.py

Added 2026-10-24. The dashboard's "Projected final electorate" section is hidden behind an acknowledgment checkbox
("I acknowledge modeling off of EV is insane but I want to look anyways"). It is filled from `data/proj.json`.
If `proj.json` is missing, or its as-of day differs from the page's, `build.py` hides the section and warns.

## Model (stress-tested in chat 2026-10-24)
- Cells: ModeledParty (5) × early/Election-Day habit in the last two generals × generals voted of the last three ×
  primary party (most recent). Party is **ModeledParty**, not PartyRollUp, so it is not the page's three-way split.
- For each cell, the chance that a voter who has not voted by day ASOF still votes comes from the training year,
  shrunk toward coarser cells (500-voter prior), and is scaled by (2026 early rate ÷ training-year early rate)^alpha.
- Central: train on 2022 (labels from the CURRENT `voters` file, registered ≤ 2022), alpha 0.5.
  Range: min/max over train {2022, 2024} × alpha {0.25, 0.5, 0.75}. Total turnout is deliberately not shown.
- Backtest (TX, 12 days out): model missed the final GOP−Dem gap by 0.4 (2022→2024) and 1.0 (2024→2022);
  the raw early vote missed by 6.9 and 4.6.
- Why current-file labels for 2022: ModeledParty labels drift between file versions (2022 TX electorate is 19.6%
  Swing under the 2022 file's scores, 14.4% under current scores). Training labels must match the 2026 labels.

## Query M (three queries; substitute ASOF = the page's as-of day from build.py, e.g. 12 on 2026-10-22)
Results are ~180 rows each; they are saved to a file automatically when large. Feed the three saved files to
`make_proj.py`. Use the same ASOF in all three.

2022 (current voters file, registered by 2022):
```sql
WITH a AS (
  SELECT DT_Regid, MAX(DATE_DIFF(DATE '2022-11-08', LEAST(IFNULL(EarlyVoted, DATE '2099-01-01'), IFNULL(IF(Rejected=0, ReturnDate, NULL), DATE '2099-01-01')), DAY)) dbe
  FROM `coldspark-voterdata.national_file.abev_2022`
  WHERE State='TX' AND (EarlyVoted IS NOT NULL OR (ReturnDate IS NOT NULL AND Rejected=0)) AND DT_Regid<>''
  GROUP BY 1 HAVING dbe BETWEEN 0 AND 60
), v AS (
  SELECT f.State st, f.ModeledParty party,
   CASE WHEN VH20G IN ('2','7') THEN 'E' WHEN VH20G='1' THEN 'D' WHEN VH18G IN ('2','7') THEN 'E' WHEN VH18G='1' THEN 'D' ELSE 'N' END habit,
   IF(VH20G IS NOT NULL,1,0)+IF(VH18G IS NOT NULL,1,0)+IF(VH16G IS NOT NULL,1,0) freq,
   CASE WHEN COALESCE(VH22P,VH20P) IN ('3','4','8') THEN 'R' WHEN COALESCE(VH22P,VH20P) IN ('5','6','9') THEN 'D' ELSE 'N' END prim,
   VH22G IS NOT NULL OR a.dbe IS NOT NULL fin, a.dbe
  FROM `coldspark-voterdata.national_file.voters` f LEFT JOIN a USING (DT_Regid)
  WHERE f.State='TX' AND (f.RegistrationYear IS NULL OR f.RegistrationYear<=2022))
SELECT st,party,habit,freq,prim, COUNT(*) n, COUNTIF(fin) fin, COUNTIF(dbe>=ASOF) eASOF, REPEAT('x',300) pad
FROM v GROUP BY 1,2,3,4,5
```
2024 (voterfile_2024 + abev_2024, Election Day 2024-11-05): same shape, with history `VH22G`/`VH20G`/`VH18G`,
primary `COALESCE(VH24P,VH22P)`, final `VH24G IS NOT NULL OR a.dbe IS NOT NULL`, no RegistrationYear filter.

2026 (voters + abev_2026, Election Day 2026-11-03): history `VH24G`/`VH22G`/`VH20G`, primary
`COALESCE(VH26P,VH24P)`, `0 fin`, no RegistrationYear filter, and `HAVING dbe BETWEEN ASOF AND 60` in `a`.

## Run
```
python3 make_proj.py data/proj.json TX ASOF <saved 2022 result> <saved 2024 result> <saved 2026 result>
python3 build.py
```
`make_proj.py` prints e.g. `TX as of d12: early +0.5 -> projected +5.4 (+0.9..+7.9)`. On 2026-10-22 (d12):
HardGOP 33.3→27.0, SoftGOP 10.4→13.3, Swing 13.0→24.8, SoftDem 14.7→14.5, HardDem 28.5→20.4.

## Other states (not yet live)
Same model was backtested in GA, NC, AZ, MI, OH, PA, IA, KS. Outside Texas the early vote overstates Democrats and
errors are larger (model's GOP−Dem error 1.5–9.0 at 12 days out, vs 0.7 in TX). Recommended order: GA, NC, then MI
(after Ryan confirms MI in-person early votes, which look missing from ABEV), with OH, PA, IA, KS shown only with
the range visible or held to the final week. The template section is state-neutral (`__STATE__`) and hides itself
when no projection is supplied, so copying `template.html` into another kit is safe.
