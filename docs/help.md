> This document mirrors the in-app Help of Commercient Data Lake \ Data Hub.
> The live copy inside the product (and via the get_help_docs MCP tool) is always current.

# Commercient Data Lake \ Data Hub — Help

Condensed reference for every Help section. Each `##` heading is one section; call `getHelpDocs` with `section` to fetch just one. (The companion `getAPIGuide` tool returns the API Usage Guide — auth, REST/GraphQL, write semantics, exports, events, MCP — as markdown.)

## Getting Started

The Data Lake gives each tenant a managed SQL Server schema (default `DLO`) with a schema builder (tables, views, stored procedures, triggers, indexes, functions), a Data Browser, a SQL editor, and a per-tenant Data API (DAB) with API keys and MCP for AI agents. The sidebar's schema dropdown switches the active schema; the Data Engine badge shows the API container's live status. Most actions are permission-gated per role (Admin / User / ReadOnly).

## Dashboard

The landing page — an operational overview of what your data is *doing*, not object counts (those collapse to a single **Schema:** line: tables · views · procedures · triggers · indexes · functions · event triggers, each a link). Each card loads independently and shows a small "Couldn't load…" note on failure rather than blanking the page.

- **Health row** (shown to users with `data.ingest`): **Sync health (24h)** — imports succeeded / failed / in-flight (error accent if any failed); **Data freshness** — when the most recent sync landed and the *stalest* enabled entity (its last run, or "never run"); **Rows moved (24h)** — rows inserted / updated / deleted across all syncs; **Attention needed** — a rollup of connection warnings, recent Data Quality mismatches, webhook dead-letters, and agents with an update available ("All clear" when empty).
- **Detail row** (shown with `data.ingest`, `tables.view`, or `dab.view`): **Recent failures** — latest failed runs with entity/time/error, linking to Import; **Storage** — the tenant DB's data/log file fill (used / size / percent, "of max" when a max is set) plus the largest tables by reserved space (row count · MB), read from SQL Server file/partition stats (no full scans); **Usage** — request volume over 24h and 7d split into **API** calls and **DAB (incl. MCP)** calls — MCP tool traffic rides the same DAB containers and DAB metrics don't tag MCP separately, so the DAB figure inherently includes MCP (shows "Metrics unavailable" if the source can't be reached, not a misleading zero).
- **Quick Actions & Recent Activity** — permission-respecting shortcuts (create objects, manage users/permissions, Access Management, Browse Data, Settings) plus recent audit activity (or a workspace summary without audit-view access).

A tenant that hasn't configured any connectors sees empty health cards — expected, not an error.

## Tables

Create and manage tables in your schema. Columns support the usual SQL Server types, nullability, defaults, identity, unique, and primary keys. System tables (platform infrastructure like Users, ApiKeys, EventOutbox, DabConfigHistory) are protected — they can't be altered or dropped and are hidden behind the "Show System Tables" toggle.

**Computed columns**: a table can have computed columns (e.g. an `id` computed from business keys to make rows URL-addressable). Computed columns are read-only — never include them in API write bodies.

The Tables list shows a **Features** column with chips for what's enabled per table: CP (Concurrency Protection), Audit (Audit Stamping), Events (Event Capture), Log (Change Logging), CT (SQL Server Change Tracking), RLS (Row-Level Security).

**Applying schema changes to the Data API.** Creating, altering, or dropping tables, views, and stored procedures **no longer triggers an automatic DAB regenerate** — each success message says so. The change is saved to your schema immediately, but reaches the Data API only when you click **Regenerate / Restart DAB**. This lets you batch several schema edits into a single restart, which is the intended workflow.

## Views

Standard SQL views over your tables. Views are exposable through the Data API like tables (give a view an "addressable key" in per-entity settings to enable by-key reads/writes). View bodies are validated; refreshing views (sp_refreshview) runs automatically after column-affecting table changes. Creating, altering, or dropping a view does not auto-restart the engine — click **Regenerate / Restart DAB** to expose the change through the Data API.

## Stored Procedures

Create/edit/drop stored procedures in your schema. Procedures can be exposed through the API as execute-entities (`execute_entity` over MCP, POST over REST). System procedures are protected. Creating, altering, or dropping a procedure does not auto-restart the engine — click **Regenerate / Restart DAB** to apply it to the Data API.

## Triggers

User triggers on your tables (AFTER/INSTEAD OF, insert/update/delete). Bodies are validated against a blocklist (no DDL, EXEC, xp_*, cross-schema escapes). **System triggers** — audit triggers (`tr_<table>_Insert/Update/Delete`), platform feature triggers (`tr_<table>_dl_*`: AuditStamp, AuditFreeze, OptimisticConcurrency, BlockDirectDelete, Event), and change-log triggers (`trg_cl_*`) — are badged System and protected from non-admin modification; their name shapes are reserved.

## Indexes

Create/drop nonclustered indexes (optionally unique) on table columns to speed queries. Index names and columns are validated against the catalog.

## Functions

Scalar and table-valued user functions in your schema. Same protection model as the other objects (system functions protected, identifiers validated).

## Schemas

The **Schemas** page lists the database schemas in your tenant and lets you create or drop them. A schema is a namespace that groups objects (tables, views, procedures) — e.g. `DLO` (the Data Lake's own schema) and `dbo` (your gateway/ERP data).

- **List** — every schema and its owner. System schemas (`sys`, `INFORMATION_SCHEMA`, `guest`, and the fixed-role `db_*` schemas) are hidden.
- **Create** — add a new empty schema.
- **Drop** — remove an *empty* schema. SQL Server refuses to drop a schema that still owns objects (move or drop those first); schemas can't be renamed.

Creating or dropping a schema requires **both**: (1) the app permission `schemas.manage` (listing needs `schemas.view`) — both **off by default**, granted per role under Access Management (the tenant owner always has them); and (2) the tenant's underlying SQL login having rights to create/drop schemas — if it doesn't, the operation fails with the exact SQL error shown. `dbo`, `DLO`, and system schemas are **protected** — shown for reference but never droppable.

## Security & Access Control

Role-based permissions per tenant: **Admin** (everything), **User** (data read/write + view schema objects), **ReadOnly** (view + data read). The catalog is **56 granular permission keys** (e.g. `tables.create`, `data.read`, `sql.execute`, `audit.view`, `events.read`, `api-keys.manage`), each shown in Access Management with a **label and a description** of what it unlocks; keys are granted to roles, the UI hides what you can't do, and the API enforces the same keys server-side. Two gates worth calling out: the **table list and detail** require **`tables.view`**, and reading a table's **metadata** (columns/features) requires **`metadata.read`** — a permission added in this release and **backfilled to every role that already holds `data.read`**, so existing users keep their access. Sensitive operations (raw SQL, exports from the console) additionally require TOTP two-factor step-up. The tenant owner bypasses permission checks.

## Data Browser

Browse table/view data with paging, sorting, and a row-count. Reads honor Row-Level Security (you only see rows your identity may see) and — for API-key-derived sessions — the key's entity/field scope. Cell values are rendered read-only; use the Data API or SQL editor to modify data.

## SQL Editor

Raw T-SQL console (Admin + TOTP step-up required). Four output modes:
- **Run** — preview up to 10,000 rows (5-minute timeout). Larger results trip an abort panel routing you to CSV/Parquet/Save-to-Table.
- **CSV / Parquet** — full result set as a background export job; the browser polls progress and downloads when ready. Programmatic callers: the signed download URL waits ~45 s then streams (200), or returns **202 + Retry-After** while still running — re-fetch until 200. The download is **resumable** (HTTP Range → `206 Partial Content`), and short-budget/sandbox callers can add **`&wait=0`** to get an immediate 202 instead of the ~45 s park — poll across calls, then pull the file in `Range: bytes=<start>-` chunks. Over MCP, `export_table` also batches: pass `tables` (up to 10) for one job + one signed URL per table in a single call.
- **Save to Table** — writes the result into a new table in your schema (adds `dl_id BIGINT IDENTITY PK`). Multi-statement scripts and top-level ORDER BY without TOP aren't wrappable — use CSV/Parquet for those.

**Export safety & limits.** CSV exports **neutralize formula-leading cells** — a value beginning with `=`, `+`, `-`, or `@` is prefixed with an apostrophe so a spreadsheet can't execute it on open (Excel shows the value with that apostrophe stripped, so the data reads normally). Every export has a **hard cap of 50,000,000 rows** enforced *while streaming* — a run that would exceed it stops with a clear message rather than producing a truncated-looking file. Exports are also **rate-limited per user and globally**; over the limit you get a friendly **HTTP 429** asking you to retry shortly. For genuinely oversized results, **Save to Table** and then export a **filtered** slice.

The interactive SQL editor runs one query per user (Cancel kills it server-side). The programmatic `query` tool allows up to **10 concurrent queries per user** so analytics clients can fan out several reads at once; over the limit returns a 409. The limit is **per user** — it's shared across all API keys issued under that user (and the user's interactive sessions), not a per-key budget. Every Run/Export/Save-to-Table is audited (`SQL_EXECUTE`, `SQL_EXPORT`, `SQL_SAVE_TO_TABLE`).

Over MCP, two read-only SQL tools expose the same engine to agents: **`query`** runs a single SELECT and returns rows inline (10k cap); **`export_query`** exports an arbitrary SELECT (joins, computed expressions, aggregates) to CSV/Parquet via the signed-URL pipeline.

Both enforce SELECT-only (no writes/DDL/EXEC/multi-statement), require table names qualified with the tenant's **active schema** (often `DLO`, not always — the `get_active_schema` tool returns it), apply RLS, and are refused for scope-restricted API keys (`SQL_QUERY` / `SQL_EXPORT_QUERY` in the audit log) — **unless the key carries the `AllowRawSql` opt-in**. An opted-in scoped key's SQL is **database-enforced against its scope grid**: the key runs under its own SQL principal whose grants mirror the grid, so it can only read the entities/columns the grid allows (plus RLS on top). This is **fail-closed** — if the key's database principal hasn't been provisioned yet (or provisioning failed), the request is **denied with 403** rather than run with weaker enforcement; re-saving the key's scope re-provisions it. Note for keys with an **exclude-mode field policy**: SQL Server treats `*` as touching every column, so `SELECT *` and `COUNT(*)` on that table are denied — select or count explicit granted columns instead.

## Ask Your Data (natural-language query)

On the SQL Editor page, an **Ask your data** box turns a plain-English question into a **read-only** `SELECT`, runs it, and shows both the answer grid (first 200 rows; the query itself capped at 10,000) and the generated SQL with a short explanation. "Open in editor" drops the SQL into the SQL Editor for tweaking/export; it never auto-runs.

Same guardrails as everything else — read-only validation (a single `SELECT`/`WITH … SELECT`; anything that writes/EXECs is blocked before running), RLS under your identity, and scope-restricted keys only ever told about their own tables (and, like the raw SQL editor, refused unless the key carries the `AllowRawSql` opt-in — then its generated SQL is database-enforced against the key's scope grid, fail-closed). Requires `sql.execute` (Admin/Owner by default); each question is audited as `SQL_NL_QUERY`, and it does **not** trigger the SQL Editor's per-run TOTP prompt (you never hand-write the SQL).

It uses your organization's own Anthropic API key, set (encrypted, never shown back) under **Settings → AI / Natural-language query**; until a key is set the box shows a "not set up yet" panel.

**Definitions (business meanings).** Two optional layers teach the model — and MCP agents — what your data means; both are edited by admins with `dab.manage`:
- **Column meanings** — per-column business descriptions, set on an entity's gear under **DAB Config → Global Scope → Column Meanings** (e.g. `cust_stat` → "Churn status: A=active, C=churned"). Up to 400 chars each.
- **Business definitions** — cross-table terms/metrics (e.g. *revenue* = paid invoice total) with an optional advisory SQL hint, edited in the **AI / Natural-language query** card under Settings. Capped at 100 terms / 8 KB.

Definitions feed **both** the Ask box and MCP `describe_entities`: column meanings and entity descriptions become the field descriptions returned to AI agents (e.g. `"description": "Churn status: A=active, C=churned. NVARCHAR(1), NOT NULL"`). The Ask box picks them up immediately; the **MCP side updates on the next Restart DAB**.

**Export** downloads `nl-definitions.json` — a schema skeleton with your current definitions plus *blank* slots for every exposed entity/column — that an external AI can fill in; **Import** merges it back, validated: it reports how many terms/meanings/descriptions applied and **skips** anything hallucinated (an entity/column that doesn't exist) or over the length/count cap. A blank meaning never erases an existing one unless you explicitly replace.

## Data Ingestion (file → table sync)

The UI has an **Import Data** page (Schema Builder → Import Data) for no-code uploads — pick a file, choose or name a table, set the options, and see a live insert/update/delete summary. For automation and large files, use the API: load a CSV / Parquet / XML dataset into a table and keep it in sync via `POST /api/ddl/ingest` (multipart upload) or `…/ingest/from-content` (inline JSON).

The endpoint **creates the table if it's missing** — inferring column types from the data, with your `keyColumns` as the primary key — then upserts **matching on the table's primary key**: new rows are inserted, existing rows are updated **only when a value actually changed**, and unchanged rows are left alone. Set `mirror=true` for a **full-set sync** that also removes rows absent from the file — soft (`dl_deleted=1`, default) or hard (`deleteMode=hard` → DELETE). For files over 100 MB, stage a signed upload (`POST …/ingest/upload` → PUT the bytes → ingest with `uploadId`+`uploadToken`).

Everything runs in the **active schema** in one transaction with RLS enforced; it's audited as `DATA_INGEST`. Gated by the **`data.ingest`** permission (granted to the Admin role) and refused for scope-restricted API keys (an arbitrary file load can't be entity/field-scoped). Over MCP this is the **`ingest_table`** tool (file contents passed inline as `content` or `contentBase64`).

Ingest enforces **size caps** — on row count, column count, header length, individual cell size, and (for XML/Parquet) raw byte size — so a malformed or runaway file **fails fast with a clear message** naming the limit it hit instead of stalling the load. When a file introduces new columns, the **schema evolution is transactional**: the table's shape and the data land together, or neither does.

## Import (Connectors)

Pull data from an external source into your tables. Seven connectors ship today — **HubSpot**, **Stripe**, **Salesforce**, **ServiceTitan**, **Commercient**, **SQL Server**, and the **ODBC Agent** — behind a common seam (more plug in over time); each imports **every object the key can read** (discovered from the account, not a fixed list): HubSpot CRM objects (contacts, companies, deals, tickets, products, quotes, engagements, custom objects), Stripe resources (customers, charges, payment_intents, invoices, subscriptions, products, prices, balance_transactions), Salesforce SObjects (Account, Contact, Lead, Opportunity, Case, ... plus custom `__c` objects, with incremental via LastModifiedDate), ServiceTitan resources (customers, locations, contacts, jobs, projects, appointments, job types, invoices, payments, estimates, technicians, business units, employees, memberships — incremental via `modifiedOnOrAfter`), — for **Commercient** — the tenant's **own `dbo.SF_*` tables** synced locally over T-SQL (no external API; incremental via each table's `rowversion`/`TimeStamp`, server-side `MERGE` with no rows round-tripping through the app), — for **SQL Server** — **any SQL Server database this platform can reach on the network** (paste a connection string, stored encrypted; tables/views discovered from the catalog with **exact column types**; needs a single-column PK or an `id` column; a `rowversion` column enables cheap incremental), or — for the **ODBC Agent** — a **push source**: a small portable agent you install NEXT TO any ODBC-reachable database (Oracle, MySQL, Access, old ERPs) that pushes data **outbound over HTTPS**, so **no inbound access to the source network is needed** (see Agent setup below). It's the **Import** page under Schema Builder (`/import`) — distinct from **Import Data** (file upload).

Flow: (1) **Connect** — pick the source and paste its key: a HubSpot private-app token (`pat-…`), a Stripe restricted read-only key (`rk_live_…`/`rk_test_…`), or — for **Salesforce** — a Connected App via **JWT Bearer** (Consumer Key + username + certificate private key; recommended, no password stored) or the **deprecated** username-password flow (Consumer Key/Secret + username + password + security token; Salesforce disables it by default and is phasing it out), with the **Login URL** selecting production (`login.salesforce.com`) or a sandbox (`test.salesforce.com`) and the org instance learned automatically at sign-in; or — for **ServiceTitan** — the OAuth2 API application's four values (**Client ID**, **Client Secret**, **App Key**, **Tenant ID** — created in ServiceTitan under *Settings → Integrations → API Application Access*) plus an **environment** choice (production or the integration sandbox); tokens are short-lived and refreshed automatically, and tables land as `st_<entity>` (or `st_<middle>_<entity>`); or — for **Commercient** — **no key at all** (it's a local source that reads this tenant's own `dbo.SF_*` tables; set the **PTID** to the tenant slug); or — for **SQL Server** — the source **connection string** itself (use a read-only login; `TrustServerCertificate` defaults on for LAN servers); or — for the **ODBC Agent** — the source's ODBC connection entered as **key/value fields** (Driver, Server, Database, UID, PWD, plus driver-specific extras; a paste box fills the fields from a full string), stored **encrypted at rest** and fetched by the agent over TLS at startup — it never sits in a plaintext file and **local overrides are not supported** (source and target tables are two halves of the same connection record, so an agent can never be cross-wired to the wrong source); on Edit every setting is visible and alterable **except the password**, which never leaves the server (blank keeps the stored one).

**Agent setup** (ODBC connections): the connection card carries a panel to **download the agent** (portable self-contained exes — 64-bit + 32-bit for legacy ODBC drivers; no .NET install; zip has just the exes + README) and **download a ready-made config** — using the **Agent API key stored (encrypted) on the connection** with one click, or a pasted key (Settings → API Keys); it generates `appsettings.json` with this tenant's REST endpoint — taken from the URL you're on — slug, connection id, and PTID; the API key is written **encrypted**, and on its first run the agent re-protects it with **machine-bound Windows encryption (DPAPI)**, so the config file is useless if copied to another machine.

There's also **Create setup link** (requires the stored key): a **single-use, expiring URL** you send to the customer's IT person — no Data Lake login needed; it shows a confirm page (so mail scanners can't consume it) and downloads **one bundle** (both exes + README + the ready-made config). The link works exactly once, defaults to a 7-day expiry, and unused links can be revoked.

Once an agent has checked in, the panel also shows its **telemetry** — installed **version** (against the latest published, highlighted when behind), **environment** (machine, OS, bitness), **IP address**, and last seen — and an **Update agent** button: it flags the connection, the agent sees the flag on its next check-in, downloads the published package, **swaps its own exe and exits**, and the next scheduled firing runs the new build (the flag clears itself when the agent reports the new version).

**Key requirement**: the agent's API key must carry the dedicated **DataLake Agent** permission (`agent.sync`, Access Management — off by default) or be an owner key — fetching the server-stored source connection string requires it, and since the string lives only server-side, a plain admin (`data.ingest`) key is **not** enough to run the agent.

Best practice: a dedicated agent user granted only `agent.sync` — its key runs the whole push flow and nothing else. Easiest install path: **Create setup link**, open it in a browser **on the source server**, save and unzip the bundle, then **double-click `DataLakeAgent.exe`** (or `DataLakeAgent-x86.exe` on a 32-bit server) and approve the Windows elevation (UAC) prompt — the bundle's `appsettings.json` is already configured, so there's nothing to edit, and double-clicking offers to install the scheduled task for you. That task is a Windows Scheduled Task named **`Commercient Data Lake \ Data Hub <slug> <ptid>`** firing every 5 minutes (one tick: check for a pushed update → report catalog → claim due work → push → exit; below-normal priority, ~half a core only while actively pushing, zero footprint between firings; `uninstall-task` + delete the folder removes everything). For a headless/scripted install, run `DataLakeAgent.exe install-task 5` from an elevated prompt instead. **Multiple agents share one server** — one folder per connection, each with its own config, task name, and single-instance guard, so agents for different connections run in parallel.

Within a minute of the first run the agent's tables appear in the entity list (system/catalog schemas — `sys`, `INFORMATION_SCHEMA` and friends — are never offered). Entities sync on the **key the driver reports** — single or composite — and the key columns **keep their real names**: the target gets the same primary key as the source. When none of the synced columns is named `id`, the target also gets a **computed, uniquely-indexed `id` column** holding the row's URL-ready API extension (`KeyName/value` — or `K1/v1/K2/v2` for composite keys), the same convention CRM Pro tables use, so every row stays addressable by a single id. Keyless views need a column named `id`.

Each agent entity carries a **Limits &amp; watermark** link under its target table (it summarizes state once set, e.g. "Limits: cutoff · wm: ModifiedDate") opening a popup with: a **data cutoff (WHERE)** — appended to the source read as `AND (…)` in the **source database's SQL dialect**, e.g. `InvoiceDate >= '2024-01-01'`, so old history is never pulled (rows already synced that later fall outside the filter are only removed by a Full-mirror run) — and a **Watermark column** picked from the entity's actual columns (a modified-date or ever-increasing numeric) that enables cheap incremental; blank = incremental re-reads everything each run (still correct — the upsert only writes real changes).

**Moving or restoring databases.** The per-table sync cursor (change-tracking version or incremental watermark) lives **in the Data Lake tenant database** alongside the data it tracks, so it travels with a move/detach and a backup/restore: restore a 2-day-old Data Lake backup and each table's cursor rolls back 2 days too, and the agent automatically **re-pulls that window** on its next run (re-ingesting is idempotent, so nothing is duplicated). This is authoritative — the agent follows the Data Lake's cursor even when it moves **backward**, so a restore can never leave a table silently short. Two safety cases fall back to a **full re-baseline** (a one-off complete re-pull) instead of resuming: the restore reached further back than the **source's** change-tracking retention window, or the **source** database itself was restored to an older point than the cursor. Change-tracking sources with **column tracking** enabled also skip re-pulling rows that changed only on columns this entity doesn't sync (a note records when this suppression is active); the cursor still advances, and a later change to a synced column pulls the row normally.

For agent connections, **Run/Resync queue the request** and the entity shows a live **"sent to agent — waiting for its next poll…"** status until the agent claims it (typically within its poll interval), then the normal phase/row-count progress takes over — the panel refreshes itself, including for scheduled and agent-initiated runs; Test shows when the agent last checked in.

Secrets are **encrypted at rest** in the tenant DB, never returned, with a **Test** that confirms it authenticates (or, for Commercient, that the local DB is reachable) and counts readable objects. Two optional free-text fields: a **table-name middle term** namespaces this connection's tables (blank → a provider's connections share tables `<prefix>_<entity>`, e.g. `stripe_customers`; set → separate `<prefix>_<middle>_<entity>`, e.g. `hs_148779116_contacts`) and is **fixed at creation** (changing it would orphan the tables already created); and a **Provider Tenant ID (PTID)** is just an editable label for the source account (metadata, does not affect table names).

Every connection has a **Write mode**: **Master** (default — the source wins; local edits to synced columns are overwritten when the source row changes, and the sync safely pauses concurrency-protection triggers inside its merge transaction) or **Editor** (the connector behaves like any other editor: rows edited in the Data Lake since their last sync keep the Lake's version, the skipped source changes are reported on the run as amber notes — not failures — and deletes are always soft; per-row sync anchors are seeded when you switch, so the first editor run doesn't false-conflict). Use Editor for bi-directional flows where both the source system and the Data Lake accept edits.

**Conflicts console.** Editor-mode conflicts are surfaced in the UI, not only in run notes. Each connection's Import page has a **Conflicts** section, and a global **Sync Conflicts** page (linked in the sidebar with an unresolved-count badge) groups every connection's conflicts — including file-editor ingests, which group under **File uploads**. Each conflict carries a kind badge (**source change skipped** or **source delete skipped**), a side-by-side **Lake (kept)** vs **Source (skipped)** view of the row, and a **Resolve** button. Resolving a conflict **acknowledges the current source value** — that same difference won't re-notify on later runs, and it re-raises only if the source value changes again.

Under the hood these conflicts are **persisted and deduped** (DLO.ImportConflicts): a standing conflict re-detected by a no-watermark full re-stage only bumps its seen-counter — run notes and warning **emails fire only for new conflicts or ones whose source value changed**; conflicts auto-resolve when values converge, are queryable (Lake values vs skipped source values, per row) via `GET /api/ddl/import/connections/{id}/conflicts`, and acknowledgeable via its `/resolve` action. Tables with **Change Logging** enabled also record each skipped source change as a special `dlo_op='C'` entry in the shadow table (and in Master mode, ordinary change logging captures every overwritten value — enable it for overwrite forensics).

Connections can be **edited** (display name, PTID, region, write mode, error-notification emails — and for ODBC connections the source's key/value settings and the stored Agent API key) and the **key rotated** (paste a new one, or leave blank to keep the current) without losing entity selections or run history — via the card's Edit button or `PUT /api/ddl/import/connections/{id}`.

A connection can also carry **notification emails** (comma/semicolon-separated): those addresses are emailed whenever a run for the connection **fails** (the error) or **completes with data warnings** (the run's notes — e.g. values that couldn't be parsed as the column's type and were stored as NULL, or a new source column that was auto-added); clean runs send nothing. Subjects and bodies are stamped with the **tenant** so operators watching several accounts can tell sources apart.

(2) **Select entities** — per object set **mode** (`incremental` default = only new/changed records since the last run, tracked by a watermark — HubSpot by last-modified date, Commercient by each table's `rowversion`/`TimeStamp` column, Stripe via the **Events API** so it catches updates too — `invoiceitems`, like `invoices`, now syncs incrementally through this same events feed, so its updates and deletes are captured (needs the Stripe key's **Events Read** permission — without it, incremental gracefully falls back to inserts-only and the connection panel shows an **orange warning** that updates aren't being captured (the warning clears automatically once a run succeeds with Events Read granted); events retain ~30 days, so idle-longer connections should run a Full mirror to catch up); the watermark keeps a small safety overlap so late/in-flight records aren't missed — the overlap is **per-connection editable** (Watermark overlap, in seconds, on the connection form; blank = the platform default of 300s) for sources with more clock drift / eventual-consistency lag; `full` = mirror, removing rows no longer present), **target table** (defaults from prefix + PTID), and an optional **property** subset (all describable by default — Stripe, having no schema API, seeds its field list from common fields plus a sampled record); **Save**. The list is auto-discovered, but you can also **Add entity** by name (an input on the Entities panel) for anything discovery doesn't surface — an extra Stripe resource (e.g. `payouts`, `refunds`) or a HubSpot object/custom object the key can read but can't enumerate; it **probes** that the key can actually read that name (clear error if not), then drops it in as an unsaved row to configure + Save like any other.

(3) **Run / Run All / Resync** — background syncs that **stream** pages from the source through the ingest pipeline without buffering the whole pull (Run = configured mode; **Resync** = one-off full re-pull that ignores the incremental watermark — to rebuild a table or catch up after Stripe's ~30-day event window, after which incremental resumes from the newest record); the table is **created if missing** and rows are **upserted on `id`** (full mode also deletes).

Column types are **schema-driven** where the provider declares them (HubSpot/Salesforce booleans → BIT, numbers → DECIMAL; strings → `NVARCHAR(MAX)` so a longer value in a later batch can't overflow) and inferred from values otherwise; if the source later **grows a new property**, the column is **added to the table automatically** (nullable) rather than dropped, with a note on the run. Unparseable **cell values are lenient** — a non-key value that isn't valid for its column's type becomes NULL instead of failing the run, counted per column and shown as an **amber warning** on the run (and emailed, see notification emails above). Stripe's `hosted_invoice_url`/`invoice_pdf` are deliberately **never imported** (Stripe rotates the signed token in those URLs on every fetch — they'd make every invoice look changed each run).

**Commercient** runs entirely **server-side** — a single T-SQL `MERGE` from `dbo.SF_*` into the schema, so no rows leave the database; the target defaults to **`cm_<object>`** (the `SF_ERP_<erp>_[Clone_]` boilerplate stripped, e.g. `dbo.SF_ERP_Salesforce_Clone_ArCustomer+` → `cm_ArCustomer+`), and unusual names (e.g. a trailing `+`) are handled end-to-end. The source's `TimeStamp` rowversion drives incremental but is **not copied**; the target gets its **own auto-managed `TimeStamp` rowversion** (never written), like other tables.

A live **Run history** shows the **hub**, rows pulled, insert/update/delete counts, and the full error on a failure.

While a run is in flight the per-entity status shows live **progress** — the current phase (queued -> starting -> reading -> staging -> merging; **queued** means it's waiting for a run slot, not stuck), records retrieved so far, and elapsed seconds (no percentage: streaming pulls and Stripe's APIs expose no upstream total, so records-retrieved is the honest signal). The first incremental run is a full pull; later runs fetch only the delta. An entity with **no records** in the source completes as a clean no-op (0 rows; the table isn't created until there's data) — not a failure, so empty/scheduled objects don't pile up errors.

Concurrent runs are **bounded two ways** — platform-wide (default 4) and **per connection** (default 3), so one connection's big resync queues on itself and can't starve other connections' syncs; over a cap, runs show **queued** and start as slots free. Long pulls hold **no database locks** (only the final brief merge is transactional), so a slow sync never blocks reads of the target table. Run history is **kept bounded automatically** (newest ~500 runs per connection, configurable) and a **Clear history** action purges it on demand (an in-progress run is preserved).

(4) **Schedule** — each entity has a Schedule dropdown (Manual only / every 15 min / hourly / every 6 hours / daily); pick an interval and Save, and a background runner syncs it automatically on that cadence (first scheduled run starts within a couple of minutes), using the same mode/table and appearing in Run history alongside manual runs. Gated by **`data.ingest`** (Admin role). Deleting a connection removes its selections + run history but keeps the imported tables. Managed via REST under `/api/ddl/import/*`; **not exposed as an MCP tool**.

**Re-discover** (button on the Entities panel) re-probes the source with the connection's *current* credentials and refreshes the entity list — for **every connector**. Use it when the source gains objects the connection can now see (a new source table, a new HubSpot/Salesforce object). **Stripe** discovery probes Stripe's *whole* listable catalog and keeps only the resources the key can actually read (unreadable ones are hidden, not errored), so elevating the Stripe key's permissions (e.g. credit-notes, `billing/credit_grants`) and re-discovering makes those resources appear.

**Sync modes — what each writes.** Each entity's **Mode** is one of: **Incremental** (default — inserts + updates records changed since the last run, tracked by a watermark; fast/light); **Full mirror** (re-pulls everything and makes the table an exact copy of the source, removing rows no longer present); or **Change tracking** (a cheap incremental that *also* captures deletes by reading SQL Server's built-in Change Tracking on the source table). Incremental's **delete** behavior depends on the provider: **Stripe** deletes (via `*.deleted` events — see below); the **ODBC Agent** deletes only when the entity runs in **Change tracking** mode; **every other provider** never deletes in incremental — use **Full mirror** to purge source-removed rows.

**Stripe deletes (incremental).** Stripe is the only provider whose Incremental mode removes rows: its Events API emits `*.deleted` events (e.g. `customer.deleted`), so an incremental run collects those ids and, after merging inserts/updates, removes the matching rows by key — a real `DELETE`, or a soft `dl_deleted = 1` when the target table carries that column (e.g. an Editor-mode connection). This needs the Stripe key's **Events Read** permission (the same one incremental updates rely on). A Full mirror run reconciles removals via its whole-table diff and needs no delete events.

**Change tracking mode (SQL Server agent sources).** Offered — in the Sync-modes reference and the per-entity Mode dropdown — **only** on an **ODBC Agent** connection whose stored source string is a *native* SQL Server connection string (no `Driver=`/`Dsn=` key); it is hidden for every other connector (Stripe, HubSpot, Salesforce, SQL Server pull, Commercient) and for agent connections whose source string is a classic ODBC/DSN string. CT mode preconditions error **only for CT-mode entities** (an entity's Incremental or Full-mirror siblings on the same connection are unaffected) with an actionable message when: the source isn't a SQL Server agent source; the source string is ODBC/DSN rather than native ("Change tracking mode requires a Microsoft SQL Server source (native provider)… Store a native SQL Server connection string (no Driver=/Dsn= key)…"); or Change Tracking isn't enabled on the customer's source. The last is the customer's own action — the run reports the exact T-SQL: at the DB level `ALTER DATABASE … SET CHANGE_TRACKING = ON (CHANGE_RETENTION = 7 DAYS, AUTO_CLEANUP = ON)` and per table `ALTER TABLE … ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON)` (the table must have a primary key). If the agent is offline longer than the retention window its saved cursor ages out, so the next run auto-re-baselines with a full pull (reconciling deletes missed during the gap) and resumes cheap CT from there.

**Row-count checks (destination parity).** Every completed run records the destination table's **live row count** (excluding soft-deleted rows) — shown as a **Dest rows** column in Run history. **Full mirror** runs also run a **parity check** comparing source vs destination counts with the run's read-cutoff and entity filter applied to both sides (apples-to-apples); a mismatch doesn't fail the run but surfaces as a **warning** note ("source has N rows but destination has M … this mirror may be incomplete"), a `⚠ mismatch` badge in Run history, and an email if the connection has Email-on-warning on. **HTTP sources** (Stripe/HubSpot/Salesforce) have no cheap `SELECT COUNT(*)`, so their runs skip the comparison and note "source row count is unavailable" (the destination count is still recorded); the parity check is most useful on SQL Server / ODBC-agent full-mirror runs.

**Verify (integrity check).** Each connector-pull entity has a **Verify** button that runs a **read-only** full-snapshot comparison of the Lake copy against the live source — it answers "is the Lake a perfect mirror of the source right now?" without writing anything. The report buckets the differences: **matched**, **missing in Lake** (in the source, not the target), **extra in Lake** (in the target, not the source), and **value mismatches** (same key, different values). Differences that are *expected* are counted separately so they don't read as corruption: soft-delete tombstones (rows the mirror removed) and, in Editor mode, local edits that intentionally kept the Lake's value. Verify and sync can't run at the same time on one entity, and the result lands in Run history as a `verify` run. Verify is available on connector-pull connections; agent-push (ODBC) connections aren't supported yet. REST: `POST /api/ddl/import/connections/{id}/entities/{entity}/verify`; over the admin control plane it's the `verify_entity_integrity` tool.

## DAB & API Keys

The Data Engine (DAB) is a per-tenant container serving REST + GraphQL over the entities an admin exposes (Global Scope tab). Key points:

- **Per-entity settings** (gear icon): addressable key for views, column aliases, GraphQL relationships, custom REST path/methods, GraphQL include/exclude, response cache TTL.
- **Computed columns & URL-ready keys**: computed `id` columns make rows addressable as `/api/<entity>/id/<value>`; computed columns are excluded from write bodies automatically in the config.
- **Writing data**: POST=insert, PATCH=partial upsert (preferred), PUT=full-replace upsert; never send `id`/`TimeStamp`/`dl_*`; CP tables need `dl_expected_ts` echoed on UPDATE; PUT/PATCH need create+update permissions.
- **API keys**: generated per user, optionally scoped per entity/action/field. Exchange `dlk_` key → JWT at the Auth API. A key **inherits every right the owning user has** and is then narrowed by a **per-key denylist** — open the key's **rights panel** (Settings → API Keys → the key) and **uncheck** any capability you don't want that key to carry (the effective right is `min(user, key)`, recomputed from the live user whenever the key is minted or refreshed, so removing a right from the user removes it from the key too). **Owner** (`is_owner`, bypasses all permission checks) and **DBO Admin** (`is_dbo_admin`, dbo-schema admin) are shown separately and are **individually deniable per key** — issue a key that can't act as owner even though its user is one. Revoking a key kills its live MCP sessions, flushes the proxy JWT cache, and is enforced at token validation so a revoked key stops working **within ~30 seconds**. Inactive (revoked/expired) keys can be soft-deleted from the list. Each key card also has an **Extend** menu (+7 / +30 / +90 days / +6 months) that pushes the expiry out — new expiry is `max(now, current) + interval`, so extending a live key never shortens it and an expired (but not revoked) key revives from now; a revoked key can't be extended. A revoked key shows a **Restore** action that un-revokes it — this clears the revoked flag *only* and does **not** change the expiry, so a key that was also past its expiry comes back still-expired and prompts you to Extend it before it works again. When an **admin** clicks **Generate Key**, an **Owner** picker lets them mint the key on behalf of another user (the key then acts as that user); non-admins can only create keys for themselves, and even an admin can't target a more-privileged account (only an owner can target an owner, only a DBO Admin a DBO Admin). **Projects**: keys can be organized into projects (a first-class grouping entity, with future per-project billing / cost-limits in mind). The **Generate Key** dialog has a **Project** field — a typeahead over existing projects where you can also **type a new name to create one on the fly**; leave it blank for no project. The **API Keys** page **groups key rows under a project header** ("No project" for ungrouped keys), with a **project filter** dropdown at the top, and each key's action menu has a **Move** action that files it under a project (get-or-create by name, case-insensitive) or clears it. Projects are organizational only — they are **not** a rights axis and never change what a key can do.
- **MCP connector URLs**: an AI client connects to the data plane at the per-tenant `https://datalake-ms-dab.commercient.com/auth/<tenant>/mcp`, or at the **generic** tenant-less `https://datalake-ms-dab.commercient.com/mcp`. The generic URL carries no tenant in the path — it resolves the tenant from the credential, in order: the OAuth JWT's tenant (bound when the user enters their domain on the authorize page), a **composite API key** `tenant:dlk_...` (in `X-API-Key`, or `Authorization: Bearer/ApiKey`), or an `X-Tenant: <slug>` header sent with a plain `dlk_` key. Everything downstream is identical either way (same tools, port lookup, lockout, token cache) — the only difference is who names the tenant.
  - *Which to use*: the **generic** URL is the one-connector-definition-for-any-customer option — hand out a single URL and the key identifies the tenant. Use a **per-tenant** URL when the tenant is fixed, or — the headline case — when **one Claude/MCP client needs several Data Lake tenants connected at the same time**: clients like Claude Desktop / claude.ai dedupe connectors by normalized URL, so each tenant needs its own distinct URL (`…/auth/acme/mcp`, `…/auth/globex/mcp`, …) to live as a separate connector. With OAuth the per-tenant URL also pre-selects that tenant on the authorize page. Suggested connector names: **Commercient Data Lake \ Data Hub** (data) and **Commercient Data Lake \ Data Hub Admin** (admin). A stuck/"expired" connector record can be worked around with a URL variant like `…?client=v2`.
- **MCP sessions** (admin): every OAuth-connected agent session is listed and can be terminated.
- **Config history**: every engine config version is recorded (who/when/what); any version can be viewed, diffed, and reverted to.
- **Engine management**: Restart DAB rebuilds the config and redeploys the container; the Resources tab shows live container CPU/memory/network stats. Schema DDL (creating/altering/dropping tables, views, procedures) never auto-restarts the engine — batch your changes and click **Restart DAB** when ready.

## Admin Control Plane (MCP)

A **second, separate MCP connection** for administering a tenant's data lake from an AI client — it *configures* the lake (setup, not data transactions) and never reads or writes table data (that's the data plane). Connect it exactly like the data-plane MCP — same OAuth flow, same tenant host — but on the `/admin` path: `https://datalake-ms-dab.commercient.com/auth/<tenant>/admin` (vs the data plane's `/auth/<tenant>/mcp`). A generic **tenant-less** URL `https://datalake-ms-dab.commercient.com/admin` also works — it resolves the tenant from the credential (OAuth JWT, a composite `tenant:dlk_...` API key, or an `X-Tenant` header) exactly like the generic data connector described under **DAB & API Keys**. Same guidance applies: use the generic URL to share one admin connector definition, or distinct per-tenant `/auth/<tenant>/admin` URLs to administer several tenants at once from one client (clients dedupe by URL). It is **stateless** JSON-RPC 2.0 (no session id), unlike the session-based data plane.

**Access is strict:** the API key must be **full-scope** *and* belong to an **Admin** user. A per-entity-scoped key, a non-admin full-scope key, and an interactive (non-key) login are all rejected with HTTP 403 + JSON-RPC error `-32001` naming the requirement. Issue a dedicated full-scope key on an admin user for this.

Its 76 setup tools:
- **Entity exposure / DAB** — `list_exposed_entities`, `set_entity_exposure`, `get_entity_settings`, `set_entity_settings` (per-entity settings incl. column meanings), `restart_dab`, `dab_status`.
- **NL definitions** — `get_nl_definitions`, `put_nl_definitions`, `export_nl_definitions`, `import_nl_definitions` (export/import round-trips a schema skeleton an external AI can fill).
- **Import / sync setup** — `list_connections` (secrets never returned), `list_source_entities` (live source re-discovery), `get_entity_sync_config`, `save_entity_sync_config` (validates CT-mode preconditions), `run_entity_sync`, `get_run_history` (runs incl. destination row counts + notes), `verify_entity_integrity` (read-only full-snapshot source-vs-Lake diff for one connector-pull entity — no writes).
- **Status** — `get_schema_status` (this tenant), `get_data_quality_summary`.
- **Data-quality rules** — `list_dq_rules`, `set_dq_rule` (upsert), `delete_dq_rule`. Rule types: `min_row_count` (`{"minRows":N}`), `not_null` (`{"columns":["a","b"]}`), `freshness_max_age_hours` (`{"timestampColumn":"ModifiedUtc","maxAgeHours":24}`), `null_rate` (`{"column":"email","thresholdPct":5}`), `row_count_drift` (`{"thresholdPct":20}`), and `row_count_parity` (`{"tolerancePct":1}` and/or `{"toleranceRows":10}`). `row_count_parity` tunes the full-mirror source-vs-destination check that runs *during* a sync: set `enabled:false` to silence its warning for the entity, or a tolerance to allow small drift; with no rule row present, parity behaves exactly as before. All other types are also evaluated on the platform's data-quality schedule.
- **GraphQL relationships** — `list_relationships`, `set_relationship` (upsert; **validates both entities and both columns exist** in the tenant schema — a bad name is rejected so it can't crash-loop DAB), `delete_relationship`. A declared parent/child link nests the child under the parent (cardinality `many` by default) and the parent under the child (`one`) in the Data API's GraphQL. Changes are **not** auto-applied — they take effect on the next Regenerate / Restart DAB.
- **Events (row-change capture)** — `list_event_captures`, `enable_event_capture` (installs the capture trigger + column map; idempotent), `disable_event_capture`, `refresh_event_capture` (rebuild the column map/trigger after raw-SQL column changes).
- **Webhooks** — `list_webhook_subscriptions` (tenant-wide; secrets never returned), `create_webhook_subscription` (URL is SSRF-checked before storage; **returns the signing secret exactly once — store it, it can never be retrieved again**), `update_webhook_subscription` (URL re-checked; `enabled:false` pauses delivery), `delete_webhook_subscription`, `get_webhook_delivery_status` (cursor, failure streak, dead-letters).
- **Time Travel / change tracking / change logging** — `get_table_history_status` (one-call posture for all three), `enable_time_travel` (needs a PK; mutually exclusive with Concurrency Protection), `disable_time_travel` (`dropHistory:true` also drops the history table), `query_row_history` (one row's full version timeline by PK, RLS-scoped, **capped at 200 versions**, default 50), `enable_change_tracking` / `disable_change_tracking` (omit `table` for the database level), `enable_change_logging` / `disable_change_logging` (shadow table + trigger; `dropShadow:true` also drops the shadow data).
- **RLS / permissions / roles** — `list_rls_policies` (read-only — creating/dropping policies stays in UI/REST), `get_permission_matrix`, `grant_permission` / `revoke_permission` (target a role *or* one user as an override), `list_roles`.
- **Docs** — `get_help_docs` (optional `section`), `get_api_guide` — return this Help / the API guide as markdown (mirror of the data plane's `getHelpDocs`/`getAPIGuide`).
- **API keys** — `list_api_keys` (metadata only, no hashes; each row includes its **project**, and an optional `project` arg filters to one project), `revoke_api_key`, `create_api_key` (mints for the calling admin by default, or pass `targetUserId` to own it by another user — admin-only, same escalation guard as the UI; pass `project` to file the key under a project — get-or-create by name, case-insensitive, grouping only; `expirationDays` must be 7/30/90/180; **returns the raw key exactly once — only its hash is stored**), `extend_api_key` (push a key's expiry out by 7/30/90/180 days — `max(now, current) + days`; refuses a revoked key), `unrevoke_api_key` (restore a revoked key; leaves the expiry untouched, so a still-expired key must then be extended), `get_key_rights`, `set_key_rights` (denylist model — removals only, applied at each exchange/refresh), `set_key_entity_scopes` (empty scope = full access; a non-empty scope also locks the key out of this admin plane).
- **Users** — `list_users` (read-only, no hashes), `activate_user`, `deactivate_user` (the tenant owner and your own account are refused).
- **On-prem agent** — `get_agent_status` (last seen, version, streaming diagnostics), `trigger_agent_update` (`cancel:true` withdraws a pending push).
- **Export jobs** — `list_export_jobs`, `cancel_export_job` (this tenant's running jobs only).
- **Audit** — `query_audit_log` (every DDL/admin op incl. `mcp-admin:*`; filter by operation/target/user; **capped at 200 rows**, default 50).
- **Notifications** — `list_notifications` (the calling user's in-app notifications + unread count).
- **Schema sweep** — `run_schema_sweep` (runs the internal schema-migration self-heal against **this tenant's own database only**; `includeManual:true` also applies operator-gated migrations).
- **Object storage (S3 outlet)** — `list_s3_connections`, `create_s3_connection` (secret write-only, redacted in the result), `test_s3_connection`, `delete_s3_connection`; `s3_list_objects` (continuation-token paging), `s3_upload` / `s3_download` (base64, **hard 8 MB cap** via `S3:McpUploadMaxBytes` — larger files go through `dlake s3 put`/`get` or the UI), `s3_delete_object`; and `export_to_s3` — export a whole table/view straight into a connection's bucket, reusing the scoped export-table pipeline (per-key scope, RLS, impersonation, 50M-row cap) and waiting up to `S3:ExportToS3WaitSeconds` (default 300s) for completion. Reads use the `data.ingest.*` view tier; writes (create/delete/upload/delete-object/export) require `data.ingest.manage`.

**Destructive tools** — `restart_dab`, `revoke_api_key`, `delete_dq_rule`, `delete_relationship`, removing an entity via `set_entity_exposure`, `import_nl_definitions` with `mode:"replace"`, `disable_event_capture`, `delete_webhook_subscription`, `disable_time_travel`, `disable_change_tracking`, `disable_change_logging`, `deactivate_user`, `delete_s3_connection`, `s3_delete_object`, and `run_schema_sweep` — require `confirm: true`. Config changes to exposure/settings/meanings/relationships persist immediately but reach the live Data API only after `restart_dab`. Revoking a key takes effect immediately and flushes the tenant's cached MCP proxy tokens (an already-issued proxy token may otherwise linger to its ~55-minute cache window). `create_api_key` and `create_webhook_subscription` each return their secret **once** in the tool result — capture it immediately. Every admin write is audited as `mcp-admin:<tool>`.

> **After this update, restart your MCP client.** An MCP client caches the tool list per session (it pulls the schema once at connect), so Claude Desktop / claude.ai won't show the new tools until you restart the client or re-add the connector.

## Command-Line Interface (dlake)

`dlake` is the Data Lake's command-line client (in the spirit of the Stripe / HubSpot CLIs) — script your tenant from a terminal or CI with an API key. The **CLI** entry in the left menu opens the full guide with every command.

**Install** — download the self-contained binary for your platform (no runtime needed) and put it on your `PATH`:

- [Windows (win-x64)](https://downloads.datalake.commercient.com/downloads/dlake/0.2.1/win-x64/dlake.exe)
- [Linux x64](https://downloads.datalake.commercient.com/downloads/dlake/0.2.1/linux-x64/dlake)
- [macOS Apple Silicon (osx-arm64)](https://downloads.datalake.commercient.com/downloads/dlake/0.2.1/osx-arm64/dlake)
- [SHA256 checksums](https://downloads.datalake.commercient.com/downloads/dlake/0.2.1/SHA256SUMS) · or `npm install -g @commercient/dlake`

**Sign in once per tenant** with an API key (generate one under Settings → API Keys), then work:

```bash
dlake login --domain yourcompany --api-key dlk_...   # stores an encrypted profile (short tenant slug)
dlake keys list                                   # your keys, grouped state incl. project
dlake query "SELECT TOP 10 * FROM DLO.MyTable"    # read-only SQL (RLS-scoped)
dlake export MyTable --format parquet -o my.parquet
dlake s3 put sales ./my.parquet exports/       # stream a file to an S3 bucket
dlake s3 export sales MyTable --format parquet # export a table straight to S3
dlake admin list                                  # every admin-plane tool, driven generically
dlake admin dab_status
```

Multiple tenants = multiple **profiles** (`--profile`), like the Stripe CLI's projects. Everything supports `--json` for scripting. The `dlake admin <tool>` passthrough exposes the full Admin Control Plane above — the same 76 tools, same permission rules (full-scope admin key), with `--help` generated from each tool's schema. First-class `dlake s3 …` commands add the object-storage outlet (connections, browse, streaming put/get, and table-to-S3 export).

## Row-Level Security

SQL Server RLS via security policies + predicate functions reading `SESSION_CONTEXT` keys (`sub`, `email`, `domain`, `sql_identifier`, `api_key_id`, `is_owner`) — set automatically by DAB on every request and mirrored by the DDL API on all bulk read paths (exports, aggregate, data-viewer paging, change tracking, events). Build policies on the RLS page: pick a table, write/choose a predicate function, enable. A predicate that works under DAB behaves identically everywhere. **Events are RLS-filtered per consumer** (see Events).

## Change Tracking

SQL Server change tracking for sync consumers: enable per table, then `GET /api/ddl/change-tracking/tables/{table}/changes?fromVersion=N` returns inserted/updated/deleted row keys since that version (`/version` gives the current watermark). Requires `data.read`, and honors a scope-restricted key's per-entity read grant (a field-restricted key gets only its granted columns in each change's row image). Ideal for incremental pull sync; pair with Events for push. When a key's database principal is provisioned, a scoped key's changes read is additionally **DB-enforced** under that principal (fail-closed 403 if the principal isn't provisioned — re-save the key's scope).

## Time Travel (temporal history)

SQL Server system-versioned temporal tables: enable Time Travel per table (Schema Builder → the table's **Time Travel** toggle) and the engine records every row version into a `<table>_History` side table automatically — query the past without pre-planned snapshots. Mutually exclusive with Concurrency Protection (a versioned table can't carry CP's INSTEAD OF DELETE trigger). Requires `data.read` to read; the read paths are **RLS-scoped over history exactly like a live read** (the base-table security predicate is mirrored onto the history table) and honor a scope-restricted key's per-entity read grant. An **All-fields** scoped key's temporal reads run under its database principal (DB-enforced, incl. history-table `SELECT`); a **field-restricted** key is refused (403) on the temporal endpoints (they return whole rows). When RLS applies to the table, temporal reads **fail closed**: if the history table's RLS mirror can't be verified, the read is refused with an error telling you to retry or re-enable Time Travel (which re-establishes the mirror) — it never falls back to returning unfiltered history. Timestamps are **ISO-8601 UTC** (marker-less values assumed universal, e.g. `2026-07-01T00:00:00Z`).

REST under `/api/ddl/temporal/tables/{table}/…`; over MCP four read-only tools (use the tenant's **active schema** table name — call `get_active_schema` if unsure):
- **`temporal_status`** `{table}` (`/status`) — is versioning on, plus period columns / primary key / history-table name / `state` (`on` | `dormant` | `off`). Call first to confirm a table is versioned.
- **`query_as_of`** `{table, asOf, page?, pageSize?}` (`/as-of`) — the table's rows **as they existed at** `asOf` (paged; page 1 / pageSize 50 by default).
- **`row_history`** `{table, key}` (`/row-history`) — the full version timeline of **one row**, keyed by primary key (`key` is a column→value object, e.g. `{"Id": 42}`; supply every column of a composite PK); each version carries its `dl_sys_start`/`dl_sys_end` UTC validity window.
- **`time_travel_diff`** `{table, from, to, page?, pageSize?}` (`/diff`) — rows **added / removed / changed** between two instants, with before/after values for changed columns (joined on the primary key).

Example: `temporal_status {"table":"account"}` → then `query_as_of {"table":"account","asOf":"2026-07-01T00:00:00Z"}` or `time_travel_diff {"table":"account","from":"2026-06-01T00:00:00Z","to":"2026-07-01T00:00:00Z"}`.

## Concurrency Protection

Optimistic-concurrency + soft-delete for a table:
- UPDATEs must echo the row's current `TimeStamp` (rowversion) as `dl_expected_ts` — a strict AFTER trigger rejects stale writes ("Row was modified").
- DELETEs are intercepted at the DB layer (INSTEAD OF) and become `dl_deleted = 1` soft deletes; a sweep purges flagged rows on demand. **Through the DAB API a direct `DELETE` is rejected** with `DAB_CONCURRENCY_BLOCKED_DELETE` — soft-delete instead by `PATCH`ing `dl_deleted=true` together with `dl_expected_ts` (the row's current `TimeStamp`).
- A bypass list exempts named accounts.
Enable/disable per table from the table detail page. The protection columns (`dl_expected_ts` is virtual — you send it, it's never stored; `dl_deleted`) are server-managed.

## Audit Stamping

Per-table `dl_` audit columns (created/modified by/at) maintained by an AFTER trigger (`tr_<t>_dl_AuditStamp`), attributing writes to the JWT identity (including which API key wrote, via `SESSION_CONTEXT('api_key_id')`). A table can be **frozen** (`tr_<t>_dl_AuditFreeze`): stamps preserved but no longer updated. Stamping columns are server-managed — never send them in write bodies.

## Events (SSE, Polling, Webhooks)

Row-change events captured per table by a `tr_<t>_dl_Event` trigger into an outbox (`DLO.EventOutbox`), consumable three ways:
- **Polling** — `GET /api/ddl/events/changes?cursor=N&tables=&ops=` (cursor-based).
- **SSE** — mint a stream token (`POST /events/stream/token`), connect to `/events/stream?token=...`; `Last-Event-ID` resume, optional `coalesce=<ms>` batching; default is live-from-now, `since=<id>` replays history.
- **Webhooks** — see Webhooks.
Permissions: `events.read` to consume (all roles), `events.manage` (Admin) to enable capture per table or manage webhooks. **Row-Level Security applies**: on RLS-protected tables, insert/update events are delivered only if the consumer's identity can read the row (SSE/polling: the caller; webhooks: the subscription owner). Delete events always pass (row key only). Suppressed events still advance cursors. **Scope-restricted keys**: a scoped key's polling + SSE feed is narrowed to the entities it may read, and a **field-restricted** key also has each event's payload **column-filtered** to its read field policy — excluded columns are stripped from `data`, `rowKey`, and `changedColumns` before delivery (a not-granted primary-key column is stripped from `rowKey` too; grant it to correlate on it).

**SSE frame format — handle BOTH shapes; do NOT key on the event name:**
- **Live** (default after connect): one Server-Sent Event per change — `event: insert` | `update` | `delete`, and `data:` is a single event object: `{ "id", "occurredUtc", "schema", "table", "op", "rowKey": {<key columns>}, "data": {<full row, incl. control + dl_ columns>}, "changedColumns": [...], "actor": {...} }`.
- **Replay** (connect with `&since=<cursor>`): coalesced batches — `event: batch`, and `data:` = `{ "from": N, "to": M, "count": K, "events": [ <event object>, ... ] }`.
- `:heartbeat` comment lines are sent periodically to keep the connection open.

Track the highest event `id` you see (or `to` from a batch) and send it as `&since=` on reconnect for lossless replay. If the server detects a delivery gap it **terminates the stream** rather than silently skipping events; reconnect with your last `&since=` and the stream **catches up** from there before resuming live. Delete events carry `rowKey` only (no `data`).

## Webhooks

`POST /api/ddl/events/webhooks {name, url, filter:{tables, ops}}` → subscription + `whsec_` secret (shown once — used to verify the Stripe-style HMAC signature header `t=...,v1=...`, 5-minute replay window). The delivery worker POSTs event batches with retries and exponential backoff; subscriptions that exhaust retries dead-letter (inspectable per subscription) and **auto-disable after 3 consecutive dead-letters** so a permanently broken endpoint stops churning. Destination URLs are **SSRF-checked (no loopback/RFC1918/link-local) both when the subscription is created and again at connect/delivery time** — a URL that resolves to a private/internal address is rejected even if it looked safe at creation.

**Undelivered events are retained until delivered** (not just for the fixed replay window), so a subscription that's briefly down catches up once it recovers — but retention is bounded: an event that goes **undelivered for 30 days** trips the max-age cap, which **auto-disables the lagging subscription and releases its retained backlog** so one dead consumer can't pin the outbox forever. Re-enable (and optionally replay) once the endpoint is healthy.

**Replay:** `POST /api/ddl/events/webhooks/{id}/replay` rewinds the delivery cursor so past events still in the outbox (≤7-day retention) are re-delivered — `fromRetentionStart`, `fromEventId`, or `fromUtc` (clamped to the retention floor; re-enables + clears failure state). Re-delivery is at-least-once (dedup by `EventId`); replayed events are existence-checked (on every table) and re-evaluated against current RLS, so a since-deleted row's insert/update is not re-sent (its delete tombstone still replays). **Snapshot vs. live:** a default-payload subscription re-sends the captured point-in-time snapshot from the outbox; a `payloadQuery` subscription re-runs the query at replay time, so it reflects *current* state (or the default fallback for since-deleted rows), not the historical row — use a default-payload subscription for exact historical replay.

**Custom payload query (nested/multi-table payloads):** a subscription can carry an optional `payloadQuery` — a read-only `SELECT … FOR JSON` run at delivery time, keyed by the changed row, whose JSON replaces each event's `data`. So one webhook can deliver a joined/nested payload (e.g. an order with its line items) instead of just the watched row. Parameters available: `@<PkColumn>` per primary-key column, plus `@dl_op`/`@dl_table`/`@dl_schema`/`@dl_rowKeyJson`. Validated on save (single SELECT, must contain FOR JSON, parsed against the DB); RLS-scoped to the subscription owner; empty result (e.g. on DELETE) falls back to the default payload; query errors retry like any delivery failure. Reads live data at delivery time. Leave blank for the default single-row payload.

## Data Quality

Define **rules** that watch a table and alert you when something looks wrong — the table emptied out, a sync stopped, a column filled with nulls, or the row count swung wildly. Rules run automatically on a schedule and keep a pass/fail history, so you find out a feed broke before your dashboards do.

**Rule types**: **Row count minimum** (fails below N rows); **Row count drift** (fails when the count moves more than X% vs the previous check — the first check just records a baseline); **Null rate** (fails when a chosen column's NULL fraction exceeds a threshold); **Freshness** (fails when the newest row is older than N hours, measured from a timestamp column you name; no rows with a value there = stale); and **Row count parity** (`row_count_parity` — tunes the full-mirror source-vs-destination check that runs *during* a sync; per-entity `tolerancePct`/`toleranceRows`, or disable to silence its warning). See the admin control plane / DAB sections for the exact `set_dq_rule` config shapes.

Lives under **Settings → Data Quality**. Each rule binds to a **connection** (which supplies the alert recipients and email switch) and a **table**. Add / edit / delete rules, see a status pill per rule (Passing / Failing / Not checked), and open a rule's **result history** (every evaluation with its measured value, pass/fail, timestamp). A rule can be toggled off (kept but not evaluated). Rules run on a background schedule (~every 5 minutes); **Evaluate now** re-checks one rule and **Evaluate All** runs every enabled rule immediately.

**Alerts** are kept quiet two ways: they respect the per-connection **Email on warning** switch (a violation is treated as a warning — off means the failure is still recorded in history but no email is sent), and they fire **only on the pass→fail transition** (one alert when a rule first goes red, not every cycle; a later recovery-then-fail is a fresh transition). Managing Data Quality rules needs the **`data.ingest`** permission (Admin role; the owner always has it) — the same tier as Import and the raw-SQL tools. Deleting a rule also removes its result history.

## Object Storage (S3)

Link AWS S3 (or S3-compatible) buckets to browse, upload and download files — and, on
SQL Server 2022+ tenants, **attach** parquet/CSV files as queryable external tables.
Lives under **Settings → Object Storage (S3)**.

**Connections**: add a connection with a name, region, bucket, optional key prefix,
optional custom endpoint (for S3-compatible storage — must be `https://` and pass the
same SSRF checks as a webhook), an access key id, and a secret access key. The **secret
is write-only** — it is encrypted at rest and never shown again; leave it blank when
editing to keep the stored one. **Test** probes the bucket. Managing connections needs
**`data.ingest.manage`**; anyone with an ingest bucket can browse read-only.

**Bucket browser**: navigate folders with a breadcrumb, see each object's size and
modified time, **upload** files (streamed straight to S3, no memory buffering),
**download**, and **delete** (with confirm). Large listings page with **Load more**.

**Attach (SQL Server 2022+ only)**: pick an object or prefix, choose parquet or CSV
(with a first-row-header toggle for CSV), name the target table, and define the columns
(name + T-SQL type). The connector creates a PolyBase external table in a per-connection
schema (`s3_<id>`, named from the connection's immutable numeric id — renaming the
connection later never renames or orphans the attached tables) that you can then query
like any table. The attached-tables list
shows what's linked, with **Detach** (drops the external table; the S3 data is
untouched). Attaching a table name that is already attached is refused with a clear
message — detach it first or pick another name. Rotating a connection's AWS keys
(saving a new secret) also refreshes the SQL-side credential of any attached tables,
so attached tables keep working after a key rotation. On SQL 2017/2019 tenants the
attach panel is replaced by an inline note —
*"Attaching S3 files as external tables requires SQL Server 2022 or later."* — because
the estate is pre-2022; browse/upload/download still work everywhere. Attach was
**live-verified on the SQL Server 2025 pilot** (see `docs/s3-connector.md`).

**Bucket locked down by IP?** PolyBase reads S3 **from the SQL Server's public IP**,
not the Data Lake app's — an `aws:SourceIp`-restricted bucket policy must also allow
the SQL server's egress IP. The tell-tale symptom: the bucket browser works fine, but
attaching (or querying an attached table) fails with *"content of directory cannot be
listed"*.

**Exporting a table into a bucket**: `dlake s3 export <connection> <table> --format
csv|parquet` (or the admin-plane `export_to_s3` tool) runs the normal scope-enforced
export pipeline server-side and drops the finished file straight into the bucket.

**Scoping keys to external tables**: attached external tables appear in the API-key
scope grid (Settings → Data Engine (DAB) → API Keys) under a **"Lake / external
tables"** group, shown schema-qualified (e.g. `s3_sales.Orders`) with an S3-backed
hint. They are **read-only surfaces** — only the Read action (and its column-level
field policy) applies. A scoped key reads them through the raw-SQL tools (with the
`AllowRawSql` opt-in), where access is **database-enforced** by the key's SQL
principal; the external tables are never exposed through the Data Engine (DAB) API
itself. A lake row never grants a working-schema table of the same name (and vice
versa).

## Audit Logs

Every DDL/SQL operation is logged to `DLO.DDL_AuditLog` (user, role, statement, target, success/error). The Audit Logs page (requires `audit.view`, Admin by default) filters by operation, user, date, and success. Auth events (logins, key exchanges, revocations) are logged separately by the Auth API.

## Notifications

In-app notification bell: schema changes, engine deploys, failed operations, and platform notices land per user. Mark-as-read individually or in bulk; ReadOnly+ can view (`notifications.view`).
