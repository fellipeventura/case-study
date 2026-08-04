# Case Study: PostgreSQL 14 to PostgreSQL 18 Upgrade Using pgBackRest Restore and pg_upgrade

## 1. Overview

This case study describes a successful PostgreSQL major version upgrade from PostgreSQL 14 to PostgreSQL 18.

The process included creating a restored PostgreSQL 14 environment from pgBackRest, validating database consistency, preparing a new PostgreSQL 18 cluster, running `pg_upgrade --check`, executing the upgrade successfully, refreshing optimizer statistics and validating access through the upgraded instance.

The main objective was to perform a controlled major version upgrade with validation steps before, during and after the process.

The analysis focused on:

- pgBackRest restore
- PostgreSQL 14 restored environment validation
- PostgreSQL 18 cluster initialization
- Encoding and locale validation
- Extension inventory
- Wraparound risk validation
- Statistics preparation
- `pg_upgrade --check`
- Successful `pg_upgrade` execution
- Post-upgrade statistics refresh
- Final instance startup and access validation

## 2. Context

A PostgreSQL 14 database needed to be upgraded to PostgreSQL 18.

To reduce risk, the process was not executed directly on the original production data directory. Instead, a new PostgreSQL 14 restored environment was created using pgBackRest.

After validating the restored source environment, a clean PostgreSQL 18 cluster was initialized and used as the target cluster for `pg_upgrade`.

This approach allowed the upgrade to be validated in a controlled way before switching the updated instance to the expected service port.

## 3. Initial Restore Preparation

A new data directory was created to store the restored PostgreSQL 14 instance.

Example:

```bash
mkdir ./14.directory_new
```

The pgBackRest package was installed on the target server. During installation, a dependency issue was identified.

Example dependency error:

```text
Failed dependencies:
    libssh2 is needed by pgbackrest
    libssh2.so.1()(64bit) is needed by pgbackrest
```

The missing dependency was installed, and pgBackRest installation was completed successfully.

Example:

```bash
dnf install -y libssh2
rpm -i pgbackrest-2.xx-1PGDG.rhel9.x86_64.rpm
```

## 4. pgBackRest Configuration

The pgBackRest configuration was adjusted to use the correct repository, backup host, user, spool path and process settings.

Generic configuration example:

```ini
[global]
process-max=4
repo1-path=/backup/repository
repo1-host=<backup_host>
repo1-host-user=postgres
log-level-console=info
log-level-file=info
archive-async=y
archive-push-queue-max=2000M
spool-path=/restore/spool
```

Backup metadata was validated using:

```bash
pgbackrest --log-level-console=detail --stanza=<stanza_name> info
```

The output confirmed that the stanza status was OK and that full backups were available.

## 5. PostgreSQL 14 Restore

A restore was executed from pgBackRest into the new PostgreSQL 14 data directory.

Generic example:

```bash
pgbackrest \
  --process-max=4 \
  --stanza=<stanza_name> \
  --log-level-console=detail \
  --pg1-path=<new_pg14_data_directory> \
  --delta \
  --set=<backup_set> \
  --type=immediate \
  --target-action=promote \
  restore
```

The restore completed successfully.

Observed result from the documented process:

```text
restore size = 51.0GB
file total = 3105
restore command end: completed successfully
real    2m17s
```

## 6. PostgreSQL 14 Restored Instance Validation

After the restore, the PostgreSQL 14 instance was started on a temporary port.

Example:

```bash
pg_ctl -D <new_pg14_data_directory> start
```

The logs confirmed that the database system was ready to accept connections.

The restored instance was accessed with `psql`, and the database list was validated.

Recovery state was checked:

```sql
SELECT pg_is_in_recovery();
```

Expected result:

```text
f
```

This confirmed that the restored instance was promoted and no longer in recovery.

## 7. Pre-Upgrade Validation

Several checks were performed before running `pg_upgrade`.

### PostgreSQL Version

```sql
SELECT version();
```

The restored source instance was confirmed as PostgreSQL 14.

### Extension Inventory

```sql
SELECT extname, extversion
FROM pg_extension
ORDER BY 1;
```

The documented environment only had `plpgsql` installed.

This reduced upgrade complexity because there were no third-party extensions requiring target-version libraries.

### Object Inventory

```sql
SELECT relkind, count(*)
FROM pg_class
GROUP BY relkind;
```

This helped validate the object count before the upgrade.

### Wraparound Risk

Transaction ID age was checked before the upgrade:

```sql
SELECT datname, age(datfrozenxid)
FROM pg_database;
```

The values were low and safe for the upgrade process, so `VACUUM FREEZE` was not required before the upgrade.

### Dead Tuple Review

Dead tuples were reviewed:

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

The documented result showed no relevant dead tuple accumulation.

### Tables Without Analyze

Tables without analyze information were listed:

```sql
SELECT
    schemaname,
    relname,
    last_analyze
FROM pg_stat_user_tables
WHERE last_analyze IS NULL;
```

The affected tables were analyzed manually before the upgrade.

Example:

```sql
ANALYZE schema.table_name;
```

### Encoding and Locale

Encoding and locale were validated:

```sql
SELECT
    datname,
    encoding,
    datcollate,
    datctype
FROM pg_database;
```

This was important because differences between source and target locale or collation settings can affect sorting behavior and index correctness.

## 8. PostgreSQL 18 Cluster Creation

A new PostgreSQL 18 data directory was created.

Example:

```bash
mkdir ./18.directory
```

The PostgreSQL 18 cluster was initialized with the required encoding and locale:

```bash
/usr/pgsql-18/bin/initdb \
  -D <new_pg18_data_directory> \
  --encoding=UTF8 \
  --locale=pt_BR.UTF-8 \
  --no-data-checksums
```

The initialization completed successfully.

The new cluster was configured to use a temporary port for validation.

Example:

```conf
port = 5490
```

The PostgreSQL 18 instance was started and accessed with `psql` to validate default objects and cluster initialization.

## 9. pg_upgrade Compatibility Check

Both clusters were stopped before running `pg_upgrade`.

Example:

```bash
/usr/pgsql-14/bin/pg_ctl -D <pg14_data_directory> stop
/usr/pgsql-18/bin/pg_ctl -D <pg18_data_directory> stop
```

The compatibility check was executed:

```bash
/usr/pgsql-18/bin/pg_upgrade \
  -d <pg14_data_directory> \
  -D <pg18_data_directory> \
  -b /usr/pgsql-14/bin \
  -B /usr/pgsql-18/bin \
  -c
```

The check completed successfully.

Key result:

```text
Checking cluster versions                                     ok
Checking database connection settings                         ok
Checking database user is the install user                    ok
Checking for prepared transactions                            ok
Checking data type usage                                      ok
Checking for not-null constraint inconsistencies              ok
Checking for presence of required libraries                   ok
Checking for new cluster tablespace directories               ok
*Clusters are compatible*
```

## 10. pg_upgrade Execution

After the compatibility check succeeded, the upgrade was executed without the `-c` option.

Example:

```bash
/usr/pgsql-18/bin/pg_upgrade \
  -d <pg14_data_directory> \
  -D <pg18_data_directory> \
  -b /usr/pgsql-14/bin \
  -B /usr/pgsql-18/bin
```

The upgrade completed successfully.

Important stages completed by `pg_upgrade` included:

- Consistency checks
- Global object dump
- Database schema dump
- Locale and encoding setup for the new cluster
- Row analysis in the new cluster
- Freezing rows in the new cluster
- Transaction ID and multixact metadata migration
- WAL archive reset
- Global object restore
- Database schema restore
- User relation file copy
- Data directory sync
- Extension update check

Final result:

```text
Upgrade Complete
```

## 11. Post-Upgrade Statistics Refresh

After the upgrade, PostgreSQL recommended refreshing optimizer statistics because some statistics are not transferred by `pg_upgrade`.

The upgraded PostgreSQL 18 instance was started.

Then the recommended commands were executed:

```bash
/usr/pgsql-18/bin/vacuumdb -p <port> --all --analyze-in-stages --missing-stats-only
```

And:

```bash
/usr/pgsql-18/bin/vacuumdb -p <port> --all --analyze-only
```

This ensured that optimizer statistics were refreshed after the major version upgrade.

## 12. Configuration Migration and Final Startup

After the post-upgrade analyze process, the PostgreSQL 18 instance was stopped.

Relevant configuration files from the previous PostgreSQL 14 environment were copied to the PostgreSQL 18 data directory.

Examples:

```bash
cp <old_pg14_data_directory>/pg_hba.conf <new_pg18_data_directory>/
cp <old_pg14_data_directory>/postgresql.auto.conf <new_pg18_data_directory>/
```

The cluster name and port were adjusted to represent the upgraded PostgreSQL 18 instance.

The old PostgreSQL 14 instance was stopped, and the PostgreSQL 18 instance was started using the expected service port.

Access was validated using:

```bash
/usr/pgsql-18/bin/psql -p <service_port>
```

Database listing confirmed that the upgraded database was available on PostgreSQL 18.

## 13. Results

| Step | Result |
|---|---|
| pgBackRest installation | Completed after dependency correction |
| Backup availability check | Successful |
| PostgreSQL 14 restore | Successful |
| Restored instance promotion | Confirmed with `pg_is_in_recovery() = false` |
| Extension inventory | Only `plpgsql` identified |
| Wraparound risk | Safe |
| Manual pre-upgrade analyze | Executed |
| PostgreSQL 18 initdb | Successful |
| `pg_upgrade --check` | Successful |
| `pg_upgrade` execution | Successful |
| Post-upgrade analyze | Executed |
| Final PostgreSQL 18 startup | Successful |
| Access validation | Successful |

## 14. Root Cause and Risk Prevention

There was no upgrade failure caused by extension libraries in this documented process.

A dependency issue occurred during pgBackRest installation due to the missing `libssh2` package. This was resolved before the restore.

The major upgrade itself succeeded because the validation process confirmed:

- Compatible clusters
- Required libraries present
- Safe transaction ID age
- No problematic third-party extensions
- Valid source and target cluster configuration
- Matching encoding and locale planning
- Clean `pg_upgrade --check` result

The key success factor was not luck. It was validation. A shocking concept, apparently still optional in some places.

## 15. Lessons Learned

The main lessons from this upgrade were:

- Always validate backup availability before preparing an upgrade.
- Restoring to a separate PostgreSQL 14 data directory reduces risk.
- Validate that the restored instance is promoted before upgrade preparation.
- Check extensions before `pg_upgrade`.
- Validate wraparound risk before major upgrade.
- Analyze tables before upgrade when statistics are missing.
- Match encoding and locale intentionally.
- Always run `pg_upgrade --check` before the real upgrade.
- Run the recommended `vacuumdb` analyze commands after upgrade.
- Do not delete the old cluster until the upgraded instance is fully validated.
- Capture all steps and commands for auditability and repeatability.

## 16. Business Impact

The process reduced operational risk during the PostgreSQL major version upgrade.

Business benefits included:

- Controlled upgrade execution
- Reduced downtime risk
- Validated restore path
- Safer migration to PostgreSQL 18
- Improved confidence in rollback readiness
- Better post-upgrade optimizer statistics
- Clear evidence of successful validation

## 17. Conclusion

The PostgreSQL 14 to PostgreSQL 18 upgrade was completed successfully using a controlled process based on pgBackRest restore, source validation, target cluster preparation, `pg_upgrade --check`, full `pg_upgrade` execution and post-upgrade statistics refresh.

The process demonstrated a safe and repeatable approach for PostgreSQL major version upgrades, with validation at each critical step.

This case highlights that successful PostgreSQL upgrades depend on preparation, compatibility checks, backup validation and post-upgrade verification, not on heroic improvisation during the maintenance window.