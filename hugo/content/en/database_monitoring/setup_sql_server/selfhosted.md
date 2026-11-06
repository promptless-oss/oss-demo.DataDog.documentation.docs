---
title: Setting Up Database Monitoring for self-hosted SQL Server
description: Install and configure Database Monitoring for self-hosted SQL Server
further_reading:
- link: "/integrations/sqlserver/"
  tag: "Documentation"
  text: "Basic SQL Server Integration"
- link: "/database_monitoring/troubleshooting/?tab=sqlserver"
  tag: "Documentation"
  text: "Troubleshoot Common Issues"
- link: "/database_monitoring/guide/sql_deadlock/"
  tag: "Documentation"
  text: "Configure Deadlock Monitoring"
- link: "/database_monitoring/guide/sql_extended_events/"
  tag: "Documentation"
  text: "Configure Query Completion and Query Error Collection"
- link: "/database_monitoring/guide/parameterized_queries/"
  tag: "Documentation"
  text: "Capturing SQL Query Parameter Values"
- link: "https://www.datadoghq.com/blog/migrate-sql-workloads-to-azure-with-datadog/"
  tag: "Blog"
  text: "Strategize your Azure migration for SQL workloads with Datadog"
- link: "https://www.datadoghq.com/blog/datadog-monitoring-always-on/"
  tag: "Blog"
  text: "Monitor your AlwaysOn availability groups with Datadog Database Monitoring"

---

Database Monitoring provides deep visibility into your Microsoft SQL Server databases by exposing query metrics, query samples, explain plans, database states, failovers, and events.

Do the following steps to enable Database Monitoring with your database:

1. [Grant the Agent access](#grant-the-agent-access)
1. [Install the Agent](#install-the-agent)

## Before you begin

Supported SQL Server versions
: 2012, 2014, 2016, 2017, 2019, 2022, 2025 (requires Agent 7.79+)

{{% dbm-sqlserver-before-you-begin %}}

## Grant the Agent access

The Datadog Agent requires read-only access to the database server in order to collect statistics and queries.

Create a read-only login to connect to your server and grant the required permissions:

{{< tabs >}}
{{% tab "SQL Server 2014+" %}}

```SQL
CREATE LOGIN datadog WITH PASSWORD = '<PASSWORD>';
CREATE USER datadog FOR LOGIN datadog;
GRANT CONNECT ANY DATABASE to datadog;
GRANT VIEW SERVER STATE to datadog;
GRANT VIEW ANY DEFINITION to datadog;
-- If not using either of Log Shipping Monitoring (available in Agent v7.50+) or
-- SQL Server Agent Monitoring (available in Agent v7.57+), comment out the next three lines:
USE msdb;
CREATE USER datadog FOR LOGIN datadog;
GRANT SELECT to datadog;
```
{{% /tab %}}
{{% tab "SQL Server 2012" %}}

```SQL
CREATE LOGIN datadog WITH PASSWORD = '<PASSWORD>';
CREATE USER datadog FOR LOGIN datadog;
GRANT VIEW SERVER STATE to datadog;
GRANT VIEW ANY DEFINITION to datadog;
-- If not using either of Log Shipping Monitoring (available in Agent v7.50+) or
-- SQL Server Agent Monitoring (available in Agent v7.57+), comment out the next three lines:
USE msdb;
CREATE USER datadog FOR LOGIN datadog;
GRANT SELECT to datadog;
```

Create the `datadog` user in each additional application database:
```SQL
USE [database_name];
CREATE USER datadog FOR LOGIN datadog;
```
{{% /tab %}}
{{< /tabs >}}

### Create the column statistics procedure

Column statistics collection is optional and disabled by default. When enabled, the Agent collects the statistics SQL Server keeps for each column. These include row and sampling counts, modification counts, average length, and histogram steps. The Agent never collects column values: each histogram step includes only its row counts and whether its upper bound is `NULL`.

<!-- TODO for the docs writer: add the minimum Agent version for `collect_column_statistics` after it is released. -->

Column statistics collection requires SQL Server 2017 or later and `dbm: true` in the instance configuration.

SQL Server treats statistics as table data, so the read-only `datadog` user can't read them directly. Instead, create a `datadog.column_statistics` procedure **in every database** you want statistics from. The procedure runs as its owner, so create it in a `datadog` schema owned by `dbo` and grant `EXECUTE` on it to the `datadog` user.

Run the following script as a user that can create schemas and procedures in the database, such as a member of `db_owner`. Replace `<DATABASE_NAME>` with the name of the database:

```SQL
USE [<DATABASE_NAME>];
GO
IF USER_ID('datadog') IS NULL
    CREATE USER datadog FOR LOGIN datadog;
GO
IF SCHEMA_ID('datadog') IS NULL
    EXEC('CREATE SCHEMA datadog AUTHORIZATION dbo');
GO
CREATE OR ALTER PROCEDURE datadog.column_statistics @max_tables INT = 500
WITH EXECUTE AS OWNER
AS
BEGIN
    SET NOCOUNT ON;
    -- One statistics object per column: unfiltered, on the column as its leading key, the most fully sampled. A column
    -- is unique when an unfiltered unique index or primary key has it as its only key, which a sampled histogram can't
    -- tell.
    SELECT object_id, stats_id, stats_name, auto_created, user_created, column_name, type_name, max_length, is_nullable,
        is_unique, last_updated, [rows], rows_sampled, steps, modification_counter, CAST(NULL AS FLOAT) AS average_length
    INTO #stats
    FROM (
        SELECT st.object_id, st.stats_id, st.name AS stats_name, st.auto_created, st.user_created, c.name AS column_name,
            -- An alias type's base type, but a CLR type's own name: CLR types share system_type_id 240, which has none.
            TYPE_NAME(CASE WHEN c.system_type_id = 240 THEN c.user_type_id ELSE c.system_type_id END) AS type_name,
            c.max_length, c.is_nullable,
            CAST(CASE WHEN EXISTS (
                SELECT 1
                FROM sys.indexes i
                JOIN sys.index_columns ic ON ic.object_id = i.object_id AND ic.index_id = i.index_id
                    AND ic.key_ordinal = 1
                WHERE i.object_id = c.object_id AND ic.column_id = c.column_id AND i.is_unique = 1
                    AND i.has_filter = 0 AND INDEXKEY_PROPERTY(i.object_id, i.index_id, 2, 'ColumnId') IS NULL
            ) THEN 1 ELSE 0 END AS BIT) AS is_unique,
            -- As ISO 8601 text with the server's offset, formatted as the activity collector formats query_start.
            -- Older drivers such as SQLOLEDB return datetime2 values as text in their own format otherwise.
            CONVERT(NVARCHAR(40), TODATETIMEOFFSET(CAST(sp.last_updated AS DATETIME),
                DATEPART(TZOFFSET, SYSDATETIMEOFFSET())), 126) AS last_updated,
            sp.[rows], sp.rows_sampled, sp.steps, sp.modification_counter,
            ROW_NUMBER() OVER (PARTITION BY st.object_id, sc.column_id ORDER BY sp.rows_sampled DESC, sp.last_updated DESC) AS rn
        FROM (SELECT TOP (@max_tables) object_id FROM sys.tables WHERE is_ms_shipped = 0 ORDER BY SCHEMA_NAME(schema_id), name) t
        JOIN sys.stats st ON st.object_id = t.object_id AND st.has_filter = 0
        JOIN sys.stats_columns sc ON sc.object_id = st.object_id AND sc.stats_id = st.stats_id AND sc.stats_column_id = 1
        JOIN sys.columns c ON c.object_id = sc.object_id AND c.column_id = sc.column_id
        CROSS APPLY sys.dm_db_stats_properties(st.object_id, st.stats_id) sp
        WHERE sp.[rows] > 0
    ) s
    WHERE rn = 1;
    CREATE CLUSTERED INDEX ix_stats ON #stats (object_id, stats_id);

    -- A variable-width column's average length is only in the density vector, which only DBCC SHOW_STATISTICS returns.
    -- Every value of a fixed-width type takes max_length, so those skip DBCC.
    CREATE TABLE #density (all_density FLOAT, average_length FLOAT, columns NVARCHAR(4000));
    DECLARE @object_id INT, @stats_id INT, @table NVARCHAR(300), @stats_name SYSNAME, @column_name SYSNAME;
    DECLARE stats_cursor CURSOR LOCAL FAST_FORWARD FOR
        SELECT object_id, stats_id, QUOTENAME(OBJECT_SCHEMA_NAME(object_id)) + N'.' + QUOTENAME(OBJECT_NAME(object_id)),
            stats_name, column_name
        FROM #stats
        WHERE type_name IN ('varchar', 'nvarchar', 'varbinary', 'sql_variant', 'hierarchyid', 'geometry', 'geography');
    OPEN stats_cursor;
    FETCH NEXT FROM stats_cursor INTO @object_id, @stats_id, @table, @stats_name, @column_name;
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Statistics can be dropped since they were listed, as auto-created ones are. DBCC's error for them would fail
        -- the whole procedure, and TRY...CATCH doesn't catch it inside INSERT...EXEC.
        IF EXISTS (SELECT 1 FROM sys.stats WHERE object_id = @object_id AND stats_id = @stats_id)
        BEGIN
            TRUNCATE TABLE #density;
            INSERT INTO #density
            EXEC sp_executesql N'DBCC SHOW_STATISTICS (@table, @stats_name) WITH DENSITY_VECTOR, NO_INFOMSGS',
                N'@table NVARCHAR(300), @stats_name SYSNAME', @table, @stats_name;
            UPDATE #stats SET average_length = (SELECT TOP 1 average_length FROM #density WHERE columns = @column_name)
            WHERE object_id = @object_id AND stats_id = @stats_id;
        END;
        FETCH NEXT FROM stats_cursor INTO @object_id, @stats_id, @table, @stats_name, @column_name;
    END;
    CLOSE stats_cursor;
    DEALLOCATE stats_cursor;

    -- One row per column. Its histogram steps are packed in step_number order as
    -- range_high_key_is_null:equal_rows:range_rows:distinct_range_rows:average_range_rows, keeping only whether a
    -- step's range_high_key is NULL, never its value. A column whose histogram is empty, as when every value is NULL,
    -- has no steps.
    SELECT OBJECT_SCHEMA_NAME(s.object_id) AS schema_name, OBJECT_NAME(s.object_id) AS table_name, s.column_name,
        s.stats_name, s.auto_created, s.user_created, s.type_name, s.max_length, s.is_nullable, s.is_unique,
        s.last_updated, s.[rows], s.rows_sampled, s.steps, s.modification_counter, s.average_length, h.histogram
    FROM #stats s
    OUTER APPLY (
        -- STRING_AGG stops at 8000 bytes unless its input is VARCHAR(MAX); 201 steps can exceed that.
        SELECT STRING_AGG(CAST(CONCAT(CASE WHEN hh.range_high_key IS NULL THEN 1 ELSE 0 END, ':',
                CAST(hh.equal_rows AS DECIMAL(19, 2)), ':', CAST(hh.range_rows AS DECIMAL(19, 2)), ':',
                CAST(hh.distinct_range_rows AS DECIMAL(19, 2)), ':', CAST(hh.average_range_rows AS DECIMAL(19, 4)))
                AS VARCHAR(MAX)), ',') WITHIN GROUP (ORDER BY hh.step_number) AS histogram
        FROM sys.dm_db_stats_histogram(s.object_id, s.stats_id) hh
    ) h
    ORDER BY schema_name, table_name, s.column_name;
END;
GO
GRANT EXECUTE ON datadog.column_statistics TO datadog;
GO
```

To verify the setup, run the procedure as the `datadog` user. It returns a row for each column with statistics in the first table, without an error:

```SQL
EXECUTE AS USER = 'datadog';
EXEC datadog.column_statistics @max_tables = 1;
REVERT;
```

After the procedure exists, enable collection in the SQL Server instance configuration:

```yaml
instances:
  - dbm: true
    ...
    collect_column_statistics:
      enabled: true
      collection_interval: 3600
      max_tables: 500
```

| Option | Default | Description |
| --- | --- | --- |
| `enabled` | `false` | Set to `true` to collect column statistics. |
| `collection_interval` | `3600` | Seconds between collections. |
| `max_tables` | `500` | Maximum number of tables to collect column statistics for in each database. This limit is separate from the `max_tables` option under `collect_schemas`. |

The Agent collects column statistics from the database set in `database`, or from every database that `database_autodiscovery` finds. It skips system databases and databases that are offline or that the Agent can't access, such as a non-readable availability group secondary. In each database, the Agent collects the first `max_tables` user tables, ordered by schema and table name. A column is included only when it's the leading column of an unfiltered statistics object on a table with rows. SQL Server creates these statistics for indexes, and also for columns that queries filter on when the `AUTO_CREATE_STATISTICS` database option is on.

If the Agent logs that `datadog.column_statistics is outdated` after an upgrade, run the script again. `CREATE OR ALTER` replaces the existing procedure. For other errors, see [Troubleshooting][1].

### Securely store your password
{{% dbm-secret %}}

## Install the Agent

It's recommended to install the agent directly on the SQL Server host as that enables the agent to collect a variety of system telemetry (CPU, memory, disk, network) in addition to SQL Server specific telemetry.

{{< tabs >}}
{{% tab "Windows Host" %}}
{{% dbm-alwayson %}}
{{% dbm-sqlserver-agent-setup-windows %}}
{{% /tab %}}
{{% tab "Linux Host" %}}
{{% dbm-alwayson %}}
{{% dbm-sqlserver-agent-setup-linux %}}
{{% /tab %}}
{{< /tabs >}}

## Example Agent Configurations
{{% dbm-sqlserver-agent-config-examples %}}

## Further reading

{{< partial name="whats-next/whats-next.html" >}}

[1]: /database_monitoring/setup_sql_server/troubleshooting/#column-statistics-are-not-collected
