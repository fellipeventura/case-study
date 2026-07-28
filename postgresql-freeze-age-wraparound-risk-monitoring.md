# Case Study: PostgreSQL Freeze Age and Transaction ID Wraparound Risk Monitoring

## 1. Overview

This case study describes the investigation and mitigation of PostgreSQL tables approaching transaction ID freeze risk.

The main objective was to reduce the risk of transaction ID wraparound by identifying tables with high XID age, evaluating whether emergency manual action was required, and defining a monitoring strategy after autovacuum tuning.

The analysis focused on:

- Transaction ID age
- Freeze age thresholds
- Autovacuum freeze behavior
- Emergency `VACUUM FREEZE`
- Post-adjustment monitoring
- Table-level preventive maintenance

## 2. Context

PostgreSQL uses transaction IDs to control row visibility.

Because transaction IDs are finite, old tuples must eventually be frozen. If freezing does not happen in time, the database can approach transaction ID wraparound risk.

Autovacuum normally handles freezing automatically, but in some environments large tables or insufficient autovacuum activity can cause relation XID age to grow too much.

This is not the kind of problem that politely asks for attention. It waits, grows silently, and then threatens the database like a villain with excellent patience.

## 3. Problem

Some tables were identified with high XID age, close to dangerous freeze thresholds.

The main concerns were:

- Tables approaching `autovacuum_freeze_max_age`
- Risk of forced aggressive vacuum
- Potential emergency maintenance outside planned windows
- Large tables requiring long freeze operations
- Need to determine whether manual `VACUUM FREEZE` was appropriate
- Need to validate whether new autovacuum settings would be enough

## 4. Investigation

The investigation started by identifying relations with high transaction ID age.

Useful query:

```sql
SELECT
    n.nspname AS schema_name,
    c.relname AS table_name,
    age(c.relfrozenxid) AS xid_age,
    current_setting('autovacuum_freeze_max_age')::bigint AS autovacuum_freeze_max_age,
    round(
        100.0 * age(c.relfrozenxid)
        / current_setting('autovacuum_freeze_max_age')::numeric,
        2
    ) AS freeze_age_pct,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size
FROM pg_class c
JOIN pg_namespace n
    ON n.oid = c.relnamespace
WHERE c.relkind IN ('r', 'p')
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(c.relfrozenxid) DESC;
```

The next step was to check table-level autovacuum activity:

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count,
    last_analyze,
    last_autoanalyze,
    analyze_count,
    autoanalyze_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

## 5. Root Cause

The primary risk was that some tables were accumulating XID age faster than autovacuum could reduce it.

Possible contributing factors included:

- Large table size
- High write activity
- Autovacuum not triggering early enough
- Vacuum operations not keeping up with workload
- Long-running transactions delaying cleanup
- Insufficient vacuum cost limits or worker availability
- Table-level settings not aligned with workload
- Freeze thresholds too high for the operational reality

## 6. Emergency Action: Manual VACUUM FREEZE

For tables already near dangerous thresholds, manual intervention may be required.

Example:

```sql
VACUUM (FREEZE, VERBOSE) schema.table_name;
```

Manual `VACUUM FREEZE` should be considered when:

- XID age is close to critical threshold
- Autovacuum is not catching up
- The table is large and needs planned execution
- There is risk of forced aggressive vacuum during business hours
- Monitoring shows continued growth after parameter tuning

However, manual freeze should be treated as emergency or corrective action, not as the main long-term strategy.

If manual freeze becomes routine, the autovacuum configuration is probably asking for help in a very PostgreSQL way.

## 7. Long-Term Solution

The long-term solution is to tune autovacuum so it can keep up with the workload before tables approach critical freeze age.

Important parameters include:

```sql
ALTER TABLE schema.table_name SET (
    autovacuum_vacuum_threshold = <value>,
    autovacuum_vacuum_scale_factor = <value>,
    autovacuum_analyze_threshold = <value>,
    autovacuum_analyze_scale_factor = <value>,
    autovacuum_freeze_min_age = <value>,
    autovacuum_freeze_table_age = <value>
);
```

The strategy should consider:

- Table size
- Churn rate
- Dead tuple generation
- Current XID age
- Time needed for vacuum
- Available I/O capacity
- Maintenance windows
- Business impact

## 8. Monitoring Strategy

Post-adjustment monitoring should focus on whether XID age stabilizes or continues growing.

Recommended daily monitoring:

```sql
SELECT
    n.nspname AS schema_name,
    c.relname AS table_name,
    age(c.relfrozenxid) AS xid_age,
    round(
        100.0 * age(c.relfrozenxid)
        / current_setting('autovacuum_freeze_max_age')::numeric,
        2
    ) AS freeze_age_pct,
    s.last_autovacuum,
    s.autovacuum_count,
    s.n_dead_tup,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size
FROM pg_class c
JOIN pg_namespace n
    ON n.oid = c.relnamespace
LEFT JOIN pg_stat_user_tables s
    ON s.relid = c.oid
WHERE c.relkind IN ('r', 'p')
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(c.relfrozenxid) DESC
LIMIT 50;
```

Recommended classification:

| Freeze Age Percentage | Classification | Action |
|---:|---|---|
| < 50% | Normal | Monitor |
| 50% to 70% | Attention | Review autovacuum behavior |
| 70% to 85% | Warning | Validate need for tuning or manual vacuum |
| 85% to 95% | Critical | Plan manual vacuum/freeze |
| > 95% | Emergency | Immediate action required |

## 9. Validation After Adjustment

After tuning, validate:

- Whether `xid_age` is decreasing or stabilizing
- Whether autovacuum is running on critical tables
- Whether dead tuples remain controlled
- Whether aggressive vacuums stop appearing unexpectedly
- Whether freeze age remains below warning thresholds
- Whether table-level parameters are appropriate

Example post-adjustment validation:

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_autovacuum,
    autovacuum_count,
    last_autoanalyze,
    autoanalyze_count
FROM pg_stat_user_tables
WHERE relname IN ('<critical_table_1>', '<critical_table_2>')
ORDER BY relname;
```

## 10. Expected Results

| Metric | Before | After |
|---|---|---|
| XID age | Increasing | Stabilized or reduced |
| Aggressive vacuum risk | High | Reduced |
| Manual freeze dependency | Possible | Reduced |
| Autovacuum activity | Insufficient or late | More consistent |
| Dead tuple control | Weak | Improved |
| Operational predictability | Low | Improved |

## 11. Lessons Learned

The main lessons were:

- Freeze age must be monitored proactively.
- Autovacuum tuning is not only about dead tuples.
- Large tables may need table-level freeze strategy.
- Manual `VACUUM FREEZE` is useful for emergency correction but should not replace proper autovacuum tuning.
- Monitoring must continue after parameter changes.
- A stable XID age trend is more important than a single isolated reading.
- Tables close to wraparound risk should be prioritized before routine optimization work.

## 12. Business Impact

Preventing transaction ID wraparound risk protects database availability.

Business benefits include:

- Lower risk of emergency downtime
- Better maintenance predictability
- Reduced probability of forced aggressive vacuum during business hours
- Improved operational control over large tables
- Safer PostgreSQL production environment
- Stronger disaster prevention posture

## 13. Conclusion

The freeze age investigation showed that PostgreSQL autovacuum tuning must include transaction ID wraparound prevention, not only dead tuple cleanup.

For tables already near critical thresholds, manual `VACUUM FREEZE` can be necessary. However, the long-term solution is to tune and monitor autovacuum so it keeps XID age under control automatically.

This case highlights the importance of proactive freeze age monitoring as part of PostgreSQL production operations.
