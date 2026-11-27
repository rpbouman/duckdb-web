---
layout: docu
title: Redshift Extension Functions
---

* [`redshift_query(handle, sql[, params := [...]])`](#redshift_queryhandle-sql-params--)

## `redshift_query(handle, sql[, params := [...]])`

`redshift_query` sends a SQL statement to Redshift through a Redshift handle and returns its result set.
The SQL statement is written in the SQL dialect of Redshift and sent as written.

<!-- inferred: from the extension's messages -->
While the DuckDB statement that calls `redshift_query` is planned, `redshift_query` asks Redshift to describe the SQL statement.
This determines the columns of the result set without running the SQL statement.
If the SQL statement returns other columns when it runs, an error is raised and the SQL statement is not run.
For a prepared DuckDB statement, the description is made when it is prepared.

`sql` can contain parameter markers.
The value for each parameter marker is given in `params`, as a list of strings.
<!-- inferred: the parameter marker syntax of Redshift SQL ('?') is not stated by the extension -->

### Parameters

| Parameter | Description | Mandatory |
|:--|:--|:--|
| `handle` | The alias of an attached Redshift handle, a `VARCHAR`. | true |
| `sql` | The SQL statement to run, a `VARCHAR`. | true |
| `params` | The values for the parameter markers in `sql`, a `VARCHAR[]`. | |

`handle` and `sql` are given by position.
`params` is given by name, as in `params := ['a', 'b']`.

<!-- TODO (to be resolved with the developers): duckdb_functions() lists params as a third positional parameter, but a third positional argument is refused. -->

The following raise an error before anything is sent to Redshift:

* `handle` or `sql` is `NULL` or the empty string.
* `handle` is not the alias of an attached database, or is the alias of a database that is not a Redshift handle.
* `params` is `NULL`.
* An element of `params` is `NULL`. To pass `NULL`, write `NULL` in `sql`.
* An element of `params` contains a NUL byte, or is longer than the Redshift ODBC driver can send.
* `sql` contains a NUL byte.

### Returns

<!-- unchecked: no SQL statement was run on Redshift -->
The result set of `sql`.
A SQL statement that returns more than one result set raises an error.
A result column of a Redshift type that the `redshift` extension cannot read raises an error before the SQL statement is run.
Such a column can be cast to a supported type, such as `VARCHAR`, in `sql`.

### Restrictions

<!-- inferred: from the extension's messages; not run -->
A Redshift handle has one connection to Redshift, which runs one SQL statement at a time.
This leads to the following restrictions.
Each raises an error, and no SQL statement is run on Redshift.

* A DuckDB statement can use a Redshift handle in only one `redshift_query` call.
* `redshift_query` cannot be called on a Redshift handle while the result of an earlier call on that handle is still being read.
* `redshift_query` cannot be used in the recursive part of a recursive CTE, because it would run once per iteration.
* `redshift_query` cannot be the source of a `PIVOT` without an `IN` list, because DuckDB would run it once per pivoted column.

In each case, the result can be stored in a DuckDB temporary table by a separate DuckDB statement first.

`redshift_query` cannot be called in an explicit DuckDB transaction, unless [`redshift_allow_query_in_duckdb_transaction`]({% link docs/2.0/core_extensions/aws/redshift/settings.md %}#redshift_allow_query_in_duckdb_transaction) is `true`.
See [Transactions]({% link docs/2.0/core_extensions/aws/redshift/overview.md %}#transactions).

### Example

<!-- unchecked: no SQL statement was run on Redshift -->

```sql
SELECT *
FROM redshift_query(
    'rs',
    'SELECT order_id, amount FROM public.orders WHERE status = ?',
    params := ['shipped']
);
```
