# Case Study: PostgreSQL FDW Server Misrouting Troubleshooting

## 1. Overview

This case study describes the investigation of a PostgreSQL Foreign Data Wrapper issue where a foreign table was pointing to an incorrect remote server.

The main objective was to identify why queries against a foreign table were reaching the wrong destination, validate FDW metadata and correct the foreign server configuration safely.

The analysis focused on:

- `postgres_fdw`
- Foreign tables
- Foreign servers
- User mappings
- Connection options
- Host and port validation
- Password and `.pgpass` behavior
- Post-change validation

## 2. Context

A PostgreSQL environment used `postgres_fdw` to access tables from another PostgreSQL instance.

A foreign table was expected to retrieve data from one remote database, but investigation showed that the FDW server configuration was pointing to an incorrect host or service.

This type of issue can be subtle because the local foreign table name may look correct while the underlying foreign server points somewhere else. Very elegant, in the same way a trapdoor under a carpet is elegant.

## 3. Problem

Queries against the foreign table did not behave as expected.

The main symptoms were:

- Foreign table accessing the wrong remote server
- Unexpected connection target
- Authentication errors after server correction
- User mapping inconsistency
- Need to validate FDW configuration without exposing credentials
- Risk of querying stale or incorrect remote data

The key operational risk was data misrouting: a local object could be technically valid but logically connected to the wrong source.

## 4. Investigation

The investigation started by checking the local foreign table metadata.

### Identify the foreign table

```sql
SELECT
    n.nspname AS schema_name,
    c.relname AS foreign_table_name,
    ft.ftserver,
    s.srvname AS foreign_server_name
FROM pg_foreign_table ft
JOIN pg_class c
    ON c.oid = ft.ftrelid
JOIN pg_namespace n
    ON n.oid = c.relnamespace
JOIN pg_foreign_server s
    ON s.oid = ft.ftserver
WHERE n.nspname = '<schema_name>'
  AND c.relname = '<foreign_table_name>';
```

### Check foreign server options

```sql
SELECT
    s.srvname,
    s.srvtype,
    s.srvversion,
    fdw.fdwname,
    s.srvoptions
FROM pg_foreign_server s
JOIN pg_foreign_data_wrapper fdw
    ON fdw.oid = s.srvfdw
WHERE s.srvname = '<foreign_server_name>';
```

This query validates the remote host, port and database configured in the foreign server options.

Example expected options:

```text
host=<remote_host>, port=<remote_port>, dbname=<remote_database>
```

### Check user mappings

```sql
SELECT
    s.srvname,
    r.rolname AS local_user,
    um.umoptions
FROM pg_user_mapping um
JOIN pg_foreign_server s
    ON s.oid = um.umserver
JOIN pg_roles r
    ON r.oid = um.umuser
WHERE s.srvname = '<foreign_server_name>';
```

User mapping validates which remote user is used when a local role queries the foreign table.

## 5. Root Cause

The root cause was a mismatch between the expected remote target and the actual FDW server configuration.

The foreign table was associated with a foreign server whose connection options pointed to the wrong remote host or port.

After correcting the foreign server target, an authentication issue appeared because the user mapping or password configuration was not aligned with the corrected destination.

This revealed two separate problems:

1. The FDW server pointed to the wrong target.
2. The credentials for the correct target also needed validation.

## 6. Solution

The first step was correcting the foreign server options.

Example:

```sql
ALTER SERVER <foreign_server_name>
OPTIONS (
    SET host '<correct_host>',
    SET port '<correct_port>',
    SET dbname '<correct_database>'
);
```

If an option does not exist yet, use `ADD` instead of `SET`:

```sql
ALTER SERVER <foreign_server_name>
OPTIONS (
    ADD host '<correct_host>',
    ADD port '<correct_port>',
    ADD dbname '<correct_database>'
);
```

If the user mapping password needs to be corrected:

```sql
ALTER USER MAPPING FOR <local_user>
SERVER <foreign_server_name>
OPTIONS (
    SET user '<remote_user>',
    SET password '<remote_password>'
);
```

In environments where `.pgpass` is used, validate:

- File location
- File owner
- File permissions
- Host
- Port
- Database
- User
- Password entry format

Expected `.pgpass` format:

```text
hostname:port:database:username:password
```

Required permission:

```bash
chmod 600 ~/.pgpass
```

## 7. Validation

After correction, validate the foreign server metadata:

```sql
SELECT
    s.srvname,
    s.srvoptions
FROM pg_foreign_server s
WHERE s.srvname = '<foreign_server_name>';
```

Validate user mapping:

```sql
SELECT
    s.srvname,
    r.rolname AS local_user,
    um.umoptions
FROM pg_user_mapping um
JOIN pg_foreign_server s
    ON s.oid = um.umserver
JOIN pg_roles r
    ON r.oid = um.umuser
WHERE s.srvname = '<foreign_server_name>';
```

Validate query access:

```sql
SELECT *
FROM <schema_name>.<foreign_table_name>
LIMIT 10;
```

Validate remote identity when possible by querying a known, non-sensitive remote object that confirms the expected source.

## 8. Recommended Operational Checks

Before changing FDW configuration in production:

| Check | Purpose |
|---|---|
| Foreign table to server mapping | Confirm which server the table uses |
| Server options | Validate host, port and dbname |
| User mapping | Validate local-to-remote credential mapping |
| Connectivity | Confirm network reachability |
| Permissions | Confirm remote user has required access |
| Data validation | Confirm the foreign table returns expected source data |
| Rollback plan | Restore previous server options if needed |

## 9. Rollback Strategy

Before changing the foreign server, capture the current configuration:

```sql
SELECT
    s.srvname,
    s.srvoptions
FROM pg_foreign_server s
WHERE s.srvname = '<foreign_server_name>';
```

Rollback example:

```sql
ALTER SERVER <foreign_server_name>
OPTIONS (
    SET host '<previous_host>',
    SET port '<previous_port>',
    SET dbname '<previous_database>'
);
```

If user mapping was changed, capture and restore mapping options as needed.

## 10. Expected Results

| Metric or Behavior | Before | After |
|---|---|---|
| Foreign table target | Incorrect server | Correct server |
| Data source | Wrong or unexpected | Expected remote source |
| Authentication | Failing or inconsistent | Validated |
| FDW metadata | Misaligned | Corrected |
| Operational confidence | Low | Improved |

## 11. Lessons Learned

The main lessons were:

- Foreign table names can hide incorrect server mappings.
- FDW troubleshooting must start with catalog metadata.
- Server options and user mappings must be validated together.
- Correcting the target server may reveal credential issues.
- `.pgpass` requires strict permission and exact host/port/database matching.
- FDW changes should always have a rollback plan.
- Validation must confirm both connectivity and data source correctness.

## 12. Business Impact

Correcting the FDW configuration reduced the risk of querying incorrect remote data.

Business benefits include:

- More reliable cross-database integration
- Lower risk of data inconsistency
- Faster troubleshooting of FDW issues
- Better operational visibility
- Improved confidence in PostgreSQL integration architecture

## 13. Conclusion

The FDW troubleshooting process identified that the local foreign table was associated with a foreign server pointing to the wrong remote target.

After correcting the server options and validating user mapping credentials, the FDW connection could be aligned with the expected remote source.

This case highlights the importance of checking PostgreSQL catalog metadata when troubleshooting foreign tables and remote data access.
