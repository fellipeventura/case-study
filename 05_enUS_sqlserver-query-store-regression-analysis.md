# Case Study: SQL Server Query Store Regression Analysis

## 1. Overview

This case study describes the use of SQL Server Query Store to identify, analyze and mitigate query performance regressions in a production environment.

The main objective was to compare query behavior over time, detect regressions in CPU usage, duration and logical reads, and support decisions such as index tuning, statistics review or plan forcing.

The analysis focused on:

- Query Store runtime statistics
- Regressed queries
- Execution plan changes
- CPU usage
- Duration
- Logical reads
- Plan forcing validation
- Recommendations for long-term monitoring

## 2. Context

SQL Server Query Store was enabled in a production database to provide historical visibility into query performance.

The environment required a structured way to answer questions such as:

- Which queries consumed the most CPU?
- Which queries became slower compared to a previous period?
- Did the execution plan change?
- Is a forced plan currently active?
- Did the forced plan fail?
- Should the issue be solved with an index, statistics update, query rewrite or plan forcing?

Without Query Store, this type of investigation often depends on what is currently in cache, which is basically asking SQL Server what it remembers after a stressful day. Query Store gives the DBA a much better historical view.

## 3. Problem

The production workload showed periods of increased resource usage and application slowness.

The main concerns were:

- Queries with increased CPU consumption
- Queries with longer average duration
- Queries with higher logical reads
- Plan changes causing inconsistent performance
- Possible parameter sensitivity
- Need to identify whether regressions were isolated or recurring
- Need to validate existing forced plans

The goal was not only to find the most expensive query, but to identify which queries got worse compared to their previous behavior.

## 4. Investigation

The investigation used Query Store catalog views, including:

- `sys.query_store_query_text`
- `sys.query_store_query`
- `sys.query_store_plan`
- `sys.query_store_runtime_stats`
- `sys.query_store_runtime_stats_interval`

The analysis was divided into three main checks.

## 5. Top CPU Queries

The first step was to identify the highest CPU-consuming queries in the analyzed period.

Example query structure:

```sql
SELECT TOP (50)
    DB_NAME() AS database_name,
    q.query_id,
    p.plan_id,
    OBJECT_NAME(q.object_id) AS object_name,
    SUM(rs.count_executions) AS execution_count,
    SUM(rs.avg_cpu_time * rs.count_executions) / 1000.0 AS total_cpu_ms,
    AVG(rs.avg_cpu_time) / 1000.0 AS avg_cpu_ms,
    SUM(rs.avg_duration * rs.count_executions) / 1000.0 AS total_duration_ms,
    AVG(rs.avg_duration) / 1000.0 AS avg_duration_ms,
    SUM(rs.avg_logical_io_reads * rs.count_executions) AS total_logical_reads,
    LEFT(qt.query_sql_text, 4000) AS query_text
FROM sys.query_store_query_text qt
JOIN sys.query_store_query q
    ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p
    ON p.query_id = q.query_id
JOIN sys.query_store_runtime_stats rs
    ON rs.plan_id = p.plan_id
JOIN sys.query_store_runtime_stats_interval rsi
    ON rsi.runtime_stats_interval_id = rs.runtime_stats_interval_id
WHERE rsi.start_time >= DATEADD(DAY, -7, SYSUTCDATETIME())
GROUP BY
    q.query_id,
    p.plan_id,
    q.object_id,
    qt.query_sql_text
ORDER BY total_cpu_ms DESC;
```

This helped identify which queries were responsible for the largest CPU consumption.

## 6. Regressed Queries

The second step was to compare current behavior against a previous baseline.

A regression can be identified when a query shows a significant increase in:

- Average duration
- CPU time
- Logical reads
- Execution count
- Wait time

Example analysis approach:

```sql
WITH query_stats AS (
    SELECT
        q.query_id,
        p.plan_id,
        CASE
            WHEN rsi.start_time >= DATEADD(DAY, -7, SYSUTCDATETIME())
                THEN 'current_period'
            WHEN rsi.start_time >= DATEADD(DAY, -14, SYSUTCDATETIME())
             AND rsi.start_time < DATEADD(DAY, -7, SYSUTCDATETIME())
                THEN 'baseline_period'
        END AS period_name,
        SUM(rs.count_executions) AS execution_count,
        AVG(rs.avg_duration) / 1000.0 AS avg_duration_ms,
        AVG(rs.avg_cpu_time) / 1000.0 AS avg_cpu_ms,
        AVG(rs.avg_logical_io_reads) AS avg_logical_reads
    FROM sys.query_store_query q
    JOIN sys.query_store_plan p
        ON p.query_id = q.query_id
    JOIN sys.query_store_runtime_stats rs
        ON rs.plan_id = p.plan_id
    JOIN sys.query_store_runtime_stats_interval rsi
        ON rsi.runtime_stats_interval_id = rs.runtime_stats_interval_id
    WHERE rsi.start_time >= DATEADD(DAY, -14, SYSUTCDATETIME())
    GROUP BY
        q.query_id,
        p.plan_id,
        CASE
            WHEN rsi.start_time >= DATEADD(DAY, -7, SYSUTCDATETIME())
                THEN 'current_period'
            WHEN rsi.start_time >= DATEADD(DAY, -14, SYSUTCDATETIME())
             AND rsi.start_time < DATEADD(DAY, -7, SYSUTCDATETIME())
                THEN 'baseline_period'
        END
)
SELECT
    current_data.query_id,
    current_data.plan_id,
    baseline_data.avg_duration_ms AS baseline_avg_duration_ms,
    current_data.avg_duration_ms AS current_avg_duration_ms,
    ((current_data.avg_duration_ms - baseline_data.avg_duration_ms)
        / NULLIF(baseline_data.avg_duration_ms, 0)) * 100 AS duration_regression_pct,
    baseline_data.avg_cpu_ms AS baseline_avg_cpu_ms,
    current_data.avg_cpu_ms AS current_avg_cpu_ms,
    baseline_data.avg_logical_reads AS baseline_avg_logical_reads,
    current_data.avg_logical_reads AS current_avg_logical_reads
FROM query_stats current_data
JOIN query_stats baseline_data
    ON baseline_data.query_id = current_data.query_id
   AND baseline_data.plan_id = current_data.plan_id
WHERE current_data.period_name = 'current_period'
  AND baseline_data.period_name = 'baseline_period'
ORDER BY duration_regression_pct DESC;
```

This approach helps prioritize queries that became worse, not only queries that are naturally expensive.

## 7. Forced Plan Validation

The third step was to validate forced plans.

Forced plans can stabilize performance, but they must be monitored because they can fail or become inappropriate after data distribution changes.

Example query:

```sql
SELECT
    q.query_id,
    p.plan_id,
    p.is_forced_plan,
    p.force_failure_count,
    p.last_force_failure_reason_desc,
    p.count_compiles,
    p.last_compile_start_time,
    LEFT(qt.query_sql_text, 4000) AS query_text
FROM sys.query_store_plan p
JOIN sys.query_store_query q
    ON q.query_id = p.query_id
JOIN sys.query_store_query_text qt
    ON qt.query_text_id = q.query_text_id
WHERE p.is_forced_plan = 1
ORDER BY p.force_failure_count DESC;
```

This validation helps answer:

- Which plans are forced?
- Are forced plans failing?
- When did they last compile?
- Is the forced plan still safe?

## 8. Root Cause Categories

Query regressions usually fall into one or more of these categories:

| Category | Description |
|---|---|
| Plan regression | SQL Server selected a worse execution plan |
| Stale statistics | Cardinality estimates became inaccurate |
| Parameter sensitivity | Different parameter values caused unstable plans |
| Index change | Missing, changed or dropped indexes affected plan quality |
| Data growth | Table growth changed query cost |
| Workload shift | Query volume or access pattern changed |
| Blocking or waits | Runtime was affected by concurrency, not only plan quality |

## 9. Solution

The recommended solution depends on the regression type.

### Statistics update

When estimates are inaccurate:

```sql
UPDATE STATISTICS schema.table_name;
```

### Index tuning

When logical reads are high or scans are excessive, evaluate index changes.

Example:

```sql
CREATE INDEX ix_table_filter_columns
ON schema.table_name (filter_column_1, filter_column_2)
INCLUDE (select_column_1, select_column_2);
```

### Query rewrite

When the query structure leads to poor plans, review joins, predicates, functions in WHERE clauses, implicit conversions and unnecessary columns.

### Plan forcing

When a known good plan exists and the workload is stable, plan forcing may be considered.

Example:

```sql
EXEC sp_query_store_force_plan
    @query_id = <query_id>,
    @plan_id = <plan_id>;
```

Plan forcing should be monitored after implementation.

## 10. Validation After Fix

After applying a fix, validate:

- CPU usage
- Duration
- Logical reads
- Execution count
- Wait stats
- Plan stability
- Query Store runtime stats
- Application behavior

Expected validation query:

```sql
SELECT
    q.query_id,
    p.plan_id,
    SUM(rs.count_executions) AS execution_count,
    AVG(rs.avg_duration) / 1000.0 AS avg_duration_ms,
    AVG(rs.avg_cpu_time) / 1000.0 AS avg_cpu_ms,
    AVG(rs.avg_logical_io_reads) AS avg_logical_reads
FROM sys.query_store_query q
JOIN sys.query_store_plan p
    ON p.query_id = q.query_id
JOIN sys.query_store_runtime_stats rs
    ON rs.plan_id = p.plan_id
JOIN sys.query_store_runtime_stats_interval rsi
    ON rsi.runtime_stats_interval_id = rs.runtime_stats_interval_id
WHERE q.query_id = <query_id>
  AND rsi.start_time >= DATEADD(DAY, -1, SYSUTCDATETIME())
GROUP BY
    q.query_id,
    p.plan_id
ORDER BY avg_duration_ms DESC;
```

## 11. Expected Results

| Metric | Before | After |
|---|---|---|
| Average duration | Increased | Reduced or stabilized |
| CPU usage | High | Reduced |
| Logical reads | High | Reduced |
| Plan stability | Unstable | Improved |
| Regression visibility | Low | Improved |
| Troubleshooting time | Higher | Lower |

## 12. Lessons Learned

The main lessons were:

- Query Store is essential for historical performance analysis.
- Top-consuming queries are not always the same as regressed queries.
- Forced plans must be monitored continuously.
- Statistics and parameter sensitivity are common causes of plan instability.
- Query tuning decisions should be based on before/after evidence.
- A good performance analysis must separate plan problems from workload or blocking problems.

## 13. Business Impact

Using Query Store improved the ability to diagnose and prioritize performance problems.

Business benefits include:

- Faster root cause analysis
- Reduced application slowness
- Better prioritization of tuning actions
- Lower risk when using forced plans
- Improved production stability
- Better communication between DBA and application teams

## 14. Conclusion

The Query Store regression analysis provided a structured way to identify queries that became slower or more expensive over time.

By combining top resource consumption, regression detection and forced plan validation, the DBA team gained a more reliable method for SQL Server performance troubleshooting.

This approach supports safer tuning decisions and more predictable production behavior.
