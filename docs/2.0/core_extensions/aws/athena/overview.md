---
layout: docu
title: Athena Extension
---

> Note The `athena` extension is not a core extension.
> It is published by AWS in the `aws.amazon.com` [extension repository]({% link docs/2.0/extensions/extension_repositories.md %}).
> This page describes version `v1.0.1209.0` of the extension on DuckDB 2.0.1.

The `athena` extension allows DuckDB to attach an [Amazon Athena](https://docs.aws.amazon.com/athena/latest/ug/what-is.html) data catalog as a database.
The tables of the attached database can be read like DuckDB tables.
The [`athena_query`]({% link docs/2.0/core_extensions/aws/athena/functions.md %}#athena_querycatalog-sql-parameter-values) and [`athena_execute`]({% link docs/2.0/core_extensions/aws/athena/functions.md %}#athena_executecatalog-sql) functions send a SQL statement to Athena as written.

## Installing and Loading

The `athena` extension is installed from the `aws.amazon.com` repository.
Register the repository [by its host]({% link docs/2.0/extensions/extension_repositories.md %}#adding-a-repository-by-its-host), then install and load the extension from it:

```sql
CREATE EXTENSION REPOSITORY aws.amazon.com;
INSTALL AND LOAD athena FROM aws.amazon.com;
```

In later sessions, the extension is loaded from the repository, as described in [Using a Repository]({% link docs/2.0/extensions/extension_repositories.md %}#using-a-repository):

```sql
LOAD athena FROM aws.amazon.com;
```

The `athena` extension depends on the [`httpfs` extension]({% link docs/2.0/core_extensions/httpfs/overview.md %}).
When the `athena` extension is loaded, the `httpfs` extension is loaded automatically.
<!-- inferred: the reason for the dependency is not verified; the credentials of the `config` provider (TYPE aws/s3 secrets) come from httpfs -->

## Credentials

The `athena` extension signs its requests to AWS with the credentials of a DuckDB [secret]({% link docs/2.0/configuration/secrets_manager.md %}) of type `aws` or `s3`.
The secret must contain credentials and must not have a `SCOPE`.
The `athena` extension does not read credentials from environment variables or AWS configuration files itself.

A secret with the [`credential_chain` provider]({% link docs/2.0/core_extensions/aws.md %}#credential_chain-provider) of the `aws` extension reads credentials from the environment, the AWS configuration files, SSO, roles and instance metadata:

```sql
LOAD aws;
CREATE SECRET aws_identity (
    TYPE aws,
    PROVIDER credential_chain,
    REFRESH 'auto'
);
```

A secret with the [`config` provider]({% link docs/2.0/core_extensions/httpfs/s3api.md %}#config-provider) takes the credentials as values:

```sql
CREATE SECRET aws_identity (
    TYPE aws,
    KEY_ID '⟨key_id⟩',
    SECRET '⟨secret⟩',
    SESSION_TOKEN '⟨session_token⟩'
);
```

## Connecting

To attach an Athena data catalog, use the [`ATTACH` statement]({% link docs/2.0/sql/statements/attach.md %}) with the `athena` type.
The path is an `athena://` path or the ARN of an Athena workgroup:

```sql
ATTACH 'athena://eu-north-1' AS ath (TYPE athena);
```

The path has these forms:

| Path | Athena workgroup |
|:--|:--|
| `athena://⟨region⟩` | The Athena workgroup set by [`athena_workgroup`]({% link docs/2.0/core_extensions/aws/athena/settings.md %}), `primary` by default |
| `athena://⟨region⟩/⟨workgroup⟩` | `⟨workgroup⟩` |
| `arn:aws:athena:⟨region⟩:⟨account⟩:workgroup/⟨workgroup⟩` | `⟨workgroup⟩` |

A path with more than one part after the region raises an error.
The account in an ARN is not checked against the account of the credentials.

If no secret as described in [Credentials](#credentials) is available, `ATTACH` raises an error.

The Athena workgroup must store query results in Athena-managed storage or configure an S3 result location.
The `athena` extension does not set a result location itself.
If the Athena workgroup has neither, `ATTACH` raises an error.

<!-- inferred: CONNECT reaches the credentials check (needs a secret); not run against Athena -->
As with other attached databases, the [`CONNECT` statement]({% link docs/2.0/sql/statements/connect.md %}) takes the alias of an attached Athena data catalog, such as `CONNECT ath`, or a connection string prefixed with the type, such as `CONNECT 'athena:athena://eu-north-1'`.

### Mapping of Athena Objects

Athena and DuckDB organize tables in different hierarchies.
Athena has data catalogs, which contain databases, which contain tables.
DuckDB has databases, which contain schemas, which contain tables.

An attached Athena data catalog is mapped as follows:

| Athena | DuckDB |
|:--|:--|
| Data catalog | Database, named by the alias in `ATTACH` |
| Database | Schema |
| Table | Table |

The tables of an attached data catalog are referred to with a [qualified name]({% link docs/2.0/sql/statements/attach.md %}#name-qualification) of the form `⟨alias⟩.⟨athena_database⟩.⟨table⟩`.
For example, the table `nation` in the Athena database `my_glue_catalog_db`, attached as `ath`, is `ath.my_glue_catalog_db.nation`:

<!-- unchecked: needs an Athena workgroup that runs the 'DuckDB' engine -->

```sql
SELECT n_name
FROM ath.my_glue_catalog_db.nation
WHERE n_regionkey = 1;
```

The `Database` connection option sets the default schema of the attached database.
After `USE`, the tables of that Athena database can be referred to by their name alone:

```sql
ATTACH 'athena://eu-north-1' AS ath (
    TYPE athena,
    Database 'my_glue_catalog_db'
);
USE ath;
SHOW TABLES;
```

Athena types are mapped to DuckDB types:

| Athena type | DuckDB type |
|:--|:--|
| `bigint` | `BIGINT` |
| `int` | `INTEGER` |
| `decimal(p,s)` | `DECIMAL(p,s)` |
| `date` | `DATE` |
| `string`, `char`, `varchar` | `VARCHAR` |

### Connection Options

Connection options are given in the option list of the `ATTACH` statement, after `TYPE athena`.
Their names are case-insensitive.
Each option takes a string value.

| Option | Description |
|:--|:--|
| `Region` | The AWS region. If the path also names a region, the two must agree. |
| `WorkGroup` | The Athena workgroup. Takes precedence over `athena_workgroup`. |
| `Catalog` | The Athena data catalog to attach. The data catalog of the AWS Glue Data Catalog is named `AwsDataCatalog`. |
| `Database` | The Athena database that becomes the default schema of the attached database. |
| `Schema` | <!-- inferred: name from the option list only --> Not verified. |
| `S3Tables` | The S3 table buckets to expose alongside the data catalog: a list of names, or `'*'` or `'all'` for every table bucket. |
| `Secret` | <!-- inferred: name from the option list only --> The name of the secret to sign requests with. |
| `EngineVersion` | The engine to run queries on. Only `'DuckDB'` is accepted. See [Engine Versions](#engine-versions). |
| `EncryptionOption` | <!-- inferred: name from the option list only --> The encryption of query results. |
| `KmsKey` | <!-- inferred: name from the option list only --> The KMS key for the encryption of query results. |

To attach the AWS Glue Data Catalog through the Athena workgroup `primary` and make the Athena database `my_glue_catalog_db` the default schema, run:

```sql
ATTACH 'athena://eu-north-1' AS ath (
    TYPE athena,
    WorkGroup 'primary',
    Catalog 'AwsDataCatalog',
    Database 'my_glue_catalog_db'
);
```

Setting an option to an empty string raises an error.
Setting an option to a value that contains whitespace raises an error.
An option without a value is read by `ATTACH` as the `BOOLEAN` value `true`.
Because the options take a string, giving one without a value raises an error.
Giving an option name that is not in the table raises an error that lists the options in the table.

The general options of `ATTACH` also apply.
For example, the [`READ_ONLY` option]({% link docs/2.0/sql/statements/attach.md %}#options) attaches the data catalog in read-only mode:

```sql
ATTACH 'athena://eu-north-1' AS ath (TYPE athena, READ_ONLY);
```

<!-- inferred: READ_ONLY is accepted by ATTACH; its effect on writes was not tested -->

## Query Execution

The `athena` extension decides for each statement on an attached data catalog where it runs.
The part of a SQL statement that the `athena` extension sends to Athena is a query fragment.
A SQL statement can result in one or more query fragments.

### Execution Strategies

When [`athena_enable_remote_pushdown`]({% link docs/2.0/core_extensions/aws/athena/settings.md %}#athena_enable_remote_pushdown) is `true`, a SQL statement that only reads tables of one attached data catalog is sent to Athena as a whole, as one query fragment.
Athena runs the SQL statement and returns its result.
`EXPLAIN` shows such a statement as a single `Athena Arrow Scan`.
This applies to filters, aggregates and joins between tables of the same attached data catalog.

When `athena_enable_remote_pushdown` is `false`, DuckDB runs the SQL statement.
Each Athena table in it is read by a query fragment of its own.
That query fragment selects only the columns DuckDB needs.
When [`athena_enable_filter_pushdown`]({% link docs/2.0/core_extensions/aws/athena/settings.md %}#athena_enable_filter_pushdown) is `true`, it also applies the filters on that table.
`EXPLAIN` then shows DuckDB operators, such as an aggregate, above one `Athena Arrow Scan` per table.
`athena_enable_remote_pushdown` is used when the data catalog is attached.

> Note A statement that combines an attached Athena table with local data sends a query to Athena while the statement is planned.
> Therefore, `EXPLAIN` of such a statement already runs a query on Athena.

### Result Transfer

<!-- TODO (open issue, revisit): the result transfer needs fuller documentation, probably in an architecture section of this overview. Per other docs the user has, the Arrow result stream is a new Athena feature built specifically for DuckDB; explain how it relates to the CSV/tab-delimited result file in S3 and to GetQueryResults, and when each is used. Source for the Arrow claim still to be added. -->
Athena runs each query fragment and stores its result.
The `athena` extension then transfers the result to DuckDB.
It can do so in these ways:

* The Arrow result stream: the result is streamed in Arrow format. This is only available on the `DuckDB` engine, on an Athena workgroup that stores results in Athena-managed storage.
* The result file: the `athena` extension reads the CSV or tab-delimited file that Athena writes to the S3 result location of the Athena workgroup.
* The `GetQueryResults` API of Athena, which returns the result in pages of 1000 rows.

By default, the `athena` extension chooses one of these.
[`athena_result_fetcher`]({% link docs/2.0/core_extensions/aws/athena/settings.md %}#athena_result_fetcher) selects one explicitly.
The other settings that apply to the result transfer are listed in [Result Transfer]({% link docs/2.0/core_extensions/aws/athena/settings.md %}#result-transfer) on the settings page.

<!-- inferred: from the setting descriptions and the extension's messages; no result transfer was observed -->

### Metadata Cache

The `athena` extension caches the table definitions and listings it reads from the data catalog.
How long they are kept is set by [`athena_metadata_cache_scope`]({% link docs/2.0/core_extensions/aws/athena/settings.md %}):

| Value | Meaning |
|:--|:--|
| `transaction` | The definitions are read again in each transaction. Changes made by other clients become visible between statements. This is the default. |
| `connection` | The definitions are kept until a rollback, a write through this connection, or a call of `athena_clear_cache`. |

A change of the setting applies from the next transaction.
The [`athena_clear_cache` function]({% link docs/2.0/core_extensions/aws/athena/functions.md %}#athena_clear_cache) clears the cache.

## Engine Versions

<!-- TODO: to be clarified by the developers of the extension: how engine selection is supposed to work, which accounts and workgroups support the 'DuckDB' engine, and whether 'Athena engine version 3' is meant to be supported. Also: which of athena_query and athena_execute is meant for writes such as INSERT (athena_query accepts "one write that produces a count"), and what athena_execute returns. The text below describes observed behavior of v1.0.1209.0. -->

> Warning How the `athena` extension selects the Athena engine is to be clarified by its developers.
> This section describes the behavior observed with version `v1.0.1209.0`.

Athena runs queries on an engine.
Each Athena workgroup selects an engine version, such as `Athena engine version 3`.
Statements on the tables of an attached data catalog, `athena_query` and `athena_execute` run queries on Athena.
Listing the schemas and tables of an attached data catalog does not run a query.

The `athena` extension runs queries only on the `DuckDB` engine.
If the Athena workgroup runs `Athena engine version 3`, the `athena` extension sends the engine name `DuckDB` with every query.
Athena accepts this only on accounts enabled for query-level engine selection.
On other accounts, every statement that runs a query raises the error Athena returns.

On an Athena workgroup that runs `Athena engine version 3`, [`athena_query`]({% link docs/2.0/core_extensions/aws/athena/functions.md %}#athena_querycatalog-sql-parameter-values) checks that both engines read its SQL statement as a read.
A SQL statement that both engines do not read as a read raises an error before it is sent.

Selecting `Athena engine version 3` raises an error:

* The `EngineVersion` connection option accepts only `'DuckDB'`.
* While [`athena_engine_version`]({% link docs/2.0/core_extensions/aws/athena/settings.md %}) is `'Athena engine version 3'`, every statement on a data catalog attached on the `DuckDB` engine raises an error before anything is sent to Athena.

## `DETACH`

To detach the data catalog, use [`DETACH`]({% link docs/2.0/sql/statements/attach.md %}#detach):

```sql
DETACH ath;
```
