---
layout: docu
title: Athena Extension Functions
---

* [`athena_clear_cache()`](#athena_clear_cache)
* [`athena_execute(catalog, sql)`](#athena_executecatalog-sql)
* [`athena_query(catalog, sql[, ...parameter values...])`](#athena_querycatalog-sql-parameter-values)

## `athena_clear_cache()`

`athena_clear_cache` clears the table definitions and listings that the `athena` extension cached for all attached Athena data catalogs.
They are read again on the next access.

### Parameters

None.

### Returns

A table with one column, `Success` (`BOOLEAN`), and no rows.

### Example

```sql
CALL athena_clear_cache();
```

## `athena_execute(catalog, sql)`

`athena_execute` sends a SQL statement that does not return a result set to Athena, through an attached Athena data catalog.
<!-- TODO (to be resolved with the developers): writes and parameters. athena_execute takes exactly two arguments (duckdb_functions() reports no varargs), so it cannot pass statement parameter values. A parameterized INSERT or UPDATE can therefore only be run with athena_query, which accepts "one write that produces a count". Open: is athena_query the intended function for parameterized writes, which function is meant for writes without parameters, and what does athena_execute return? Once resolved, adjust the intro of both functions. -->
<!-- unchecked: the CREATE VIEW example is assumed; needs an Athena workgroup that runs the 'DuckDB' engine -->
Examples of such SQL statements are DDL statements, such as `CREATE VIEW`.
For queries, or any other SQL statements that return a result set, use [`athena_query`](#athena_querycatalog-sql-parameter-values).

### Parameters

| Parameter | Description | Mandatory |
|:--|:--|:--|
| `catalog` | The name of an attached Athena data catalog, a `VARCHAR`. | true |
| `sql` | The SQL statement to run, a `VARCHAR`. A SQL statement that does not return a result set. | true |

Both parameters are given by position.

If `catalog` is not the name of an attached database, an error is raised.

### Returns

<!-- unchecked: needs an Athena workgroup that runs the 'DuckDB' engine -->
Not verified.

### Example

<!-- unchecked: needs an Athena workgroup that runs the 'DuckDB' engine -->

```sql
CALL athena_execute('ath', 'CREATE VIEW my_glue_catalog_db.v AS SELECT 42 AS answer');
```

## `athena_query(catalog, sql[, ...parameter values...])`

`athena_query` sends a SQL statement that returns a result set to Athena, through an attached Athena data catalog, and returns that result set.
Because the SQL statement is sent as written, `athena_query` can be used to run a SQL statement that the `athena` extension would not send to Athena as a whole.
For SQL statements that do not return a result set, use [`athena_execute`](#athena_executecatalog-sql).

`sql` must be one SQL statement that reads, or one write that produces a count.
<!-- TODO (to be resolved with the developers): see the TODO at athena_execute on parameterized INSERT and UPDATE. -->
A SQL statement that cannot be classified as either raises an error before it is sent.
On an Athena workgroup that runs `Athena engine version 3`, `sql` is subject to an additional check, described in [Engine Versions]({% link docs/2.0/core_extensions/aws/athena/overview.md %}#engine-versions).

`sql` can contain parameter markers: `?` or numbered `$` markers outside strings, comments and identifiers.
The value for each parameter marker is given as a statement parameter value, after `sql`.
If `sql` contains a parameter marker and no value is given for it, an error is raised before the SQL statement is sent.

### Parameters

| Parameter | Description | Mandatory |
|:--|:--|:--|
| `catalog` | The name of an attached Athena data catalog, a `VARCHAR`. | true |
| `sql` | The SQL statement to run, a `VARCHAR`. A query, or another SQL statement that returns a result set. | true |
| statement parameter values | The values for the parameter markers in `sql`, one per parameter marker, of any type. | |

All parameters are given by position.
`athena_query` takes no named parameters.

If `catalog` is the name of a database that is not an Athena data catalog, an error is raised.

### Returns

<!-- unchecked: needs an Athena workgroup that runs the 'DuckDB' engine -->
The result set of `sql`.

### Example

<!-- unchecked: needs an Athena workgroup that runs the 'DuckDB' engine -->

```sql
SELECT *
FROM athena_query('ath', 'SELECT n_name FROM my_glue_catalog_db.nation WHERE n_regionkey = ?', 1);
```
