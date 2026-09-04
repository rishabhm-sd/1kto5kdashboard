# Weekly data refresh runbook — HITS Retention Dashboards

This runs every **Monday 1:00pm IST (07:30 UTC)** as a scheduled task bound to Rishabh's Mac
(Metabase is only reachable from there). It refreshes the embedded data in all three
dashboards and their GitHub-site mirrors. There is no live database connection inside the
published pages — every number is baked into the HTML at refresh time.

## Files touched every run

| Artifact (published, has its own URL) | GitHub-site mirror (same content, wrapped in a full `<html>` doc) |
|---|---|
| `/home/claude/artifact/hits_churn_retention.html` | `/home/claude/site/retention.html` |
| `/home/claude/artifact/hits_churn_timing.html` | `/home/claude/site/timing.html` |
| `/home/claude/artifact/retention_curve.html` | `/home/claude/site/retention-3k.html` |

After editing an artifact file, republish it with the Artifact tool (`action:"read"` on its
URL first, then publish) so the live card updates. After editing the mirror in `/home/claude/site/`,
if the user's GitHub repo path is known and `device_bash` is available, `git add/commit/push`
from that path; otherwise just leave the refreshed files and tell the user in the summary that
GitHub still needs a manual push (git auth there hasn't been wired into this task yet).

## Step 1 — pull the three source queries from Metabase

Use `mcp__remote-devices__Metabase__Unofficial___Community___execute` (requires the Mac to be
linked; if `get_device_info` fails or Metabase isn't in `localMcpServers`, stop and tell the
user this run needs their Mac online with the Claude desktop app open, then reschedule).

**Query A — HITS churn cohort (feeds `hits_churn_retention.html` HITS side, and
`hits_churn_timing.html` both the main chart and `COHORT_MATRIX.HITS`).**
Run as `card_id: 14636` (card "HITSChurn-GCView3"), or the SQL below against `database_id: 23`
if the card has drifted — the eligibility CTE below adds `SELECT DISTINCT` versus the saved
card, which matters if `csv_upload.hit_master_data` ever grows another duplicate row (it has
before, for one seller in Aug-26):

```sql
WITH eligible_sellers AS (
  SELECT DISTINCT seller_id, hit_year, hit_month
  FROM csv_upload.hit_master_data
  WHERE (team = 'HITS' OR hit2 = 1) AND good_seller IS NULL
),
weekly_spend_raw AS (
  SELECT g.seller_id,
    DATE_ADD(DATE_TRUNC(DATE(CAST(SUBSTR(CAST(g.year_week AS STRING),1,4) AS INT64),1,4), ISOWEEK),
      INTERVAL (CAST(SUBSTR(CAST(g.year_week AS STRING),5,2) AS INT64)-1)*7 DAY) AS week_start,
    SUM(COALESCE(g.marketing_spend_tax_,0)) AS weekly_spend
  FROM nushop.gc_view_3 g JOIN eligible_sellers e ON e.seller_id = g.seller_id
  GROUP BY 1,2
),
seller_calendar AS (
  SELECT e.seller_id, week_start
  FROM eligible_sellers e
  CROSS JOIN UNNEST(GENERATE_DATE_ARRAY(
    DATE_TRUNC(DATE(e.hit_year, e.hit_month, 1), ISOWEEK),
    DATE_SUB(DATE_TRUNC(CURRENT_DATE('Asia/Kolkata'), ISOWEEK), INTERVAL 7 DAY),
    INTERVAL 7 DAY)) AS week_start
),
seller_weeks AS (
  SELECT c.seller_id, c.week_start, COALESCE(w.weekly_spend,0) AS weekly_spend
  FROM seller_calendar c
  LEFT JOIN weekly_spend_raw w ON c.seller_id=w.seller_id AND c.week_start=w.week_start
),
flagged AS (
  SELECT seller_id, week_start, CASE WHEN weekly_spend>=1000 THEN 1 ELSE 0 END AS has_spend
  FROM seller_weeks
),
ranked AS (
  SELECT seller_id, week_start, has_spend,
    ROW_NUMBER() OVER (PARTITION BY seller_id, has_spend ORDER BY week_start) AS rn
  FROM flagged
),
islands AS (
  SELECT seller_id, week_start, has_spend, DATE_SUB(week_start, INTERVAL rn WEEK) AS grp
  FROM ranked
),
runs AS (
  SELECT seller_id, has_spend, grp, MIN(week_start) AS run_start, MAX(week_start) AS run_end, COUNT(*) AS run_length
  FROM islands GROUP BY seller_id, has_spend, grp
),
churn_events AS (
  SELECT seller_id, run_start AS zero_spend_start, run_end
  FROM runs WHERE has_spend=0 AND run_length>=3
),
churn_events_valid AS (
  SELECT ce.seller_id, ce.zero_spend_start, ce.run_end
  FROM churn_events ce JOIN eligible_sellers e ON e.seller_id=ce.seller_id
  WHERE ce.zero_spend_start >= DATE_TRUNC(DATE(e.hit_year, e.hit_month, 1), ISOWEEK)
    AND NOT EXISTS (SELECT 1 FROM flagged f WHERE f.seller_id=ce.seller_id AND f.week_start>ce.run_end AND f.has_spend=1)
),
latest_churn_per_seller AS (
  SELECT seller_id, zero_spend_start,
    ROW_NUMBER() OVER (PARTITION BY seller_id ORDER BY zero_spend_start DESC) AS rn
  FROM churn_events_valid
),
churn_with_cohort AS (
  SELECT e.seller_id, e.hit_year, e.hit_month, lc.zero_spend_start,
    CASE WHEN lc.zero_spend_start IS NOT NULL
      THEN (EXTRACT(YEAR FROM lc.zero_spend_start)*12 + EXTRACT(MONTH FROM lc.zero_spend_start))
           - (e.hit_year*12 + e.hit_month) END AS months_from_hit
  FROM eligible_sellers e
  LEFT JOIN latest_churn_per_seller lc ON lc.seller_id=e.seller_id AND lc.rn=1
)
SELECT FORMAT_DATE('%b-%y', DATE(hit_year, hit_month, 1)) AS hit_month_label, hit_year, hit_month,
  COUNT(DISTINCT seller_id) AS num_sellers,
  SUM(CASE WHEN months_from_hit=0 THEN 1 ELSE 0 END) AS m0,
  SUM(CASE WHEN months_from_hit=1 THEN 1 ELSE 0 END) AS m1,
  SUM(CASE WHEN months_from_hit=2 THEN 1 ELSE 0 END) AS m2,
  SUM(CASE WHEN months_from_hit=3 THEN 1 ELSE 0 END) AS m3,
  SUM(CASE WHEN months_from_hit=4 THEN 1 ELSE 0 END) AS m4,
  SUM(CASE WHEN months_from_hit=5 THEN 1 ELSE 0 END) AS m5,
  SUM(CASE WHEN months_from_hit=6 THEN 1 ELSE 0 END) AS m6
FROM churn_with_cohort GROUP BY hit_year, hit_month ORDER BY hit_year DESC, hit_month DESC LIMIT 8;
```

**Query B — Revenue churn cohort** (feeds the Revenue side of both tabs). Identical shape,
adapted from card 14577 with the same `SELECT DISTINCT` safety fix and with the `%`-formatting
stripped back to raw counts (so it returns the same m0-m6 raw-count shape as Query A, not
percentages):

Same as Query A above, except the `eligible_sellers` filter is:
```sql
WHERE team IS NULL AND hit2 IS NULL AND good_seller IS NULL
```
and the same raw `SUM(CASE WHEN months_from_hit = N THEN 1 ELSE 0 END)` columns (not the
`ROUND(100.0*...)` version card 14577 saves).

**Query C — 1-5K / 3K weekly retention** (feeds `retention_curve.html`). Verified against the
live database on 2026-09-04 to reproduce the currently-published numbers exactly, cell for
cell, across all 7 cohorts:

```sql
WITH eligible_sellers AS (
  SELECT DISTINCT seller_id, hit_year, hit_month
  FROM csv_upload.hit_master_data
  WHERE (team = 'HITS' OR hit2 = 1) AND good_seller IS NULL
),
weekly_spend_raw AS (
  SELECT g.seller_id,
    DATE_ADD(DATE_TRUNC(DATE(CAST(SUBSTR(CAST(g.year_week AS STRING),1,4) AS INT64),1,4), ISOWEEK),
      INTERVAL (CAST(SUBSTR(CAST(g.year_week AS STRING),5,2) AS INT64)-1)*7 DAY) AS week_start,
    SUM(COALESCE(g.marketing_spend_tax_,0)) AS weekly_spend
  FROM nushop.gc_view_3 g JOIN eligible_sellers e ON e.seller_id = g.seller_id
  GROUP BY 1,2
),
seller_calendar AS (
  SELECT e.seller_id, e.hit_year, e.hit_month, week_start,
    DATE_DIFF(week_start, DATE_TRUNC(DATE(e.hit_year, e.hit_month, 1), ISOWEEK), WEEK) AS rel_week
  FROM eligible_sellers e
  CROSS JOIN UNNEST(GENERATE_DATE_ARRAY(
    DATE_TRUNC(DATE(e.hit_year, e.hit_month, 1), ISOWEEK),
    DATE_SUB(DATE_TRUNC(CURRENT_DATE('Asia/Kolkata'), ISOWEEK), INTERVAL 7 DAY),
    INTERVAL 7 DAY)) AS week_start
),
seller_weeks AS (
  SELECT c.seller_id, c.hit_year, c.hit_month, c.rel_week, COALESCE(w.weekly_spend,0) AS weekly_spend
  FROM seller_calendar c
  LEFT JOIN weekly_spend_raw w ON c.seller_id=w.seller_id AND c.week_start=w.week_start
),
cohort_size AS (
  SELECT hit_year, hit_month, COUNT(DISTINCT seller_id) AS cohort_size FROM eligible_sellers GROUP BY 1,2
)
SELECT sw.hit_year, sw.hit_month, cs.cohort_size, sw.rel_week,
  COUNT(DISTINCT sw.seller_id) AS n_reporting,
  COUNTIF(sw.weekly_spend >= 3000) AS n_retained
FROM seller_weeks sw JOIN cohort_size cs USING (hit_year, hit_month)
GROUP BY 1,2,3,4 ORDER BY 1,2,4;
```
Run against `database_id: 23`, `row_limit: 500`.

## Step 2 — transform into each file's constants

All three tabs use the same **"reached"** idea: for a cohort whose HIT month is `(hit_year,
hit_month)`, `reached = (current_year*12+current_month) - (hit_year*12+hit_month)`. That many
whole calendar months (0-indexed m0..m(reached-1)) are "closed"; the month at index `reached`
itself is still open/in-progress this week and gets marked **projected**.

**`hits_churn_retention.html`** — `DATA` / `REV_DATA`: for each cohort, walk `m0..m(reached-1)`
from Query A/B, keep a running `n_churned_cum` (cumulative sum), and emit one entry per closed
month: `{"pct": round(100*(cohort_size-n_churned_cum)/cohort_size,1), "n_retained":
cohort_size-n_churned_cum, "n_churned_cum": n_churned_cum, "cohort_size": cohort_size}`, then
pad the array with `null` out to 7 entries total. `PROJECTED`/`REV_PROJECTED`: one entry per
cohort still short of 7 closed months, using month index `m = reached`, with the same
cumulative-sum formula carried one more step using the (still-partial) `m[reached]` value from
the same query row — this is provisional and will move next week, hence the † marker.
`ALL_META.ALL_HITS`/`ALL_REV` ("reached-only base" Overall line): `base[i]` / `retained[i]` at
month index i = the sum of `cohort_size` / `(cohort_size - n_churned_cum at month i)` across
just the cohorts whose `reached > i`; `cohortsReach[i]` = how many cohorts contribute at i.

**`hits_churn_timing.html`** — `COHORT_MATRIX.HITS`/`REV`: pass Query A/B's `m0..m6` straight
through as the `m` array per cohort (`label`, `total: num_sellers`, `m: [m0..m6]`, `reached`),
no transformation needed — this is exactly the shape already on the page. The main chart's
`Mt_share` = `m[t] / sum(m0..m6)` per cohort (per the fix already applied this session, sum the
`m` array rather than trust a separate `grand_total` column, since one seller's out-of-range
`months_from_hit` can make those disagree by 1).

**`retention_curve.html`** — `DATA`: for each cohort, one entry per `rel_week` from Query C:
`{"pct": round(100*n_retained/n_reporting,1), "n_retained": n_retained, "cohort_size":
n_reporting}` (Query C already only returns weeks that have started, so no reached/null
padding is needed here — this tab uses a point-in-time weekly test, not a cumulative-churn
one). `OVERALL_META` ("reached-only base"): `base[w]` = sum of `cohort_size` and `retained[w]`
= sum of `n_retained` across every cohort whose data extends to relative week `w`;
`cohortsReach[w]` = how many cohorts do.

## Step 3 — sanity-check before publishing

- Every cohort's `cohort_size` should only ever grow week over week (sellers don't leave
  `hit_master_data`); if one shrinks versus last week's published value, treat that as a data
  anomaly worth flagging to Rishabh in the summary rather than silently publishing.
- If Query A/B returns a hit month older or newer than the 7 currently on the page (a new HIT
  month starting, or one rolling past whatever window each card's `LIMIT` uses), that's a
  structural change — add the new cohort key (and its `MONTH_LABEL`/CSS `--sN` slot) rather
  than dropping data, and say so in the summary.
- `n_reporting` should equal `cohort_size` for every row in Query C (no seller should be
  missing a week once it's started) — if it doesn't, the duplicate-row-style bug is back;
  re-check the `SELECT DISTINCT` in `eligible_sellers`.

## Step 4 — publish and report

1. Read each artifact URL, apply the targeted edits above, republish via the Artifact tool.
2. Update the matching file in `/home/claude/site/` the same way.
3. If a local git repo path for the GitHub Pages site is known (ask once, then remember it for
   next time by updating this trigger's prompt), commit and push from there; otherwise leave
   the refreshed files as-is.
4. Send Rishabh a short summary: what changed (new week's numbers, any new cohort, any
   anomaly caught in Step 3), and whether GitHub got pushed or still needs it.

Artifact URLs:
- HITS Churn-Cohort Retention: https://claude.ai/code/artifact/caf974fa-c483-4223-8b3d-1709cf93c74d
- HITS Churn Timing Diagnostic: https://claude.ai/code/artifact/c02f7ce3-5110-4e48-8440-10b7d6baa250
- 1-5K 3K Retention: https://claude.ai/code/artifact/bf27f30d-52ac-4bfc-8a80-fc918d3ca86f
