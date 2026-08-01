# Commercient Data Lake \ Data Hub — overview

> This is a short public overview of what the platform does.
>
> **The full, current, authoritative help lives inside the product** — the Help section
> of the web console, or from your terminal once you are signed in:
>
> ```bash
> dlake guide help                    # the whole help document
> dlake guide help --section events   # one section
> dlake guide api                     # the API usage guide
> dlake guide cli                     # the CLI reference
> ```
>
> Those commands fetch the live copy from your own tenant, so they always match the
> version you are actually running. Feature detail, permission requirements and API
> semantics are documented there rather than here.

## What it is

A managed data platform on Microsoft SQL Server. Each tenant gets its own database and its
own API surface. You define a schema, load data into it, and serve it out — to
applications, to BI tools, and to AI agents.

## What you can do with it

**Build a schema.** Tables, views, stored procedures, functions, triggers, indexes and
full-text search, from a visual schema builder, the REST API, or this CLI.

**Get data in.** Upload CSV, Excel, JSON, XML or Parquet files and have the table created
and kept in sync. Connect a source system — HubSpot, Stripe, Salesforce, ServiceTitan, SQL
Server, ODBC — and pull it on a schedule, or receive inbound webhooks from a vendor. An
on-premise agent covers sources that live behind a firewall.

**Query and explore.** A data browser, a SQL editor, saved queries, and a
natural-language "ask your data" box. Background exports to CSV and Parquet.

**Serve it out.** A per-tenant REST and GraphQL API over exactly the entities you choose to
expose, with API keys, and MCP connectors so AI assistants can work with your data
directly — a data plane for records and queries, and an administrative plane for setup.

**Track change.** Row-change events by server-sent events, polling or signed webhooks;
change tracking for incremental sync; time travel over row history; and optimistic
concurrency protection for multi-writer tables.

**Govern it.** Role-based access, API keys that can be narrowed per key, row-level
security, audit logs, and data-quality rules with alerting.

**Extend to object storage.** Link S3-compatible buckets, browse and transfer objects,
export tables straight into a bucket, and on SQL Server 2022+ attach Parquet and CSV
files as queryable external tables.

**Documents and search.** Upload documents, extract and search their text, and run
semantic search over a knowledge corpus.

## Using the CLI

`dlake` drives all of the above from a terminal or CI. Human-readable output by default,
`--json` everywhere for scripting, and single-file binaries with no runtime to install.
See the [README](../README.md) for install and sign-in, and `dlake guide cli` for the
complete command reference.

Exit codes: **0** success, **1** error, **2** usage, **3** permission or authentication denied.

## Getting help

- The in-product Help section, or `dlake guide help`
- `dlake <command> --help` for any command
- Support: your Commercient account contact
