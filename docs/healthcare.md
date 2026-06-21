---
icon: lucide/qr-code
---

# Cutting a Healthcare Analytics Pipeline's Runtime by 71% (and Its Compute Bill by Half)

**Role:** Analytics Engineer
**Stack:** dbt, Databricks (SQL Warehouses, Delta Lake), Python, Git/CI

---

## The Situation

A healthcare analytics team I supported ran a nightly dbt pipeline on Databricks that transformed claims, eligibility, and clinical encounter data into models used for population health reporting and payer reconciliation. The pipeline had grown organically over two years — new sources bolted on, new models added — and by the time I got involved, it was taking **3 hours 40 minutes** to run nightly, occasionally blowing past the 6 AM SLA when source data arrived late. The Databricks SQL Warehouse it ran on was costing the team roughly **$14,000/month**, and finance had started asking pointed questions.

My job wasn't just "make it faster" — it was to find out _why_ it was slow, fix the actual bottlenecks instead of throwing bigger compute at the problem, and leave behind a pipeline the team could reason about.

## Step 1: Instrument Before You Optimize

Before touching a single model, I needed real numbers, not guesses. I used `dbt run` with `--log-level debug` combined with Databricks' query history API to build a simple runtime breakdown per model:

```sql
-- Pulled from system.query.history, joined against dbt's run_results.json
SELECT
  query_text,
  total_duration_ms,
  read_bytes,
  spilled_local_bytes
FROM system.query.history
WHERE start_time >= current_date() - INTERVAL 7 DAYS
ORDER BY total_duration_ms DESC
LIMIT 25
```

This immediately reframed the problem. The team's assumption was "we have too much data." The data showed something different:

- **5 models** (out of 140+) accounted for **62% of total runtime**.
- Several of those models showed heavy **disk spill**, meaning shuffle operations were exceeding cluster memory.
- A handful of "incremental" models were quietly running as **full table scans** every night.

That last point was the headline finding.

## Step 2: The Root Cause — Incremental Models That Weren't

Digging into the slowest model, `fct_claims_adjudicated`, the `is_incremental()` macro was technically present, but the filter condition referenced a column that wasn't actually indexed or partitioned on, and a left join earlier in the CTE chain silently forced Spark to re-read the entire upstream table before the incremental filter could prune anything:

```sql
-- Before: filter applied too late to help
{{ config(materialized='incremental', unique_key='claim_id') }}

with claims as (
    select * from {{ ref('stg_claims') }}
),
joined as (
    select c.*, e.eligibility_status
    from claims c
    left join {{ ref('stg_eligibility') }} e on c.member_id = e.member_id
)
select * from joined
{% if is_incremental() %}
where claim_id not in (select claim_id from {{ this }})
{% endif %}
```

The `not in (select claim_id from {{ this }})` pattern forced a full scan of the target table on every run, and the join against eligibility (a slowly-changing, much larger table) happened _before_ any pruning. I rewrote it to filter at the source and use a watermark column instead:

```sql
-- After: filter pushed down, watermark-based incremental logic
{{ config(
    materialized='incremental',
    unique_key='claim_id',
    incremental_strategy='merge',
    partition_by=['claim_date_key'],
    cluster_by=['member_id']
) }}

with claims as (
    select * from {{ ref('stg_claims') }}
    {% if is_incremental() %}
    where claim_updated_at > (select max(claim_updated_at) from {{ this }})
    {% endif %}
),
joined as (
    select c.*, e.eligibility_status
    from claims c
    left join {{ ref('stg_eligibility_current') }} e
        on c.member_id = e.member_id
)
select * from joined
```

Two changes mattered most: filtering _before_ the join instead of after, and replacing `eligibility` (full history) with a `_current` snapshot model built once nightly, so the join itself touched far less data. Adding `cluster_by` on `member_id` (Databricks' Liquid Clustering) also meant Spark could skip files instead of scanning them.

## Step 3: Right-Sizing Compute Instead of Just Scaling It Up

The team's instinct when things were slow had been to bump the SQL Warehouse size. I found the opposite problem: the warehouse was sized for the worst 5 models, then idled most of the night running small transformations at the same (expensive) size. I split the workload:

- A **small Serverless SQL Warehouse** for the ~130 lightweight models, with auto-stop after 5 minutes idle.
- A **dedicated Medium warehouse with Photon enabled**, scoped via dbt tags, for the handful of genuinely heavy fact-table builds.

```yaml
# dbt_project.yml — routing models to the right warehouse via tags
models:
  healthcare_analytics:
    marts:
      facts:
        +tags: ["heavy_compute"]
      dims:
        +tags: ["standard"]
```

Orchestration (Databricks Workflows) then split the dbt run into two tasks using `dbt run --select tag:heavy_compute` and `tag:standard`, each pointed at its own SQL Warehouse, running in parallel instead of in series.

## The Results

| Metric                       | Before   | After          |
| ---------------------------- | -------- | -------------- |
| Nightly runtime              | 3h 40m   | 1h 04m (-71%)  |
| Monthly compute cost         | ~$14,000 | ~$6,800 (-51%) |
| Models with full-table scans | 7        | 0              |
| SLA misses (per month)       | 4–6      | 0              |

Beyond the numbers, the bigger win was diagnosability: the team now had a per-model runtime dashboard built on `system.query.history`, so the next time something regressed, it would take minutes to find, not days.

## What I'd Do Differently

If I were starting this project today, I'd build the runtime/cost instrumentation **first**, before any model existed, rather than retrofitting it onto two years of accumulated technical debt. Cheap to add early, expensive to reconstruct later — a lesson that's shown up in every pipeline I've touched since.
