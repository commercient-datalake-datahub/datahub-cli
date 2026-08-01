---
name: dlake
description: >-
  Drive the Commercient Data Lake / Data Hub CLI (`dlake`, npm `@commercient/dlake`) to build and
  operate a SQL Server backend and its auto-generated REST/GraphQL Data API (DAB). Use this skill
  whenever the task involves `dlake`, a Commercient Data Lake / Data Hub tenant, or building an app,
  backend, schema, stored procedure, API key, or Data API on that platform — even if the user just
  says "the data lake CLI", names a `dlk_` key, or asks how to create tables / expose entities /
  place transactional writes there. It encodes the correct command ORDERING and the non-obvious
  gotchas that otherwise cost repeated failed calls.
---

# Driving the `dlake` CLI

`dlake` is the terminal/CI client for a **Commercient Data Lake / Data Hub** tenant — a per-tenant
Microsoft SQL Server database with a schema builder, an auto-generated **Data API (Microsoft Data
API Builder / "DAB")** exposing REST + GraphQL, scoped API keys, row-level security, change events,
and MCP connectors for AI agents. Source is proprietary; the CLI ships as binaries + npm.

The platform is the backend. You build on it by: creating schema, exposing entities to the Data API,
minting a scoped key, and having an app call the REST/GraphQL endpoints. This skill exists because
the *ordering* of those steps and a handful of argument/behavior quirks are not obvious, and getting
them wrong produces confusing errors (a `403` on a valid key, `EntityNotFound` on a table that
exists). Follow the sequence and rules below and it goes smoothly.

## Command surface

```
dlake login --domain <tenant>                 # auth with a dlk_ API key: env DLAKE_API_KEY | --api-key-stdin | no-echo prompt
dlake status                                  # tenant + service health
dlake admin list                              # 126 control-plane tools (DDL, keys, exposure, DAB, RLS, events)
dlake admin <tool> --help                     # argument schema for one tool
dlake admin <tool> [--arg value ...]          # invoke it (schema/keys/exposure/etc.)
dlake tool  list                              # ~23 data-plane tools (records, query, aggregate, ingest, export)
dlake tool  <tool> [--arg value ...]          # invoke it (create_record, query, execute_entity, ...)
dlake guide api | guide help | guide cli      # live API + platform docs, and the CLI reference
```

Two planes: **`admin`** = control plane (define & expose schema, manage keys, restart DAB).
**`tool`** = data plane (read/write rows, run queries) — and it only sees *exposed* entities.

## The canonical build sequence

Do it in this order. The two `restart_dab` calls are the load-bearing part.

```bash
# 1. Authenticate with a FULL-SCOPE admin key (the admin plane refuses scoped keys)
dlake login --domain <tenant>
dlake tool get_active_schema                       # confirm the schema you'll build in (usually DLO)

# 2. Define schema (operates on the ACTIVE schema)
dlake admin create_table --tableName <t> --columns @cols.json --primaryKey Id
dlake admin describe_table --tableName <t>         # read it back: per-column typeToken, nullability, PK, FKs
dlake admin create_procedure --procedureName <p> --parameters @params.json --body @proc.sql

# 3. Expose every entity you want in the API, THEN apply with a restart
dlake admin set_entity_exposure --entity <t> --expose true          # repeat per entity + proc
dlake admin restart_dab --confirm true

# 4. Now the data plane can see them — seed / read / execute
dlake tool create_record --help                                     # confirm the row-payload arg name
dlake tool create_record --entity <t> ...                           # (or ingest_table for bulk, natural-key tables)

# 5. Mint a scoped key for the app, THEN restart again so DAB learns its role
dlake admin create_api_key --keyName <app> --expirationDays 180 --scope @scope.json
dlake admin restart_dab --confirm true                             # ← REQUIRED, see Rule 1
```

## Rules (learned the hard way — each prevents a specific failure)

1. **After creating (or re-saving the scope of) a *scoped* API key, `restart_dab` again.** DAB's
   running config only gains the key's per-key role (`key_<id>`) on regeneration. Skip this and every
   request with that key returns `403 AuthorizationCheckFailed` even though the key and scope are
   correct. Unscoped keys are exempt (they use the shared `datalake_user` role DAB already knows).
   The restart must be driven by a **full-scope admin key** — a scoped key is locked out of the admin
   control plane, so it can never restart DAB for itself. (Revoking a key is the one case the
   platform regenerates immediately, so the dead role disappears at once.)

2. **The data plane only sees *exposed* entities.** `create_record` / `read_records` /
   `execute_entity` / `export_table` return `EntityNotFound` for a table that exists but hasn't been
   through `set_entity_exposure --expose true` + `restart_dab`. Expose first, then use. Same for a new
   view or procedure. (Raw-SQL reads — `query` / `export_query` — go straight to the database and need
   no exposure.)

3. **The active schema (usually `DLO`, not always) can't be switched from the CLI** — only the web
   UI's schema dropdown switches it; there is deliberately no `set_active_schema` tool. All DDL and
   `ingest_table` land in the active schema. Read it with `dlake tool get_active_schema` (REST:
   `GET /api/ddl/dab/active-schema`); to keep multiple apps tidy in one schema, prefix object names
   (e.g. `pizza_orders`).

4. **On Windows, pass array/object args as `@file.json`.** Inline quoted JSON gets mangled by the
   shell. This is the reliable path on every platform and works for **any** arg type; `@@` passes a
   literal leading `@`. Simple scalar arrays also accept `a,b,c`.

5. **Scope JSON shape:** `[{ "entityName": "<name>", "canRead": true, "canExecute": true }]` — plus
   `canCreate` / `canUpdate` / `canDelete` as needed, and an optional `schemaName` (omit it for the
   tenant's working schema; set it only for an external/lake schema such as `s3_<conn>`). Wrong field
   names fail with "empty entityName" (there is no `entity` field) or "no actions selected" (there is
   no bare `read`/`write`). REST `PATCH`/`PUT` by key need **both** `canCreate` and `canUpdate` — DAB
   treats them as upserts. A scoped key **cannot** drive the admin control plane.

6. **Stored-procedure parameters:** each is `{ name, dataType, maxLength?, direction? }` — the type
   key is **`dataType`**, not `type` (only `create_table --columns` uses `type`), and the **name must
   start with `@`** (`{"name":"@CartJson","dataType":"NVARCHAR","maxLength":-1}`; `"CartJson"` is
   rejected with *"must start with '@'"*). Also avoid **reserved T-SQL keywords as column names** —
   `LineNo` fails because `LINENO` is reserved, as are `Key`, `Order`, `User`, `Percent`, `Public`;
   rename to `LineNumber` / `SeqNo`.

7. **`ingest_table` refuses IDENTITY-PK tables** ("Row-by-row upsert needs a key whose values come
   from the file"). Bulk-load only natural-key tables; seed identity-keyed tables with
   `create_record` / REST `POST`. Ingest also needs the `data.ingest` permission and is **refused for
   scope-restricted keys** entirely — use an unscoped key for it.

8. **`dlake query` / `dlake tool query` are SELECT-only** (a single `SELECT` or `WITH … SELECT`; no
   `;`-separated statements), capped at **10,000 rows**, and require the active-schema qualifier (e.g.
   `FROM DLO.<table>` — an unqualified name resolves against the login's default schema and errors).
   They are **refused for scope-restricted keys** unless the key was minted with the `AllowRawSql`
   opt-in. Writes go through `create_record` / `execute_entity` / procedures.

9. **Registration and passwords are the user's to enter.** `dlake register start` creates an account
   and takes a portal password (`--password-stdin` or a prompt) — hand that step to the user; don't
   run it for them. (`dlake register status --watch` then surfaces the once-only 7-day bootstrap
   owner-admin key and saves it to a profile.)

## Choosing a write path

- **Multi-row writes that must be atomic** (an order header + its line items): put them in a stored
  procedure and expose it. REST/GraphQL writes are per-statement and can half-succeed, leaving
  orphans. The proc takes a flat param set (pass nested data as a JSON string parameter and shred it
  with `OPENJSON`), runs in one transaction, and returns the created rowset. Invoke via
  `POST /auth/{tenant}/api/<proc>` (procedures are **POST-only** on DAB) or `dlake tool execute_entity`.
- **Single-row insert/update:** `create_record` / REST `POST` / `PATCH` (PATCH is the preferred
  upsert — send only changed columns; never send computed/`dl_*`/rowversion columns).
- **Reads with joins/nesting/aggregates:** prefer **GraphQL** (`/auth/{tenant}/graphql`) — one round
  trip for nested reads once relationships are declared (`set_relationship`). Fall back to the
  read-only SQL endpoints only for what GraphQL can't express (window functions, `UNION`, self-joins,
  grouping/sorting by a *related* entity's column, nested aggregations, offset paging).
  On MCP/CLI there is no GraphQL surface at all — use `read_records`, `aggregate` and `query` there.

## Connecting an app to the Data API

Never ship a `dlk_` key to a browser. Put it in a tiny server-side proxy that adds the auth headers
and forwards to DAB. Two auth options (full endpoint shapes in the **Data API (DAB) — HTTP contract**
section below):

- **Simplest:** send the raw key as `X-API-Key: dlk_...` to the auth-proxy path; it exchanges for you.
- **Explicit:** `POST /api/auth/apikeys/exchange` → Bearer JWT, then `Authorization: Bearer <jwt>`.

Both also send `X-MS-API-ROLE`: `datalake_user` for an unscoped key, `key_<apiKeyId>` for a scoped one
(the id is in the `create_api_key` response and the JWT `roles` claim). The auth-proxy infers the role
from the JWT when the header is absent, but send it explicitly — it is what the API guide documents and
it fails loudly instead of silently picking a role.

See the **Data API (DAB) — HTTP contract** section at the end of this file when wiring the HTTP
layer — it has the base URLs, the exchange call, OData read params, write semantics
(PATCH/PUT/DELETE, concurrency `dl_expected_ts`), the stored-procedure POST contract, and the
events/SSE/webhook endpoints for live updates.

## Discovering the truth on a live tenant

The platform evolves and tools have exact argument schemas — verify rather than guess:

- `dlake admin <tool> --help` / `dlake tool <tool> --help` — the authoritative arg schema.
- `dlake admin list` / `dlake tool list` — what actually exists on this tenant.
- `dlake admin list_exposed_entities` (or `dlake entities list`) — what the Data API currently serves.
- `dlake admin dab_status` — whether the API container is running (after a restart).
- `dlake guide api` — the live API guide (auth, REST/GraphQL, events, write semantics).

Exit codes: `0` success · `1` error · `2` usage error · `3` permission/auth denied.

---

## dlake Data API (DAB) — HTTP contract

Read this when building the HTTP layer that talks to a tenant's Data API. Everything here is also
retrievable live via `dlake guide api`; this is the distilled, app-building subset.

### Base URLs

| Surface | URL |
|---|---|
| Data API (DAB) via auth proxy | `https://datalake-ms-dab.commercient.com/auth/{tenant}/api/{entity}` |
| GraphQL | `https://datalake-ms-dab.commercient.com/auth/{tenant}/graphql` |
| Auth API (key exchange) | `https://datalake-ms-auth-api.commercient.com` |
| DDL API (exports, aggregates, events) | `https://datalake-ms-ddl-api.commercient.com` |

`{tenant}` is the **short slug** (dot-free lowercase DNS label), not the portal hostname.

### Authentication — two paths

**A. Raw key (simplest for a server-side proxy).** Send the key directly; the auth-proxy exchanges it:

```
X-API-Key: dlk_...
X-MS-API-ROLE: key_<apiKeyId>      # scoped keys; unscoped keys use datalake_user
```

**B. Explicit exchange** (when you want to manage the ~60-min JWT yourself):

```bash
curl -X POST https://datalake-ms-auth-api.commercient.com/api/auth/apikeys/exchange \
  -H "Content-Type: application/json" \
  -d '{"apiKey":"dlk_...","domain":"{tenant}","issueRefreshToken":false}'
## -> { "accessToken": "eyJ...", "expiresUtc": "..." }
```

Then: `Authorization: Bearer <accessToken>` plus `X-MS-API-ROLE` (`datalake_user`, or `key_<apiKeyId>`
for a scoped key). Find the role in the `create_api_key` response (`apiKeyId`) or the JWT `roles` claim.

> A newly-minted **scoped** key returns `403 AuthorizationCheckFailed` until a **full-scope admin key**
> runs `dlake admin restart_dab --confirm true` — DAB must regenerate to know the `key_<id>` role.

### Reading (REST, OData-style)

```
GET /auth/{tenant}/api/{entity}?$filter=City eq 'Atlanta'&$select=Name,City&$orderby=Name&$first=100
GET /auth/{tenant}/api/{entity}/{keyColumn}/{value}          # by key
```

URL-encode filter values (spaces in `$orderby=Id desc` must be encoded). List responses are
`{ "value": [ ... ], "nextLink": "..." }`. OData `$count` and `contains()` are **not** supported —
use GraphQL for totals and substring search. Listing "not deleted" rows needs
`$filter=dl_deleted eq null or dl_deleted eq false`: `dl_deleted` is nullable and `eq false` does not
match `NULL`, so the short form hides every untouched row.

### Reading (GraphQL — preferred for joins/nesting/aggregates)

Declare relationships once (admin), then one restart, then nest in a single request:

```bash
dlake admin set_relationship --parentEntity orders --childEntity order_items \
  --parentKeyColumn Id --childFkColumn OrderId --cardinality many
dlake admin restart_dab --confirm true
```

```graphql
{ orders(first: 20, orderBy: { Id: DESC }) {
    items { Id Status Total
      order_items(first: 200) { items { MenuItemId Qty LineTotal } } }
    hasNextPage endCursor } }
```

Relationships are DAB config only — they don't create DB foreign keys and enforce nothing.

### Writing (REST) — per-statement, NOT transactional

| Method | Semantics |
|---|---|
| `POST /api/{entity}` | Insert (body = column values) |
| `PATCH /api/{entity}/{key}/{value}` | Partial upsert — **preferred**, send only changed columns |
| `PUT /api/{entity}/{key}/{value}` | Full replace (omitted columns → NULL; fragile) |
| `DELETE /api/{entity}/{key}/{value}` | Delete (non concurrency-protected tables only) |

Hard rules:
- **Never send** computed/server-managed columns: `id` (computed), `TimeStamp` (rowversion),
  `dl_*` audit columns — DAB returns `400`. `dl_deleted` is the only documented exception.
- **Concurrency-Protected (CP) tables:** every UPDATE must echo the row's current `TimeStamp` as
  `dl_expected_ts` in the body. Read it via the **list** endpoint immediately before the PATCH
  (key-path GET may be cached/stale). A stale value fails "Row was modified" — re-read and retry.
- Soft-delete a CP row with `PATCH ... {"dl_deleted": true, "dl_expected_ts": <ts>}`; a direct
  `DELETE` on a CP table is rejected with `DAB_CONCURRENCY_BLOCKED_DELETE`.
- `PATCH`/`PUT` by key require the key to hold **both** create and update rights (upsert semantics).
- Two mutations in one GraphQL document are **not** atomic. For atomicity, use a stored procedure.
- Data-plane failures return a generic `DatabaseOperationFailed`. To see the real SQL error, switch
  the tenant's DAB host mode to `development` (`PUT /api/ddl/dab/host-mode`), restart DAB, and set it
  back to `production` when done.

### Transactions (stored procedures)

Expose a proc, then `POST` a flat name→value body. It commits everything or nothing and returns the
created rowset:

```
POST /auth/{tenant}/api/{proc}
{ "Param1": "...", "CartJson": "[{\"menuItemId\":2,\"qty\":1,\"toppingIds\":[1,5]}]" }
## -> { "value": [ { ...returned row(s)... } ] }   (HTTP 201)
```

- Procedure entities are **POST-only** by default (a read-only proc can opt into `GET`).
- Pass nested/array input as a **JSON string parameter** and shred it in the body with `OPENJSON`.
- In GraphQL a proc appears as `execute<ProcName>` on the **Query** type (not Mutation).
- Body pattern for atomicity: `SET XACT_ABORT ON; BEGIN TRY BEGIN TRAN ... COMMIT END TRY
  BEGIN CATCH IF @@TRANCOUNT>0 ROLLBACK; THROW END CATCH`.
- Editing a proc is `drop_procedure` + `create_procedure` + a restart. Rolled-back attempts still
  consume IDENTITY values — gaps are expected.

### Live updates (events)

Row-change events from event-captured tables (`dlake admin enable_event_capture --table <t>`), all
Bearer-authenticated on the DDL API:

```
GET  /api/ddl/events/changes?cursor=0&max=100                 # poll
POST /api/ddl/events/stream/token   -> { liveToken }          # then:
GET  /api/ddl/events/stream?token=...&coalesce=250            # SSE (live from now; &since=<id> replays)
POST /api/ddl/events/webhooks  { name, url, filter:{tables,ops}, payloadQuery? }  # -> whsec_ secret (once)
```

Handle **both** SSE frame shapes and don't key on the event name: live frames are one event object
per change (`event: insert|update|delete`), while a `&since=` reconnect replays coalesced
`event: batch` frames (`{ from, to, count, events: [...] }`). Track the highest `id` (or `to`) and
send it as `&since=` on reconnect. A scoped key's feed is narrowed to entities it may read; a stream
closes ~1 min after the key is revoked or re-scoped — treat an unexpected close as "re-exchange and
reconnect". Webhook deliveries carry a Stripe-style HMAC signature (`t=...,v1=...`, 5-min window).

### Aggregates & read-only SQL (DDL API)

```
POST /api/ddl/sql/aggregate   { table, group_by:[{column,date_part?}], aggregates:[{function,column}], filters, having, order_by, top }
POST /api/ddl/sql/query       { sql }                         # single SELECT / WITH…SELECT, ≤10,000 rows
POST /api/ddl/sql/export-query?format=csv|parquet  { sql }    # async signed-URL export, no row cap
```

Functions: `count, count_distinct, sum, avg, min, max`. All enforce RLS and key scope. The two raw-SQL
endpoints are refused for scope-restricted keys without the `AllowRawSql` opt-in, and no more than
**10 of them may run concurrently per user** (an 11th returns `409 Too many concurrent queries`).

### Security checklist for an app

- Keep the `dlk_` key server-side (env var / secret manager); never expose to the browser.
- Give the app a **scoped** key with the minimum actions (read the menu, execute the checkout proc).
- Lean on a stored proc for multi-row writes so ownership chaining means the key needs only
  `canExecute` on the proc, not direct table write grants.
- Add Row-Level Security so a key/user sees only its own rows before going to production.
- Use the shortest workable key expiry — `expirationDays` must be **7, 30, 90, or 180** — and revoke
  unused keys (revocation takes effect within ~30 s and kills live MCP sessions).
