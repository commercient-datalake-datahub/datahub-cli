<p align="center">
  <img src="assets/logo.svg" alt="Commercient Data Lake / Data Hub" width="360" />
</p>

# dlake — Commercient Data Lake / Data Hub CLI

**`dlake` is the CLI for Commercient Data Lake / Data Hub — a backend-as-a-service
for Microsoft SQL Server: instant REST and GraphQL APIs, row-level security, and
MCP access for AI agents, over your ERP, CRM, and database data.**

> A place to aggregate all your company data from multiple systems so that AI
> agents of any type can see it all — and, if set up, even write data back.
> Code your own solutions against all your data, safely and securely. Let loose
> your staff's creativity on your data without risk.

`dlake` is the official cross-platform command-line client — in the spirit of
the Stripe and HubSpot CLIs. It authenticates with a tenant API key, manages
profiles, and wraps the platform's REST and admin surfaces with a
scripting-friendly UX.

> **Binary distribution.** This repository publishes the official `dlake`
> binaries and release notes. The source code is not published here.

## What is Commercient Data Lake / Data Hub?

A managed, per-tenant data platform **built on Microsoft SQL Server** that turns
your business data into an API- and AI-ready lake. Every tenant gets its own
managed SQL Server database — schema builder, Data API, events, time travel,
row-level security, and object-storage attach are all **native Microsoft SQL
Server** capabilities, not a bolt-on store. (PolyBase external-table attach and
vector features require SQL Server 2022+ / 2025.)

- **Bring data in** — through **Commercient SYNC**, Commercient Data Lake / Data
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
  an admin control plane with 126 tools), natural-language querying, row-change
  events (SSE / polling / signed webhooks), time travel, and row-level
  security.
- **Govern it** — role-based permissions, scoped API keys enforced down to
  entity and field level *in the database* (fail-closed), audit logs, and data
  quality rules with alerting.
- **Extend it to object storage** — link S3 buckets, browse/upload/download,
  export tables straight to a bucket, and (on SQL Server 2022+) attach parquet
  and CSV files as queryable external tables with automatic schema discovery.

`dlake` is the terminal/CI way to drive all of it. [docs/help.md](docs/help.md) is a
short overview; the full product help is in the console's Help section and from
`dlake guide help` once you are signed in (it fetches your tenant's live copy).

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
dlake register start --email you@company.com --company "Acme Inc" --phone +15551234567 \
  --consent-crm-backup --consent-erp-backup --consent-phone
# the backup flags confirm you have made YOUR OWN backups of your CRM/ERP data;
# --consent-phone consents to phone contact
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
dlake keys create --name ci-reader  # unscoped key; prints the raw key ONCE

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

### Scoped keys need a DAB restart

`dlake keys create` mints an **unscoped** (full-access) key. A key restricted to
specific entities is minted on the admin plane:

```bash
dlake admin create_api_key --keyName pizza-web --expirationDays 180 --scope @scope.json
dlake admin restart_dab --confirm true      # <-- REQUIRED, see below
```

> **Warning — after creating (or re-scoping) a *scoped* key, run
> `dlake admin restart_dab --confirm true`.** DAB's running configuration only
> gains the key's per-key role (`key_<id>`) when it is regenerated; until you
> restart, the key returns **`403 AuthorizationCheckFailed`** on every request
> even though the key, its scope, and the entities are all correct. Unscoped /
> full-access keys are unaffected. In the web UI the same save shows a
> **"Restart Needed"** notice and waits for you to press **Restart DAB** —
> restarts are always user-initiated. The restart must be driven by a
> **full-scope admin** key: a scoped key cannot reach the admin plane, so it can
> never restart DAB for itself. (Revoking a key is the one exception — it
> regenerates immediately.)

A scope entry is `{ "entityName": "<table/view/proc>" }` plus one or more action
booleans — `canRead`, `canCreate`, `canUpdate`, `canDelete`, `canExecute` — with
an optional `schemaName` for an external/lake schema (omit it for the working
schema). `entity` is rejected ("empty entityName"), and so is a bare
`read: true` ("no actions selected").

```json
[
  { "entityName": "pizza_menu_items", "canRead": true },
  { "entityName": "usp_PlaceOrder",   "canExecute": true }
]
```

### Build an app end-to-end

The full happy path, in the only order that works — schema, an atomic write, the
API surface, then a least-privilege key for the front end:

```bash
dlake login --domain <tenant> --api-key dlk_...          # admin, FULL-SCOPE key
dlake tool get_active_schema                             # which schema am I in?

dlake admin create_table --tableName pizza_menu_items \
  --columns @menu_items.json --primaryKey Sku
dlake admin create_procedure --procedureName usp_PlaceOrder \
  --parameters @params.json --body @proc.sql

dlake admin set_entity_exposure --entity pizza_menu_items --expose true
dlake admin set_entity_exposure --entity usp_PlaceOrder   --expose true
dlake admin restart_dab --confirm true                   # publishes the entities

dlake admin create_api_key --keyName pizza-web --expirationDays 180 --scope @scope.json
dlake admin restart_dab --confirm true                   # publishes the key's role
# then, from the app: POST /auth/<tenant>/api/usp_PlaceOrder
```

Two restarts, not one: the first publishes the **entities**, the second the new
key's **role**, and the key can only be minted after the entities exist.

Three things that bite on the way through:

- **The data plane sees only *exposed* entities.** `create_table` /
  `create_procedure` touch your database, not the Data API — `dlake tool
  create_record --entity pizza_menu_items` returns `EntityNotFound` until
  `set_entity_exposure --expose true` **and** `restart_dab --confirm true` have
  both run. Raw-SQL reads (`dlake query`, `dlake tool query`) don't need
  exposure.
- **The active schema is read-only from the CLI.** All DDL tools operate on the
  tenant's **active schema** (usually `DLO`). Read it with `dlake tool
  get_active_schema`; it is switched **only** from the web UI's schema dropdown —
  there is no `set_active_schema` tool and no `use` verb. Qualify raw SQL with it
  (`FROM DLO.pizza_menu_items`), and prefix object names per app to keep several
  apps tidy in one schema.
- **Stored-procedure parameter names must start with `@`.**
  `create_procedure --parameters` rejects `{"name":"CustomerName"}` with *"must
  start with '@'"* — use `{"name":"@CustomerName","dataType":"NVARCHAR","maxLength":100}`.

Two more small ones: **`ingest_table` needs a natural-key PK** — it refuses a
table whose primary key is an `IDENTITY` column (*"Row-by-row upsert needs a key
whose values come from the file"*), so seed identity-keyed tables with
`create_record` or give the table a natural key; and **avoid reserved T-SQL words
in column names** — a column called `LineNo` fails because `LINENO` is reserved
(likewise `Key`, `Order`, `User`, `Percent`).

## Atomic multi-row writes

Data API writes are **not** transactional — a multi-step write (a header, its line items, a status row)
can partially succeed and leave orphans. When you need all-or-nothing, put the work in a stored
procedure that wraps `BEGIN TRAN` / `COMMIT` / `ROLLBACK`, expose it, and call it as one request:

```bash
dlake admin create_procedure --procedureName usp_PlaceOrder --parameters @params.json --body @proc.sql
dlake admin set_entity_exposure --entity usp_PlaceOrder --expose true
dlake admin restart_dab --confirm true
```

It either commits everything or leaves the database untouched, and it returns the created rows.
Procedure entities are exposed on `POST` only — a write should not be reachable by a "safe" method.

## Argument conventions

Tool arguments (`dlake admin <tool>` / `dlake tool <tool>`) accept three forms:

- **Scalars** — `--tableName Invoice`, `--confirm true`.
- **Arrays and objects** — a value starting with `[` or `{` is sent as JSON
  (`--primaryKey '["Id"]'`); a plain comma list still works for arrays of
  scalars (`--primaryKey Id`). Malformed JSON is reported as a usage error
  naming the argument.
- **`@file`** — read the value from a file: `--columns @columns.json`. `@@`
  escapes a literal leading `@`. This is the reliable form on Windows, where
  quoted JSON is mangled by the shell before the program sees it.

```bash
dlake admin create_table --tableName Invoice --columns @columns.json --primaryKey Id
```

`dlake admin <tool> --help` documents these forms for every tool that takes an
array or object argument.

## Documentation

- **API usage guide** — endpoints, auth, scopes, rate limits: see the Data Lake
  \ Data Hub API guide served from your tenant's Help page.
- **AI-agent skill** — [`skills/dlake/SKILL.md`](skills/dlake/SKILL.md) is a
  drop-in skill that teaches a coding agent the correct command ordering and the
  sharp edges (the scoped-key DAB restart, exposed-vs-raw entities, IDENTITY-key
  limits). Install notes: [`skills/README.md`](skills/README.md).
- **Permissions** — object writes and connection management need
  `data.ingest.manage`; reads accept any `data.ingest.*` tier; scoped API keys
  are enforced server-side (fail-closed) down to entity and field level.

## Configuration

| Env var | Default | Purpose |
|---|---|---|
| `DLAKE_DOWNLOAD_BASE` | `https://datalake-ms-dab.commercient.com/downloads/dlake` | Binary mirror base (npm installs) |
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
`datalake-ms-dab.commercient.com`, so a mirror can only serve bytes the
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
Data Lake / Data Hub subscription — see [LICENSE](LICENSE.txt).
