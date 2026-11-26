---
layout: docu
title: Athena Extension Settings
---

The `athena` extension adds the following [settings]({% link docs/2.0/configuration/overview.md %}).
All of them are global.

Some settings are used when a data catalog is attached.
A change of such a setting affects only data catalogs attached after the change.
Other settings are used each time a SQL statement sends a query to Athena.
A change of such a setting affects the next SQL statement.
Each setting below states which applies.

The terms query fragment and result transfer are explained in [Query Execution]({% link docs/2.0/core_extensions/aws/athena/overview.md %}#query-execution).

## Query Execution

### `athena_enable_remote_pushdown`

Whether a SQL statement that only reads tables of one attached data catalog is sent to Athena as a whole.
When `athena_enable_remote_pushdown` is `false`, each Athena table in such a SQL statement is read by a query fragment of its own, and DuckDB runs the rest of the plan.

When `athena_enable_remote_pushdown` is `false`, a write or DDL statement is still sent to Athena as a whole if every query it contains is a view body, a `VALUES` list or a `SELECT` without `FROM`.

The setting is used when a data catalog is attached.

* Type: `BOOLEAN`
* Default: `true`

### `athena_enable_filter_pushdown`

Whether the filters on an Athena table are added to the query fragment that reads the table.
When `athena_enable_filter_pushdown` is `true`, Athena applies the filters, and only the rows that pass them are transferred to DuckDB.
When `athena_enable_filter_pushdown` is `false`, the query fragment reads all rows of the table, and DuckDB applies the filters after the transfer.

<!-- inferred: from the setting's description ("Whether or not to use filter pushdown") and the EXPLAIN text "no filter on this table travels"; the effect of false was not run -->
The setting applies to query fragments that read one table.
A SQL statement that is sent to Athena as a whole contains its filters.

* Type: `BOOLEAN`
* Default: `true`

### `athena_engine_version`

The engine name sent with every query fragment, for data catalogs attached without the `EngineVersion` connection option.
The values are:

* `DuckDB`
* `Athena engine version 3`
* the empty string, which is the default: the engine that the data catalog selected when it was attached is used

The values are case-sensitive.
Any other value raises an error.
See [Engine Versions]({% link docs/2.0/core_extensions/aws/athena/overview.md %}#engine-versions).

The setting is used each time a SQL statement sends a query fragment to Athena.

* Type: `VARCHAR`
* Default: empty

### `athena_workgroup`

The Athena workgroup for data catalogs attached without a workgroup in the path or in the `WorkGroup` connection option.

The setting is used when a data catalog is attached.

* Type: `VARCHAR`
* Default: `primary`

### `athena_timeout_seconds`

The maximum number of seconds a query fragment may run on Athena.
When a query fragment runs longer, the `athena` extension stops it on Athena, and the SQL statement fails.
`0` selects the built-in default.
A value above one year is used as one year.

The setting is used each time a SQL statement sends a query fragment to Athena.

* Type: `UBIGINT`
* Default: `900`

### `athena_poll_ms`

The `athena` extension repeatedly asks Athena for the state of a running query fragment.
`athena_poll_ms` is the maximum number of milliseconds between two of these requests.
During the first five seconds, the waits start at 20 milliseconds, or at `athena_poll_ms` if that is lower, and double up to `athena_poll_ms`.
After that, each wait doubles, up to 2000 milliseconds, but is never shorter than `athena_poll_ms`.
`0` selects the built-in default.
A value above 60000 is accepted, but used as 60000.

The setting is used each time a SQL statement sends a query fragment to Athena.

* Type: `UBIGINT`
* Default: `200`

## Result Transfer

### `athena_result_fetcher`

How the result of a query fragment is transferred from Athena to DuckDB.
The values are:

| Value | Result transfer |
|:--|:--|
| `auto` | The Arrow result stream on the `DuckDB` engine with Athena-managed storage. Otherwise, the result file in S3. If the result file is client-side encrypted, or S3 denies access to it, the `GetQueryResults` API. This is the default. |
| `GetQueryResultStream` | The Arrow result stream. Only available on the `DuckDB` engine with Athena-managed storage. |
| `S3` | The result file in S3 only. |
| `GetQueryResults` | The `GetQueryResults` API of Athena, in pages of 1000 rows. DuckDB does not read from S3. |
| `GetQueryResultsStream` | The result stream of `Athena engine version 3`. Refused, because no result on the `DuckDB` engine is transferred this way. |

The values are case-insensitive.
A value that the engine and the result storage of the Athena workgroup cannot serve raises an error before the query fragment is sent.

The setting is used each time a SQL statement sends a query fragment to Athena.

* Type: `VARCHAR`
* Default: `auto`

### `athena_fetch_size`

The target number of rows per read when the result of a query fragment is read from the result file in S3.
Athena writes that file as CSV or tab-delimited text.
The `athena` extension reads it in consecutive byte ranges.
The first range is 256 KiB.
The size of each next range is computed from the bytes per row read so far and `athena_fetch_size`, between 256 KiB and 10 MiB.

`0` selects the default of 10000.
The maximum is 1000000.
The setting does not limit the size of a row or the memory used for the result.
It does not apply to the other kinds of result transfer.

The setting is used each time the result file of a query fragment is opened.

* Type: `UBIGINT`
* Default: `0`

### `athena_decode_threads`

The number of threads that decode the Arrow result stream of one query fragment.
`1` decodes on the thread that reads the result.
`0` selects the built-in default.

* Type: `UBIGINT`
* Default: `4`

### `athena_stream_buffer_bytes`

The maximum number of bytes of the Arrow result stream of one query fragment that are held in memory at once.
This includes received data that is not yet decoded, and decoded data that DuckDB has not yet consumed.

* Type: `UBIGINT`
* Default: `8388608` (8 MiB)

### `athena_stream_stall_seconds`

The maximum number of seconds the result transfer of a query fragment may receive no data.
When no data arrives for longer, the query fragment is abandoned, and the SQL statement fails.
The time only counts while DuckDB is waiting for data.
A slow consumer on the DuckDB side does not count.
`0` disables the check.

* Type: `UBIGINT`
* Default: `300`

### `athena_max_fragment_bytes`

The maximum number of bytes the result transfer of one query fragment may receive.
The bytes are counted as they arrive, over the whole result of the query fragment.
When the count exceeds `athena_max_fragment_bytes`, the query is aborted.
`0` disables the limit.

The limit protects against a result that does not end.
It is a limit on the volume of one query fragment's result.
It does not limit the memory used for the result: [`athena_stream_buffer_bytes`](#athena_stream_buffer_bytes) does that.
It does not limit a SQL statement as a whole, because a SQL statement can consist of several query fragments.

* Type: `UBIGINT`
* Default: `68719476736` (64 GiB)

## Metadata

### `athena_metadata_cache_scope`

How long the `athena` extension keeps the table definitions and listings it reads from the data catalog.
The values are:

* `transaction`: the definitions are read again in each transaction. Changes made by other clients become visible between SQL statements. This is the default.
* `connection`: the definitions are kept until a rollback, a write through this connection, or a call of [`athena_clear_cache`]({% link docs/2.0/core_extensions/aws/athena/functions.md %}#athena_clear_cache).

The values are case-insensitive.

A change of the setting applies from the next transaction.

* Type: `VARCHAR`
* Default: `transaction`

### `athena_schema_cache_entries`

Before a query fragment is sent, the `athena` extension determines the columns of its result.
For that, it runs a metadata query on Athena while the SQL statement is planned.
`athena_schema_cache_entries` is the number of these result schemas that one connection keeps across SQL statements.
A SQL statement that sends a kept query fragment again does not run the metadata query again.
When the number is exceeded, the least recently used result schema is dropped.
`0` keeps a result schema only for the SQL statement that determined it.

* Type: `UBIGINT`
* Default: `100`

### `athena_lazy_schema_listing`

Whether attaching a data catalog skips listing its Athena databases.
When `athena_lazy_schema_listing` is `true`, an Athena database named in a SQL statement is looked up on its own.
The full list is read when a SQL statement needs all schemas, such as `SHOW ALL TABLES`.
A data catalog attached with the `S3Tables` connection option is always listed when it is attached.

The setting is used when a data catalog is attached.

* Type: `BOOLEAN`
* Default: `true`

### `athena_single_table_lookup`

Whether a table named in a SQL statement is looked up on its own, instead of listing all tables of its Athena database first.
This applies until the Athena database has been listed.

The setting is used when a data catalog is attached.

* Type: `BOOLEAN`
* Default: `true`

### `athena_metadata_concurrency`

Some SQL statements need the tables of all Athena databases of a data catalog, such as `SHOW ALL TABLES`.
For these, the `athena` extension requests the table lists of several Athena databases at the same time.
`athena_metadata_concurrency` is the number of Athena databases whose tables are listed at the same time, per attached data catalog.
`1` lists them one after another.

Athena limits the rate of these requests per AWS account, for all clients of the account together.
When Athena throttles a request, the remaining Athena databases are listed one at a time.

<!-- inferred: SET athena_metadata_concurrency = 0 is accepted without an error; what 0 means is unknown -->

* Type: `UBIGINT`
* Default: `2`

## Endpoints and Logging

### `athena_endpoint`

The Athena endpoint that query fragments are sent to.
The empty string, the default, means `https://athena.⟨region⟩.amazonaws.com` in the region of the attached data catalog.

* Type: `VARCHAR`
* Default: empty

### `athena_glue_endpoint`

The AWS Glue endpoint used to list the S3 table buckets when the `S3Tables` connection option is `'*'` or `'all'`.
The empty string, the default, means `https://glue.⟨region⟩.amazonaws.com` in the region of the attached data catalog.

* Type: `VARCHAR`
* Default: empty

### `athena_log_http`

Whether the requests to Athena and AWS Glue are written to the DuckDB HTTP log.
Reads of the result transfer are not logged.

* Type: `BOOLEAN`
* Default: `false`
