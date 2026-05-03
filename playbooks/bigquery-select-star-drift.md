---
name: BigQuery SELECT * Drift
description: Gradual accumulation of SELECT * queries against wide, unpartitioned, or large tables -- the single most common cause of runaway BigQuery on-demand costs.
scope: gcp
color: "#1A73E8"
emoji: 🔷
---

## Problem

BigQuery on-demand pricing charges per byte scanned (~$6.25/TB). `SELECT *`
reads every column in the table. On a 1,000-column event table where a query
only needs 10 columns, `SELECT *` scans 100x more data than necessary.

The problem compounds in three ways:

1. **Wide tables**: analytics tables commonly have 100-500+ columns. Projecting
   all columns when only 5 are needed can multiply cost by 10-100x per query.

2. **Missing partition filters**: `SELECT * FROM events WHERE user_id = '123'`
   on a partitioned-by-date table scans every partition. Add
   `AND DATE(event_timestamp) >= '2025-01-01'` and the scan drops to 1/365th.

3. **Dashboard proliferation**: Looker / Looker Studio dashboards run on a
   schedule. A dashboard with 20 tiles each running `SELECT *` against a
   100 GB table runs 20 full scans per refresh, every hour, every day.

A single `SELECT * FROM events` on a 5 TB table: $31.25.
That query running hourly from a dashboard: $750/day, $22,500/month.

## Symptoms

- BigQuery on-demand bill growing faster than data volume
- `INFORMATION_SCHEMA.JOBS_BY_PROJECT` shows queries with
  `total_bytes_billed > 1 TB` from BI tools (Looker, Looker Studio,
  Tableau, Power BI service accounts)
- Same table appearing in the top-10 scanned tables month after month
- Tables with no `require_partition_filter` flag and no default partition
  column in query `WHERE` clauses
- `referenced_tables` in `JOBS_BY_PROJECT` showing 1-2 large tables
  responsible for > 50% of total bytes billed

## Detection

```sql
-- Top 10 queries by bytes billed (last 30 days)
SELECT
  user_email,
  SUBSTR(query, 1, 200) AS query_preview,
  ROUND(total_bytes_billed / POW(1024, 4), 3) AS tb_billed,
  ROUND(total_bytes_billed / POW(1024, 4) * 6.25, 2) AS est_cost_usd,
  creation_time
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  AND job_type = 'QUERY'
  AND state = 'DONE'
  AND error_result IS NULL
ORDER BY total_bytes_billed DESC
LIMIT 10;
```

```sql
-- Tables with the highest scan volume (candidates for partition + clustering)
SELECT
  ref.project_id,
  ref.dataset_id,
  ref.table_id,
  COUNT(*) AS query_count,
  ROUND(SUM(total_bytes_billed) / POW(1024, 4), 2) AS total_tb_billed,
  ROUND(SUM(total_bytes_billed) / POW(1024, 4) * 6.25, 2) AS total_cost_usd
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT,
     UNNEST(referenced_tables) AS ref
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  AND job_type = 'QUERY'
  AND state = 'DONE'
GROUP BY 1, 2, 3
ORDER BY total_tb_billed DESC
LIMIT 20;
```

```sql
-- Check if top tables have partition filter enforcement
SELECT
  table_schema,
  table_name,
  ROUND(active_logical_bytes / POW(1024, 3), 1) AS active_gb,
  require_partition_filter
FROM `region-us`.INFORMATION_SCHEMA.TABLES t
JOIN `region-us`.INFORMATION_SCHEMA.TABLE_STORAGE s USING (table_catalog, table_schema, table_name)
WHERE table_schema IN ('your_dataset_name')
  AND active_logical_bytes > 10 * POW(1024, 3)  -- tables > 10 GB
ORDER BY active_logical_bytes DESC;
```

## Fix

### 1. Enable `require_partition_filter` on large tables

```sql
ALTER TABLE my_dataset.events
SET OPTIONS (require_partition_filter = true);
```

This returns an error for any query that doesn't include a filter on the
partition column — forcing the query author to add a date range. It is
the single most impactful guardrail for on-demand costs.

### 2. Add dry-run cost estimates to internal tooling

```python
from google.cloud import bigquery

def estimate_query_cost(query: str) -> float:
    client = bigquery.Client()
    config = bigquery.QueryJobConfig(dry_run=True, use_query_cache=False)
    job = client.query(query, job_config=config)
    tb = job.total_bytes_processed / (1024 ** 4)
    return tb * 6.25

# Show before submitting
cost = estimate_query_cost("SELECT * FROM my_dataset.events")
print(f"Estimated cost: ${cost:.2f}")
```

Show this estimate in any query UI before execution. Cost visibility
at the moment of submission is the highest-leverage behavioral intervention.

### 3. Replace `SELECT *` with explicit column lists in dashboards

For Looker / Looker Studio, edit the LookML or data source query to select
only the columns used by the dashboard tiles. For Tableau / Power BI,
use custom SQL or a BigQuery authorized view that projects only the needed
columns.

```sql
-- Create a narrow view for dashboards (scans only 8 columns, not 500)
CREATE OR REPLACE VIEW my_dataset.events_dashboard_view AS
SELECT
  event_id,
  event_timestamp,
  user_id,
  event_type,
  session_id,
  platform,
  country,
  revenue_usd
FROM my_dataset.events;
```

Then authorize this view on the source dataset -- downstream queries
against the view pay only for the 8 projected columns.

### 4. Add BI Engine for repeat dashboard queries

For dashboards that re-run the same queries repeatedly, BI Engine caches
results in memory and eliminates the scan cost:

```bash
# Reserve 10 GB of BI Engine capacity for a project
gcloud alpha bq reservations bi-reservations update \
  --location=us-central1 \
  --size=10Gi
```

### 5. Enforce column-level access with authorized views

Use authorized views to physically limit what columns a user or service
account can access. If a service account can't see column X, it can't
accidentally scan it.

## Anti-patterns

- **Fixing `SELECT *` reactively after the bill arrives**: By then, the
  query has been running in a dashboard for months. Set `require_partition_filter`
  before the table grows.
- **Telling users to "just add columns"**: They won't. Change the default
  by creating narrow views and making `SELECT *` fail via partition filter
  enforcement.
- **BI Engine as the first fix**: BI Engine reduces repeat-scan cost; it
  doesn't fix the underlying query. Fix the query first.

## References

- [BigQuery INFORMATION_SCHEMA.JOBS](https://cloud.google.com/bigquery/docs/information-schema-jobs)
- [Partition filter requirements](https://cloud.google.com/bigquery/docs/querying-partitioned-tables#require-filter)
- [BigQuery pricing](https://cloud.google.com/bigquery/pricing)
- Agent: `data-platforms/bigquery-cost-optimizer.md`
