---
layout: docu
title: Redshift Extension
---

> Note The `redshift` extension is not a core extension.
> It is published by AWS in the `aws.amazon.com` [extension repository]({% link docs/2.0/extensions/extension_repositories.md %}).
> This page describes version `v1.1.0` of the extension on DuckDB 2.0.1.

<!-- Verification: everything on this page was checked against v1.1.0 offline only, with the DuckDB 2.0.1 CLI: function signatures, setting names, ATTACH option names and the validation of their values, and the extension's error messages. No connection to Redshift was made, because the bundled driver could not be installed in the test environment. Statements marked "unchecked" were not run; statements marked "inferred" are taken from the extension's messages. -->

The `redshift` extension allows DuckDB to run SQL statements on [Amazon Redshift](https://docs.aws.amazon.com/redshift/latest/mgmt/welcome.html), on a provisioned cluster or a Redshift Serverless workgroup.
The [`redshift_query` function]({% link docs/2.0/core_extensions/aws/redshift/functions.md %}#redshift_queryhandle-sql-params--) sends a SQL statement to Redshift and returns its result set.

Unlike the [`athena` extension]({% link docs/2.0/core_extensions/aws/athena/overview.md %}), the `redshift` extension does not make the schemas and tables of Redshift available as DuckDB schemas and tables.
SQL statements on Redshift are written in the SQL dialect of Redshift.

## Installing and Loading

The `redshift` extension is installed from the `aws.amazon.com` repository.
Register the repository [by its host]({% link docs/2.0/extensions/extension_repositories.md %}#adding-a-repository-by-its-host), then install and load the extension from it:

```sql
CREATE EXTENSION REPOSITORY aws.amazon.com;
INSTALL AND LOAD redshift FROM aws.amazon.com;
```

In later sessions, the extension is loaded from the repository, as described in [Using a Repository]({% link docs/2.0/extensions/extension_repositories.md %}#using-a-repository):

```sql
LOAD redshift FROM aws.amazon.com;
```

### Redshift ODBC Driver

The `redshift` extension connects to Redshift through the Amazon Redshift ODBC driver, which it contains.
The `redshift` extension installs the driver when a Redshift handle is first attached.
By default, the driver is installed in the directory of the extension, or in a cache directory of the user.

The environment variable `REDSHIFT_DRIVER_DIR` names another directory for the driver.
The directory must meet these requirements:

* The path is absolute and contains no symbolic link.
* The directory exists, on a file system that allows executing code.
* The directory and every directory above it are owned by the user or by `root`.
* No other user can write to the directory or to any directory above it.

If no directory meets these requirements, `ATTACH` raises an error that names the reason.

<!-- inferred: the driver is installed per version ("no valid copy of this driver version") and the directory must be owner-only (mode 0700); not observed. -->

## Credentials

The `redshift` extension supports only IAM authentication.
A database user name and password cannot be given.
The sign-in plugins of the Redshift ODBC driver, such as those for an identity provider, are not supported.

The AWS credentials come from one of two sources, chosen with the connection options of `ATTACH`:

* A DuckDB [secret]({% link docs/2.0/configuration/secrets_manager.md %}) of type `aws` or `s3`, named in the `SECRET` connection option.
* The AWS SDK that the `redshift` extension contains, selected with the `CREDENTIAL_PROVIDER` connection option.

Exactly one of `SECRET` and `CREDENTIAL_PROVIDER` is given.
If neither is given, `ATTACH` raises an error.

`CREDENTIAL_PROVIDER` takes one of these values:

* `aws_default_chain`: the default credential chain of the AWS SDK. It reads the environment, the AWS configuration files, SSO and instance metadata.
* `profile`: the AWS profile named in the `PROFILE` connection option.

A profile that assumes a role, with `role_arn` and `source_profile`, is not read by the default credential chain.
Such a profile is named with `CREDENTIAL_PROVIDER profile` and `PROFILE`.

A secret can be created with the [`credential_chain` provider]({% link docs/2.0/core_extensions/aws.md %}#credential_chain-provider) of the `aws` extension, or with the [`config` provider]({% link docs/2.0/core_extensions/httpfs/s3api.md %}#config-provider):

```sql
LOAD aws;
CREATE SECRET aws_identity (
    TYPE aws,
    PROVIDER credential_chain,
    REFRESH 'auto'
);
```

If the secret named in `SECRET` does not exist, `ATTACH` raises an error.

## Connecting

The `redshift` extension connects to Redshift with the [`ATTACH` statement]({% link docs/2.0/sql/statements/attach.md %}) and the `redshift` type.
The attached database is a Redshift handle.
A Redshift handle holds one connection to Redshift, with one Redshift session.
The alias of the Redshift handle is passed to `redshift_query`.

<!-- unchecked: no ATTACH reached Redshift -->
To attach a Redshift Serverless workgroup, run:

```sql
ATTACH '' AS rs (
    TYPE redshift,
    REGION 'eu-north-1',
    WORKGROUP 'my-workgroup',
    ACCOUNT_ID '123456789012',
    DATABASE 'dev',
    CREDENTIAL_PROVIDER 'aws_default_chain'
);
```

To attach a provisioned cluster, run:

```sql
ATTACH '' AS rs (
    TYPE redshift,
    REGION 'eu-north-1',
    CLUSTER 'my-cluster',
    DATABASE 'dev',
    SECRET 'aws_identity'
);
```

<!-- inferred: the attached database exposes no catalog ("cannot be browsed as a catalog: catalog access is not available") -->
A Redshift handle does not expose the schemas and tables of Redshift.
A query that refers to a table of the handle, such as `FROM rs.public.orders`, raises an error.

A Redshift handle cannot be named `memory`, the name of the default database of DuckDB.

<!-- inferred: CONNECT behavior from the extension's messages; CONNECT to a connection string was tested and is refused -->
As with other attached databases, the [`CONNECT` statement]({% link docs/2.0/sql/statements/connect.md %}) takes the alias of a Redshift handle, such as `CONNECT rs`.
While connected, DuckDB sends every SQL statement except `DISCONNECT` to Redshift.
`CONNECT` does not accept a connection string, such as `CONNECT 'redshift:…'`, for the `redshift` type.

When [`enable_external_access`]({% link docs/2.0/configuration/overview.md %}#configuration-reference) is `false`, `ATTACH` raises an error.
Redshift handles attached before the setting was changed can still be used.

### Targets

The target of a Redshift handle is a Redshift Serverless workgroup or a provisioned cluster.
The target is given either by connection options or by the path of `ATTACH`.

| Target | Connection options | Path |
|:--|:--|:--|
| Redshift Serverless workgroup | `WORKGROUP` and `ACCOUNT_ID` | `redshift://⟨workgroup⟩.⟨account⟩.⟨region⟩.redshift-serverless.amazonaws.com[:⟨port⟩][/⟨database⟩]` |
| Provisioned cluster | `CLUSTER` | `redshift://⟨cluster⟩.⟨label⟩.⟨region⟩.redshift.amazonaws.com[:⟨port⟩][/⟨database⟩]` |

When the target is given by connection options, the path is the empty string.
When the path names the target, the connection options `WORKGROUP`, `ACCOUNT_ID` and `CLUSTER` cannot be given.
A path that is not empty and not a `redshift://` path raises an error.

The ARN of a provisioned cluster or of a Redshift Serverless workgroup is not accepted as the path.
The error names the connection options or the path to use instead.

<!-- inferred: from the extension's messages; not run -->
An `ATTACH` without a target connects to the Redshift Serverless workgroup with express configuration, in the region named by `REGION`.
Its database is `dev`.

### Connection Options

Connection options are given in the option list of the `ATTACH` statement, after `TYPE redshift`.
Their names are case-insensitive.

| Option | Description |
|:--|:--|
| `REGION` | The AWS region of the target. See [Region](#region). |
| `WORKGROUP` | The name of a Redshift Serverless workgroup: 3 to 64 lower-case letters, digits and `-`. Requires `ACCOUNT_ID`. |
| `ACCOUNT_ID` | The 12-digit ID of the AWS account that owns the workgroup. Only valid with `WORKGROUP`. |
| `CLUSTER` | The identifier of a provisioned cluster: 1 to 63 lower-case letters, digits and `-`, starting with a letter, without two `-` in a row and not ending with `-`. |
| `DATABASE` | The Redshift database. Requires a target. Cannot be given when the path names a database. |
| `DBNAME` | The same as `DATABASE`. If both are given, they must name the same database. |
| `PORT` | The port, a number from 1 to 65535. Only valid with a path. It must be the port of the path. |
| `SECRET` | The name of a DuckDB secret of type `aws` or `s3` that holds the AWS credentials. Cannot be combined with `CREDENTIAL_PROVIDER`. |
| `CREDENTIAL_PROVIDER` | Where the AWS SDK reads the AWS credentials: `aws_default_chain` or `profile`. See [Credentials](#credentials). |
| `PROFILE` | The name of an AWS profile. Only valid with `CREDENTIAL_PROVIDER profile`, which requires it. |
| `SCHEMA` | <!-- inferred: the option name is accepted; its effect was not observed --> Not verified. |
| `ISOLATION_LEVEL` | <!-- inferred: the option name is accepted; its effect was not observed --> Not verified. |

Setting `DATABASE` or `DBNAME` to an empty string raises an error.
Giving an option name that is not in the table raises an error.
The options `HOST`, `DB_USER`, `USER` and `PASSWORD` are not accepted.
For `HOST`, the error names the path to use instead.

The general option [`READ_ONLY`]({% link docs/2.0/sql/statements/attach.md %}#options) is not accepted: a Redshift handle cannot be read-only.
Read-only access is given by Redshift credentials that only have `SELECT` grants.
<!-- inferred: the READ_WRITE requirement for a read-only DuckDB is taken from the extension's messages; not run -->
When DuckDB itself is read-only, `ATTACH` attaches read-only by default.
Then the `READ_WRITE` option is needed to attach a Redshift handle.

#### Region

The region of the target is taken from the first of these that is set:

* The `REGION` connection option.
* The region in the path.
* When `SECRET` is given: the region of that secret.
* When `SECRET` is not given: the [`s3_region` setting]({% link docs/2.0/configuration/overview.md %}#configuration-reference), then the environment variables `AWS_REGION` and `AWS_DEFAULT_REGION`.

If `REGION` and the path name different regions, `ATTACH` raises an error.
If no region is found, `ATTACH` raises an error.

<!-- inferred: the order of REGION and the path region is not stated by the extension; that they must agree was tested -->

## Transactions

DuckDB `COMMIT` and `ROLLBACK` do not control the operations that `redshift_query` runs on Redshift.
A DuckDB `ROLLBACK` does not undo them.
Therefore, `redshift_query` raises an error when it is called in an explicit DuckDB transaction, one opened with `BEGIN`.
`CONNECT` to a Redshift handle raises the same error while a DuckDB transaction is open.

When [`redshift_allow_query_in_duckdb_transaction`]({% link docs/2.0/core_extensions/aws/redshift/settings.md %}#redshift_allow_query_in_duckdb_transaction) is `true`, `redshift_query` can be called in an explicit DuckDB transaction.
`redshift_query` still raises an error on a Redshift handle that was attached in the same open DuckDB transaction.
A rollback of that transaction would detach the handle and close its Redshift session.

The Redshift session of a Redshift handle can hold state, such as an open Redshift transaction.
A Redshift transaction is opened and ended with SQL statements sent through `redshift_query`.
When a Redshift handle is detached or replaced, its Redshift session and its state are lost.
The next call on a handle with that name opens a new Redshift session.

<!-- inferred: this section is taken from the extension's messages; no transaction was run -->

## `DETACH`

To close the connection to Redshift, use [`DETACH`]({% link docs/2.0/sql/statements/attach.md %}#detach):

```sql
DETACH rs;
```

<!-- TODO (to be resolved with the developers):
- duckdb_functions() reports params as the third positional parameter of redshift_query, but the binder only accepts it by name (params := [...]).
- redshift_allow_query_in_duckdb_transaction can be SET and read with current_setting(), but is not listed by duckdb_settings().
- What do the SCHEMA and ISOLATION_LEVEL connection options do?
- Which Redshift types can be read, and how are they mapped to DuckDB types? Unsupported types raise an error that suggests a cast in the Redshift SQL.
- What does redshift_query return for a SQL statement without a result set (DDL, INSERT)?
- Is the "express configuration" workgroup reached by an ATTACH without a target a generally available Redshift feature?
- Which environment variables does the driver read (the extension refuses to connect while some are set, and names a CA bundle variable)?
-->
