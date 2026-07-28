# Case Study: PostgreSQL Autovacuum and Autoanalyze Tuning Across Production Databases

## 1. Overview

This case study describes a PostgreSQL autovacuum and autoanalyze tuning initiative performed across multiple production databases.

The main objective was to improve the behavior of automatic maintenance routines by reducing excessive or heavy autovacuum executions, improving statistics freshness and increasing the predictability of database maintenance activity.

The analysis compared pre-adjustment and post-adjustment log data from different PostgreSQL environments, focusing on:

- Autovacuum execution count
- Autoanalyze execution count
- Total maintenance time
- Average execution time
- Table-level behavior
- Critical table improvements
- Post-adjustment stability

The evaluated environments included transactional, Database_Production1 and document-management workloads.

## 2. Context

PostgreSQL autovacuum is responsible for cleaning dead tuples, preventing table and index bloat, maintaining visibility maps and protecting the database from transaction ID wraparound.

Autoanalyze is responsible for refreshing table statistics used by the query planner.

In large or high-churn databases, default autovacuum parameters may not be adequate. Large tables can accumulate too many dead tuples before cleanup, while volatile tables may suffer from stale statistics if autoanalyze does not run frequently enough.

The tuning effort focused on adjusting autovacuum behavior according to table size, workload pattern and operational impact.

## 3. Problem

Before the tuning effort, the analyzed environments showed symptoms such as:

- Excessive automatic maintenance activity in some databases
- Heavy autovacuum executions on critical tables
- Repeated maintenance on high-churn tables
- Delayed cleanup behavior in large transactional tables
- Risk of stale statistics affecting query plans
- Need for better visibility over table-level maintenance behavior

The goal was not simply to reduce the number of executions, but to make automatic maintenance more aligned with each workload.

In PostgreSQL, fewer executions are not always better. A low number of autovacuum runs can mean either a healthy database or a maintenance process that is arriving too late, because apparently databases also enjoy being misunderstood.

## 4. Methodology

The analysis was based on PostgreSQL log data collected before and after autovacuum parameter adjustments.

The following dimensions were compared:

- Total number of autovacuum and autoanalyze executions
- Total execution time in the analyzed window
- Average execution time
- Median execution time
- Largest observed execution
- Daily execution behavior
- Time accumulated by table
- Critical table behavior before and after tuning

The comparison was performed using equivalent time windows whenever possible.

## 5. Environment 1: Transactional Workload

### Database Profile

The first environment was a transactional PostgreSQL database with tables related to individual and corporate transactional data.

The comparison covered the period from April 9 to April 18, 2026, against June 9 to June 18, 2026.

### Results Summary

| Metric                    | Before    | After     | Variation |
|---------------------------|-----------|-----------|-----------|
| Total executions          | 90        | 109       | +21.1%    |
| Total maintenance time    | 16.56 min | 22.89 min | +38.2%    |
| Average execution time    | 11.04 s   | 12.60 s   | +14.1%    |
| Total autovacuum time     | 11.84 min | 9.15 min  | -22.7%    |
| Average autovacuum time   | 17.33 s   | 11.94 s   | -31.1%    |
| Total autoanalyze time    | 4.72 min  | 13.74 min | +191.1%   |
| Average autoanalyze time  | 5.78 s    | 13.08 s   | +126.4%   |

### Interpretation

The total number of automatic maintenance events increased after the adjustment. However, the most relevant improvement was seen in autovacuum behavior.

Autovacuum total time dropped from 11.84 minutes to 9.15 minutes, while average autovacuum duration dropped from 17.33 seconds to 11.94 seconds.

This indicates that cleanup became lighter and more efficient on critical tables.

The increase in autoanalyze time was expected, because the tuning made statistics refresh more sensitive to data changes. In practical terms, the database started updating statistics more often, improving the query planner’s ability to choose adequate execution plans.

### Table-Level Improvements

| Table                     | Before Time | After Time | Variation |
|---------------------------|-------------|------------|-----------|
| person_table         | 6.09 min    | 3.01 min   | -50.5%    |
| enterprise_table     | 2.02 min    | 0.94 min   | -53.3%    |
| classification_table         | 4.63 min    | 6.00 min   | +29.7%    |
| classification_alternative_table | 0.69 min    | 0.99 min   | +43.4%    |
| change_table          | 3.13 min    | 5.04 min   | +61.1%    |

The strongest operational gains were observed in the transactional tables, where accumulated maintenance time dropped by more than 50%.

## 6. Environment 2: Database_Production1 Workload

### Database Profile

The second environment was a Database_Production1 PostgreSQL database with high-volume application tables.

The comparison covered April 1 to April 15, 2026, against June 1 to June 15, 2026.

### Results Summary

| Metric                        | Before        | After         | Variation |
|-------------------------------|---------------|---------------|-----------|
| Total executions              | 10,262        | 3,377         | -67.1%    |
| Total maintenance time        | 1,661.00 min  | 832.01 min    | -49.9%    |
| Average execution time        | 9.71 s        | 14.78 s       | +52.2%    |
| Largest observed execution    | 93.66 s       | 128.63 s      | +37.3%    |
| Autovacuum executions         | 5,133         | 1,584         | -69.1%    |
| Total autovacuum time         | 970.70 min    | 651.81 min    | -32.9%    |
| Autoanalyze executions        | 5,129         | 1,793         | -65.0%    |
| Total autoanalyze time        | 690.30 min    | 180.20 min    | -73.9%    |

### Interpretation

This environment showed the strongest overall reduction.

Total automatic maintenance events dropped by 67.1%, from 10,262 to 3,377 executions. Total maintenance time dropped by 49.9%, from 1,661.00 minutes to 832.01 minutes.

Both autovacuum and autoanalyze improved:

- Autovacuum executions dropped by 69.1%
- Autovacuum total time dropped by 32.9%
- Autoanalyze executions dropped by 65.0%
- Autoanalyze total time dropped by 73.9%

Although the average autovacuum execution time increased, the total number of executions and total accumulated time decreased significantly. This suggests that the database stopped performing excessive repetitive maintenance and moved toward a more controlled maintenance pattern.

### Critical Tables

| Table                 | Executions Before | Time Before   | Executions After  | Time After  | Time Variation |
|-----------------------|-------------------|---------------|-------------------|-------------|----------------|
| account_table         | 4,192             | 985.01 min    | 986               | 477.61 min  | -51.5%         |
| person_table          | 1,574             | 342.99 min    | 441               | 171.58 min  | -50.0%         |
| product_table         | 657               | 176.82 min    | 182               | 102.03 min  | -42.3%         |
| register_table        | 760               | 47.67 min     | 430               | 26.92 min   | -43.5%         |
| collateral_table      | 628               | 42.02 min     | 204               | 23.55 min   | -44.0%         |

The largest absolute gain occurred on the `account_table` table, which reduced accumulated maintenance time from 985.01 minutes to 477.61 minutes.

The `product_table` table also improved, reducing accumulated time by 42.3%. During the analysis, an unused index was removed, reducing write and maintenance overhead.

### Key Outcome

The Database_Production1 environment became significantly more efficient from an automatic maintenance perspective.

The tuning reduced excessive recurrence while keeping maintenance active on high-volume tables.

## 7. Environment 3: Document Management Workload

### Database Profile

The third environment was a document-management PostgreSQL database.

The comparison evaluated the baseline period before June 18, 2026, against the stabilized post-adjustment period from June 20 to June 22, 2026.

The adjustment day and transition day were treated separately.

### Results Summary

| Metric                    | Before    | Stabilized After  | Variation | Interpretation            |
|---------------------------|-----------|-------------------|-----------|---------------------------|
| Automatic events per day  | 25.5      | 110.3             | +332.7%   | More preventive activity  |
| Vacuums per day           | 10.0      | 57.3              | +473.3%   | More automatic cleanup    |
| Analyzes per day          | 15.5      | 53.0              | +241.9%   | Fresher statistics        |
| Average autoanalyze time  | 24.80 s   | 10.95 s           | -55.9%    | Relevant reduction        |
| Largest average execution | 80.60 s   | 39.45 s           | -51.1%    | Lower peak concentration  |

### Interpretation

Unlike the Database_Production1 workload, this environment improved by increasing the frequency of automatic maintenance.

After the adjustment, the number of automatic events per day increased from 25.5 to 110.3. This was positive because maintenance started running earlier, before excessive accumulation occurred.

At the same time, average autoanalyze duration dropped from 24.80 seconds to 10.95 seconds, a reduction of 55.9%.

This indicates a healthier preventive maintenance pattern: more frequent executions, but lighter and less concentrated.

### Critical Tables

The main monitored tables were:

- document_identifier
- document_folder_identifier

These tables concentrated much of the automatic maintenance activity.

For `document_identifier`, post-adjustment maintenance remained frequent, but the largest autoanalyze executions in the stabilized period stayed below previous baseline peaks.

For `document_folder_identifier`, average autoanalyze time dropped sharply after stabilization, especially on June 20 and June 21.

## 8. Root Cause

The main root cause was that default or previous autovacuum settings were not fully aligned with table size, churn rate and workload behavior.

In some environments, maintenance was too frequent and repetitive.

In others, maintenance was not frequent enough, causing larger work per execution.

The same autovacuum strategy does not work equally well for every table. Large tables, high-churn tables and frequently analyzed tables require different thresholds and scale factors.

## 9. Solution

The solution involved reviewing and adjusting autovacuum behavior using table-level tuning.

The tuning strategy considered:

- Table size
- Estimated row count
- Dead tuple generation
- Update and delete volume
- Frequency of data changes
- Autovacuum execution history
- Autoanalyze behavior
- Risk of stale statistics
- Transaction ID freeze risk
- Operational impact during business hours

Typical PostgreSQL parameters considered in this type of tuning include:

Example:

ALTER TABLE schema.table_name SET (
    autovacuum_vacuum_threshold = <value>,
    autovacuum_vacuum_scale_factor = <value>,
    autovacuum_analyze_threshold = <value>,
    autovacuum_analyze_scale_factor = <value>
);

## 10. Results

The tuning produced different but positive outcomes depending on each workload.

Transactional Database
Autovacuum total time reduced by 22.7%
Average autovacuum time reduced by 31.1%
Transactional tables reduced accumulated maintenance time by more than 50%
Autoanalyze became more active, improving statistics freshness
Database_Production1 Database
Total maintenance executions reduced by 67.1%
Total maintenance time reduced by 49.9%
Autovacuum executions reduced by 69.1%
Autoanalyze total time reduced by 73.9%
Critical tables showed significant reductions in accumulated maintenance time
Document Management Database
Automatic events per day increased by 332.7%
Vacuums per day increased by 473.3%
Average autoanalyze time reduced by 55.9%
Largest average execution reduced by 51.1%
Maintenance became more preventive and less concentrated

## 11. Lessons Learned

The main lessons from this tuning initiative were:

Autovacuum tuning must be table-aware.
A reduction in execution count is not always the only success metric.
More executions can be positive when they prevent excessive accumulation.
Autoanalyze behavior must be evaluated together with autovacuum.
Large tables usually require custom thresholds and scale factors.
Unused indexes increase maintenance cost and should be reviewed carefully.
Log-based analysis is essential for validating real autovacuum behavior.
Before and after comparisons are necessary to prove improvement.

## 12. Business Impact

The tuning improved database maintenance efficiency and operational predictability.

The main business benefits were:

Lower maintenance overhead in critical databases
Reduced excessive automatic maintenance recurrence
Better statistics freshness for the query planner
Lower risk of stale execution plans
More predictable database behavior
Reduced operational risk on high-volume tables
Better visibility for future maintenance decisions

## 13. Conclusion

The autovacuum and autoanalyze tuning initiative improved maintenance behavior across multiple PostgreSQL production workloads.

The Database_Production1 workload showed a strong reduction in total executions and accumulated maintenance time.

The transactional workload showed lighter autovacuum behavior on critical tables, even with increased autoanalyze activity.

The document-management workload moved toward a more preventive model, with more frequent but lighter executions.

Overall, the results show that autovacuum tuning should not be treated as a global one-size-fits-all configuration. Effective tuning requires table-level analysis, workload understanding and continuous monitoring.

The final outcome was a more predictable, efficient and controlled PostgreSQL maintenance process.