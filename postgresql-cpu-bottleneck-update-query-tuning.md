# Case Study: PostgreSQL CPU Bottleneck Caused by Concurrent UPDATE and Sequential Scan

## 1. Overview

This case study describes the investigation of a PostgreSQL CPU bottleneck caused by a high-frequency `UPDATE` statement executed concurrently by multiple application sessions.

The issue was identified during a period of elevated CPU usage.

The analysis focused on:

- Active database sessions
- Repeated concurrent statements
- Execution plan behavior
- Sequential scan cost
- Missing or inadequate indexing
- Query concurrency impact
- Recommended tuning actions

## 2. Context

A PostgreSQL production environment experienced high CPU usage during specific moments of the day.

Database activity showed multiple concurrent sessions executing a similar `UPDATE` statement against the same table.

The query pattern filtered by account identifier and ownership type.

Generic example:

```sql
UPDATE application_schema.customer_holder
SET
    status = 'DELETED',
    last_load_date = localtimestamp
WHERE
    account_id = '<account_id>'
    AND ownership_type = <value>;
```

The table contained a large number of rows, and the execution plan showed that PostgreSQL was scanning the table instead of using an efficient index access path.

## 3. Problem

The main symptoms were:

- High CPU usage
- Multiple concurrent sessions executing the same update pattern
- Sequential scans on a frequently updated table
- Large number of rows filtered out
- Increased execution time under concurrency
- Risk of lock contention and workload amplification

The query itself was simple, which is usually where trouble hides. Simple queries without the right index are just expensive queries wearing casual clothes.

## 4. Investigation

The investigation used database activity monitoring and execution plan analysis.

### Active session analysis

The first step was to identify repeated active sessions executing the same statement.

Useful diagnostic query:

```sql
SELECT
    pid,
    usename,
    datname,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS runtime,
    query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY runtime DESC;
```

### Execution plan analysis

The execution plan showed a sequential scan.

Generic example:

```text
Update on customer_holder
  -> Seq Scan on customer_holder
       Filter: account_id = '<account_id>' AND ownership_type = <value>
       Rows Removed by Filter: <large_number_of_rows>
```

The key issue was that PostgreSQL had to scan many rows to find a very small target set.

### Concurrency analysis

The issue became more severe because the same pattern was executed simultaneously by several sessions.

A query that is inefficient once becomes a CPU problem when executed repeatedly and concurrently.

## 5. Root Cause

The root cause was an inefficient access path for the `UPDATE` statement.

The database was using a sequential scan because there was no suitable index for the filter pattern, or the existing index was not selective or aligned enough with the query predicates.

The problematic filter pattern was:

```sql
WHERE account_id = ?
  AND ownership_type = ?
```

Because this pattern was executed concurrently, the sequential scan multiplied CPU consumption.

## 6. Solution

The recommended solution was to create or validate a composite index aligned with the query predicate.

Example:

```sql
CREATE INDEX CONCURRENTLY idx_customer_holder_account_ownership
ON application_schema.customer_holder (account_id, ownership_type);
```

If the update frequently targets only active rows or a specific status, a partial index may be considered.

Example:

```sql
CREATE INDEX CONCURRENTLY idx_customer_holder_account_ownership_active
ON application_schema.customer_holder (account_id, ownership_type)
WHERE status <> 'DELETED';
```

The best option depends on data distribution and query patterns.

## 7. Validation Steps

Before creating the index, validate:

```sql
SELECT
    attname,
    n_distinct,
    correlation
FROM pg_stats
WHERE schemaname = 'application_schema'
  AND tablename = 'customer_holder'
  AND attname IN ('account_id', 'ownership_type', 'status');
```

Review existing indexes:

```sql
SELECT
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname = 'application_schema'
  AND tablename = 'customer_holder';
```

After creating the index, validate the execution plan:

```sql
EXPLAIN (ANALYZE, BUFFERS)
UPDATE application_schema.customer_holder
SET
    status = 'DELETED',
    last_load_date = localtimestamp
WHERE
    account_id = '<account_id>'
    AND ownership_type = <value>;
```

Expected improvement:

- Index Scan or Bitmap Index Scan instead of Seq Scan
- Fewer rows scanned
- Lower CPU usage
- Lower execution time
- Reduced impact under concurrency

## 8. Additional Recommendations

### Control concurrency

If the application executes many identical updates in parallel, review whether concurrency can be limited.

Database tuning helps, but letting eight sessions punch the same table at once is still a lifestyle choice.

### Batch processing

If the operation affects many rows, batch updates may reduce lock duration and resource spikes.

Example pattern:

```sql
UPDATE application_schema.customer_holder
SET
    status = 'DELETED',
    last_load_date = localtimestamp
WHERE ctid IN (
    SELECT ctid
    FROM application_schema.customer_holder
    WHERE account_id = '<account_id>'
      AND ownership_type = <value>
    LIMIT 1000
);
```

### Keep statistics fresh

If the table changes frequently, autoanalyze settings should be reviewed.

Outdated statistics can lead PostgreSQL to choose poor execution plans.

### Monitor after change

Post-change monitoring should include:

- CPU usage
- query duration
- rows scanned
- buffer reads
- index usage
- lock waits
- update frequency

## 9. Expected Results

| Metric | Before | After |
|---|---|---|
| Access method | Sequential Scan | Index Scan or Bitmap Index Scan |
| Rows scanned | High | Low |
| CPU usage | High under concurrency | Reduced |
| Query duration | Higher | Lower |
| Concurrency impact | Significant | Reduced |
| Plan stability | Weak | Improved |

## 10. Lessons Learned

The main lessons were:

- Simple `UPDATE` statements can become expensive without proper indexing.
- Sequential scans under concurrency can quickly become CPU bottlenecks.
- Index design must follow real query predicates.
- Execution plans must be validated with `EXPLAIN ANALYZE`.
- Application concurrency can amplify database inefficiencies.
- Statistics freshness is essential for plan quality.

## 11. Business Impact

The tuning approach reduces the risk of application slowness caused by database CPU saturation.

Expected business benefits:

- Faster update operations
- Lower CPU pressure
- Better application response time
- Reduced risk of peak-hour incidents
- More predictable database behavior
- Improved scalability for concurrent workloads

## 12. Conclusion

The investigation showed that the CPU bottleneck was caused by a repeated concurrent `UPDATE` statement using an inefficient sequential scan.

The recommended solution was to add a composite index aligned with the query predicate, validate the execution plan and monitor concurrency after implementation.

This case highlights the importance of combining session monitoring, execution plan analysis and index design when troubleshooting PostgreSQL performance issues.
