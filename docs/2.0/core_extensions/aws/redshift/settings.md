---
layout: docu
title: Redshift Extension Settings
---

The `redshift` extension adds the following [setting]({% link docs/2.0/configuration/overview.md %}).

<!-- TODO (to be resolved with the developers): the setting can be SET and read with current_setting(), but duckdb_settings() does not list it. -->

## Transactions

### `redshift_allow_query_in_duckdb_transaction`

Whether [`redshift_query`]({% link docs/2.0/core_extensions/aws/redshift/functions.md %}#redshift_queryhandle-sql-params--) can be called in an explicit DuckDB transaction, one opened with `BEGIN`.
When `redshift_allow_query_in_duckdb_transaction` is `false`, such a call raises an error.
When it is `true`, the call runs.
DuckDB `COMMIT` and `ROLLBACK` then do not control what the call does on Redshift.
A DuckDB `ROLLBACK` does not undo it.

When `redshift_allow_query_in_duckdb_transaction` is `true`, `redshift_query` still raises an error on a Redshift handle that was attached in the same open DuckDB transaction.
See [Transactions]({% link docs/2.0/core_extensions/aws/redshift/overview.md %}#transactions).

Setting the value to `true` writes a notice that says so.

* Type: `BOOLEAN`
* Default: `false`
