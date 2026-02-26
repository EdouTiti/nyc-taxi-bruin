# nyc-taxi-bruin

### Project overview
This hands-on tutorial guides you through building a complete NYC Taxi data pipeline from scratch using Bruin - a unified CLI tool for data ingestion, transformation, orchestration, and governance.

We will create a pipeline with 3 layers:
1. Ingestion (Python + seed): fetch monthly NYC TLC parquet files + load a small lookup CSV
2. Staging (SQL): clean + deduplicate + enrich (join lookup)
3. Reports (SQL): aggregate metrics + run quality checks
Bruin runs assets in dependency order and supports validation (dry-run style) before executing.

### Project sources
NYC Taxi dataset (01.2019-11.2025)

### tools used: 
- Bruin for Data platform & pipeline orchestration
- SQL for Querying and transformations
- Python for Data processing
- R for Analytics & visualization
- GitHub Codespaces for Cloud development environment

### Steps
   ````bash
   #Use GitHub Codespaces to do the project.

   #Create a new repository named as "nyc-taxi-bruin"

   #Install Bruin in Codespaces in terminal bash of docker and verify the version
    curl -LsSf https://getbruin.com/install/cli | sh
   # check the version
   bruin version

   #Initialize the project zoomcamp folder and go inside it
    bruin init zoomcamp my-taxi-pipeline
    cd my-taxi-pipeline
    ls -la````

   # Configure .bruin.yml (DuckDB local)
   
   # Ingesting Historical Data: To backfill historical data, use the --start-date and --end-date flags:

   # Development: ingest 1-3 months

   # run the pipeline sequentially (no parallelism)
   # bruin run my-taxi-pipeline/pipeline/pipeline.yml \
  --environment default \
  --full-refresh \
  --start-date 2022-01-01 \
  --end-date 2022-02-01 \
  --var 'taxi_types=["yellow"]' \
  --workers 1

  # The message I got:
  ````bash
  bruin run completed successfully in 15.647s

 ✓ Assets executed      4 succeeded
 ✓ Quality checks       6 succeeded ````

  # quick data sanity check on the report:
  ````bash
  bruin query --connection duckdb-default --query "
SELECT pickup_date, taxi_type, total_trips
FROM reports.trips_report
ORDER BY pickup_date, taxi_type
LIMIT 20
" 
  ````

  Running this command should return the first few rows of the report table, for example:

  ```text
  ┌──────────────────────┬───────────┬─────────────┐
  │ PICKUP_DATE          │ TAXI_TYPE │ TOTAL_TRIPS │
  ├──────────────────────┼───────────┼─────────────┤
  │ 2022-01-01T00:00:00Z │ yellow    │ 63433       │
  │ 2022-01-02T00:00:00Z │ yellow    │ 58417       │
  │ ...                  │ ...       │ ...         │
  │ 2022-01-20T00:00:00Z │ yellow    │ 90770       │
  └──────────────────────┴───────────┴─────────────┘
  ```

  ## Quality checks summary

  The pipeline run also executes quality checks, which look something like
  this when you run the full pipeline:

  ```text
   ✓ Assets executed      4 succeeded
   ✓ Quality checks       6 succeeded
  ```

  To see the individual check results without re-running assets, use
  `--only checks`:

  ```bash
  bruin run ./pipeline/pipeline.yml --start-date 2022-01-01 \
    --end-date 2022-02-01 --var 'taxi_types=["yellow"]' --only checks
  ```

  ```text
  ingestion.payment_lookup:payment_type_id:not_null        PASS
  ingestion.payment_lookup:payment_type_name:not_null      PASS
  staging.trips:vendor_id:not_null                        PASS
  staging.trips:pickup_datetime:not_null                  PASS
  staging.trips:trip_distance:non_negative                PASS
  reports.trips_report:total_trips:non_negative           PASS
  ```

  ## Quality check details & run logs
### Executed queries

Below are the actual commands run against the local DuckDB and their results.

```bash
# row counts for all tables in the three schemas
bruin query --connection duckdb-default --query "
SELECT table_schema, table_name, COUNT(*) as row_count
FROM information_schema.tables
WHERE table_schema IN ('ingestion','staging','reports')
GROUP BY table_schema, table_name;
"
```

```
┌──────────────┬─────────────────────┬───────────┐
│ TABLE_SCHEMA │ TABLE_NAME          │ ROW_COUNT │
├──────────────┼─────────────────────┼───────────┤
│ ingestion    │ trips               │ 8         │
│ ingestion    │ _dlt_loads          │ 8         │
│ ingestion    │ _dlt_pipeline_state │ 8         │
│ ingestion    │ _dlt_version        │ 8         │
│ staging      │ trips               │ 8         │
│ ingestion    │ payment_lookup      │ 8         │
│ reports      │ trips_report        │ 8         │
└──────────────┴─────────────────────┴───────────┘
```

```bash
# sample aggregation on reports (no payment_type column available here)
bruin query --connection duckdb-default --query "
SELECT taxi_type, SUM(total_trips) as trips
FROM reports.trips_report
GROUP BY taxi_type
ORDER BY trips DESC
LIMIT 10;
"
```

```
┌───────────┬───────────┐
│ TAXI_TYPE │ TRIPS     │
├───────────┼───────────┤
│ yellow    │ 1476503   │
└───────────┴───────────┘
```

```bash
# show schema for staging.trips
bruin query --connection duckdb-default --query "DESCRIBE staging.trips;"
```

```
┌───────────────────┬──────────────────────────┬──────┬───────┬─────────┬───────┐
│ COLUMN_NAME       │ COLUMN_TYPE              │ NULL │ KEY   │ DEFAULT │ EXTRA │
├───────────────────┼──────────────────────────┼──────┼───────┼─────────┼───────┤
│ vendor_id         │ BIGINT                   │ YES  │ <nil> │ <nil>   │ <nil> │
│ pickup_datetime   │ TIMESTAMP WITH TIME ZONE │ YES  │ <nil> │ <nil>   │ <nil> │
│ dropoff_datetime  │ TIMESTAMP WITH TIME ZONE │ YES  │ <nil> │ <nil>   │ <nil> │
│ passenger_count   │ DOUBLE                   │ YES  │ <nil> │ <nil>   │ <nil> │
│ trip_distance     │ DOUBLE                   │ YES  │ <nil> │ <nil>   │ <nil> │
│ fare_amount       │ DOUBLE                   │ YES  │ <nil> │ <nil>   │ <nil> │
│ payment_type_id   │ BIGINT                   │ YES  │ <nil> │ <nil>   │ <nil> │
│ payment_type_name │ VARCHAR                  │ YES  │ <nil> │ <nil>   │ <nil> │
│ taxi_type         │ VARCHAR                  │ YES  │ <nil> │ <nil>   │ <nil> │
│ extracted_at      │ TIMESTAMP WITH TIME ZONE │ YES  │ <nil> │ <nil>   │ <nil> │
│ staged_at         │ TIMESTAMP WITH TIME ZONE │ YES  │ <nil> │ <nil>   │ <nil> │
└───────────────────┴──────────────────────────┴──────┴───────┴─────────┴───────┘
```

```bash
# earliest and latest pickup_datetime in staging.trips
bruin query --connection duckdb-default --query "
SELECT MIN(pickup_datetime) AS min_dt, MAX(pickup_datetime) AS max_dt
FROM staging.trips
"
```

```
┌──────────────────────┬──────────────────────┐
│ MIN_DT               │ MAX_DT               │
├──────────────────────┼──────────────────────┤
│ 2022-01-01T00:00:08Z │ 2022-05-18T20:41:57Z │
└──────────────────────┴──────────────────────┘
```

```bash
# compare staging row count with report sum
bruin query --connection duckdb-default --query "
SELECT
  (SELECT COUNT(*) FROM staging.trips) AS staging_n,
  (SELECT SUM(total_trips) FROM reports.trips_report) AS report_sum
"
```

```
┌───────────┬──────────────┐
│ STAGING_N │ REPORT_SUM   │
├───────────┼──────────────┤
│ 2526166   │ 2.463679e+06 │
└───────────┴──────────────┘
```

## Three quick analyses

1) Data parity: staging vs reports

Query:

```bash
bruin query --connection duckdb-default --query "
SELECT staging_n, report_sum, staging_n - report_sum AS diff,
       ROUND((staging_n - report_sum) * 100.0 / report_sum, 2) AS pct_diff_vs_report
FROM (
  SELECT
    (SELECT COUNT(*) FROM staging.trips) AS staging_n,
    (SELECT SUM(total_trips) FROM reports.trips_report) AS report_sum
) t
"
```

Result:

```
┌───────────┬──────────────┬────────┬─────────────────┐
│ STAGING_N │ REPORT_SUM   │ DIFF   │ PCT_DIFF_VS_REP │
├───────────┼──────────────┼────────┼─────────────────┤
│ 2526166   │ 2463679      │ 62487  │ 2.53            │
└───────────┴──────────────┴────────┴─────────────────┘
```

Interpretation: staging contains ~2.5% more rows than the aggregated
report total. Common causes: deduplication removing raw duplicates,
date/window filters, or rows excluded from aggregation (NULL taxi types,
etc.). Next step: inspect a sample of rows excluded from reports to find
systematic causes.

2) Time coverage

Query:

```bash
bruin query --connection duckdb-default --query "
SELECT MIN(pickup_datetime) AS min_dt, MAX(pickup_datetime) AS max_dt
FROM staging.trips
"
```

Result:

```
┌──────────────────────┬──────────────────────┐
│ MIN_DT               │ MAX_DT               │
├──────────────────────┼──────────────────────┤
│ 2022-01-01T00:00:08Z │ 2022-05-18T20:41:57Z │
└──────────────────────┴──────────────────────┘
```

Interpretation: data currently covers early 2022 through mid‑May 2022. If
you expected later months (through 2025), re-run or backfill ingestion for
the missing months and check the `taxi_types` variable and source file
availability.

3) Quality-check posture and recommendations

Checks run:

```bash
bruin run ./pipeline/pipeline.yml --start-date 2022-01-01 \
  --end-date 2022-02-01 --var 'taxi_types=["yellow"]' --only checks
```

Results summary:

```
ingestion.payment_lookup:payment_type_id:not_null        PASS
ingestion.payment_lookup:payment_type_name:not_null      PASS
staging.trips:vendor_id:not_null                        PASS
staging.trips:pickup_datetime:not_null                  PASS
staging.trips:trip_distance:non_negative                PASS
reports.trips_report:total_trips:non_negative           PASS
```

checks to catch subtle issues:
- Composite uniqueness on `(vendor_id, pickup_datetime, dropoff_datetime, trip_distance, fare_amount)`.
- `pickup_datetime <= now()` to detect future timestamps.
- Cross-layer reconciliation: assert `ABS(SUM(reports.total_trips) - COUNT(staging.trips)) / COUNT(staging.trips) < 0.05` (or your chosen threshold).



