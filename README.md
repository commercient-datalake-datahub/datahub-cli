<p align="center">
  <img src="assets/logo.svg" alt="Commercient Data Lake \ Data Hub" width="360" />
</p>

# dlake — Commercient Data Lake \ Data Hub CLI

> **A place to aggregate all your company data from multiple systems so that AI
> agents of any type can see it all — and, if set up, even write data back.
> Code your own solutions against all your data, safely and securely. Let loose
> your staff's creativity on your data without risk.**

`dlake` is the official cross-platform command-line client for the
**Commercient Data Lake \ Data Hub** — in the spirit of the Stripe and HubSpot
CLIs. It authenticates with a tenant API key, manages profiles, and wraps the
platform's REST and admin surfaces with a scripting-friendly UX.

> **Binary distribution.** This repository publishes the official `dlake`
> binaries and release notes. The source code is not published here.

## What is Commercient Data Lake \ Data Hub?

A managed, per-tenant data platform **built on Microsoft SQL Server** that turns
your business data into an API- and AI-ready lake. Every tenant gets its own
managed SQL Server database — schema builder, Data API, events, time travel,
row-level security, and object-storage attach are all **native Microsoft SQL
Server** capabilities, not a bolt-on store. (PolyBase external-table attach and
vector features require SQL Server 2022+ / 2025.)

- **Bring data in** — through **Commercient SYNC**, Commercient Data Lake \ Data
  Hub can **read and write to and from more than 150 systems** (ERPs, CRMs,
  e-commerce, and more — see [www.commercient.com](https://www.commercient.com)).
  Built-in connectors cover HubSpot, Stripe, Salesforce, ServiceTitan, SQL
  Server, and any ODBC source (via a small on-prem push agent), with incremental
  sync, change tracking, scheduling, integrity verification, and conflict
  handling for bi-directional flows. File ingest (CSV / Parquet / XML) creates
  and evolves tables automatically.
- **Shape it** — a full schema builder (tables, views, procedures, triggers,
  indexes, functions), a data browser, and a SQL editor with background
  CSV/Parquet exports.
- **Serve it** — a per-tenant Data API (REST + GraphQL) over exactly the
  entities you expose, plus **MCP connectors for AI agents** (a data plane and
  an admin control plane with 116 tools), natural-language querying, row-change
  events (SSE / polling / signed webhooks), time travel, and row-level
  security.
- **Govern it** — role-based permissions, scoped API keys enforced down to
  entity and field level *in the database* (fail-closed), audit logs, and data
  quality rules with alerting.
- **Extend it to object storage** — link S3 buckets, browse/upload/download,
  export tables straight to a bucket, and (on SQL Server 2022+) attach parquet
  and CSV files as queryable external tables with automatic schema discovery.

`dlake` is the terminal/CI way to drive all of it. The full product help lives
in [docs/help.md](docs/help.md).

- Human-readable tables by default; `--json` everywhere for scripting.
- Exit codes: **0** ok, **1** error, **2** usage, **3** permission/auth denied.
- Self-contained single-file binaries — no .NET runtime install required.

## Install

### npm (recommended)

```bash
npm install -g @commercient/dlake
dlake login --domain mycompany --api-key dlk_...
```

The package downloads the platform-matched binary on install and exposes it as
the `dlake` command.

### Direct download

Grab the binary for your platform from the latest
[Release](../../releases/latest) (or from
`https://downloads.datalake.commercient.com/downloads/dlake/<version>/<rid>/dlake[.exe]`),
verify it against `SHA256SUMS`, and put it on your `PATH`.

| Platform | Asset |
|---|---|
| Windows x64 | `dlake-win-x64.exe` |
| Linux x64 | `dlake-linux-x64` |
| Linux arm64 | `dlake-linux-arm64` |
| macOS Apple Silicon | `dlake-osx-arm64` |
| macOS Intel | `dlake-osx-x64` |

## Quickstart

```bash
# No account yet? Sign up from the terminal — no browser, no key needed.
dlake register start --email you@company.com --company "Acme Inc" --phone +15551234567
dlake register status --watch          # email verified -> provisioned -> seeded
```

Click the single verification link we e-mail you; everything after that is
automatic. When your lake is seeded, `register status` prints a **once-only,
7-day owner-admin API key** and saves it into your profile — copy it, it is
never shown again — and the same terminal can immediately drive both the data
plane and the control plane. A second e-mail carries your welcome message and
the tenant owner's temporary password.

```bash
# Authenticate once per tenant; profiles switch between tenants.
dlake login --domain mycompany --api-key dlk_...
dlake status                        # tenant, key, agent + service health

# API keys & projects
dlake keys list
dlake keys create --name ci-reader  # prints the raw key ONCE

# Query & export
dlake query "SELECT TOP 10 * FROM account" --json
dlake export account --format parquet --out ./account.parquet

# Object storage (S3 outlet)
dlake s3 connections list
dlake s3 ls sales
dlake s3 put sales ./q1.csv reports/
dlake s3 get sales reports/q1.csv ./local.csv
dlake s3 export sales account --format parquet   # server-side table → bucket

# Generic admin control plane (MCP passthrough)
dlake admin list                    # list every admin tool your key can use
dlake admin list_schemas            # call one directly — no `call` sub-verb
dlake admin create_table --help     # every tool self-documents its arguments
```

Run `dlake --help` or `dlake <command> --help` for the full surface.

## Reading related data

The tenant's Data API serves the same entities over **REST** and **GraphQL**, with the same key. For
anything relational, GraphQL is the one you want — nested reads across declared relationships, filters
that reach into a related entity, `contains` substring search, and `groupBy` aggregates for counts and
totals. The REST/OData surface has no `$count` and no `contains()`, so a search box or a pagination
total on REST alone will send you to raw SQL unnecessarily.

```bash
# Declare the relationship once (admin), then regenerate the engine for the whole model
dlake admin set_relationship --parentEntity Customers --childEntity Orders \
  --parentKeyColumn CustomerId --childFkColumn CustomerId --cardinality many
dlake admin restart_dab --confirm true
```

```graphql
# One request instead of a four-way join
{ orders(first: 20) {
    items { OrderNumber Total
            Customers { FullName City }
            OrderItems(first: 100) { items { ItemName LineTotal } } }
    hasNextPage endCursor } }

# A filtered total — the pagination count OData cannot give you
{ orders(filter: { Customers: { City: { eq: "Atlanta" } } }) {
    groupBy { aggregations { count(field: OrderId) } } } }
```

GraphQL's limits, past which the read-only SQL endpoints are the right answer: no `groupBy`/`orderBy`
on a *related* entity's column, no nested aggregations, and cursor paging only (no offset).

## Atomic multi-row writes

REST and GraphQL writes are **not** transactional, and two GraphQL mutations in one document are not
atomic either — a multi-step write can partially succeed. When you need all-or-nothing, put the work in
a stored procedure that wraps `BEGIN TRAN` / `COMMIT` / `ROLLBACK`, expose it, and call it as one
request:

```bash
dlake admin create_procedure --name usp_PlaceOrder --body @proc.sql --parameters @params.json
dlake admin set_entity_exposure --entity usp_PlaceOrder --expose true
dlake admin restart_dab --confirm true
```

It either commits everything or leaves the database untouched, and it returns the created rows.
Procedure entities are exposed on `POST` only — a write should not be reachable by a "safe" method.

## Argument conventions

Tool arguments (`dlake admin <tool>` / `dlake tool <tool>`) accept three forms:

- **Scalars** — `--table Invoice`, `--confirm true`.
- **Arrays and objects** — a value starting with `[` or `{` is sent as JSON
  (`--primaryKey '["Id"]'`); a plain comma list still works for arrays of
  scalars (`--primaryKey Id`). Malformed JSON is reported as a usage error
  naming the argument.
- **`@file`** — read the value from a file: `--columns @columns.json`. `@@`
  escapes a literal leading `@`. This is the reliable form on Windows, where
  quoted JSON is mangled by the shell before the program sees it.

```bash
dlake admin create_table --table Invoice --columns @columns.json --primaryKey Id
```

`dlake admin <tool> --help` documents these forms for every tool that takes an
array or object argument.

## Documentation

- **API usage guide** — endpoints, auth, scopes, rate limits: see the Data Lake
  \ Data Hub API guide served from your tenant's Help page.
- **Permissions** — object writes and connection management need
  `data.ingest.manage`; reads accept any `data.ingest.*` tier; scoped API keys
  are enforced server-side (fail-closed) down to entity and field level.

## Configuration

| Env var | Default | Purpose |
|---|---|---|
| `DLAKE_DOWNLOAD_BASE` | `https://downloads.datalake.commercient.com/downloads/dlake` | Binary mirror base (npm installs) |
| `DLAKE_VERSION` | npm package version | Pin a specific binary version (same or newer only) |
| `DLAKE_SHA256` | — | Operator-pinned expected digest (64 hex) for air-gapped installs |
| `DLAKE_ALLOW_MIRROR_CHECKSUMS` | off | Take `SHA256SUMS` from the mirror too (unsafe) |
| `DLAKE_ALLOW_DOWNGRADE` | off | Permit an older `DLAKE_VERSION` |

## Verifying downloads

Every release ships a `SHA256SUMS` file:

```bash
sha256sum -c SHA256SUMS --ignore-missing
```

The npm `postinstall` verifies automatically, and its integrity chain does not
follow the mirror: pointing `DLAKE_DOWNLOAD_BASE` at a private mirror moves the
**binary** only — `SHA256SUMS` is still fetched from
`downloads.datalake.commercient.com`, so a mirror can only serve bytes the
publisher already vouched for. Air-gapped installs pin `DLAKE_SHA256=<digest>`
(recommended) or opt into `DLAKE_ALLOW_MIRROR_CHECKSUMS=1`, which transfers full
trust to the mirror host and prints a warning. An older `DLAKE_VERSION` than the
installed package is refused unless `DLAKE_ALLOW_DOWNGRADE=1`.

## Questions & support

**For any questions, please email [support@commercient.com](mailto:support@commercient.com).**
That is the fastest way to reach us for help with the CLI, the platform, API
keys, connectors, or a Commercient SYNC integration. You can also open a GitHub
issue here for CLI bugs and feature requests.

Learn more about Commercient and the 150+ systems SYNC connects at
[www.commercient.com](https://www.commercient.com).

## License

The `dlake` binaries are proprietary software, free to use with a Commercient
Data Lake \ Data Hub subscription — see [LICENSE](LICENSE.txt).
