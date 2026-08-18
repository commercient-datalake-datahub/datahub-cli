---
name: dlake-txdownloaderpro
description: >-
  Set up AND operate the TxDownloaderPro writeback leg of a Commercient integration with the
  `dlake` CLI (npm `@commercient/dlake`). Setup: make the gateway sync-state objects —
  `TxDownloaderPro`, `TxDownloaderProTrans`, `TxDownloaderProBlockDuplicateId`,
  `TimeStampRepository`, `TimeStampRepositoryHistory` — reachable through the tenant's Data API,
  then scope an API key to them and apply it with a single deliberate DAB restart. Operation:
  CRUD on the setup/configuration rows and on the in-flight transaction rows, reading and editing
  the JSON- and XML-bearing columns that carry the field mapping, and the filter-operator
  vocabulary that decides which retrieved CRM records reach the source system. Use this skill
  whenever the task mentions TxDownloaderPro, writeback, CRM-to-ERP sync state, `SFUpdated`,
  a stuck or held writeback record, "expose the TxDownloaderPro tables / add them to a key",
  editing a writeback field mapping, `ProcessStructure` / `ResultStructure` / `XMLResult` /
  `JsonResponse`, `{{Object.Field}}` template paths, or filtering/transforming downloaded CRM
  data before it is written back to the source. This is ONE STEP of standing up an integration —
  for registration, CRM choice and the ERP connector use `dlake-integration-setup`; for general
  tenant operation use `dlake`.
---

# Setting up TxDownloaderPro writeback with `dlake`

A Commercient integration has two legs. The **inbound** leg pulls the customer's ERP data into
the Data Lake. **TxDownloaderPro** is the **writeback** leg going the other way: it picks up
records a user flagged in the CRM, carries them back toward the source/ERP system, and reports
the outcome back to the CRM.

This skill covers the Data Lake side of that, in two halves:

- **Setup** (sections 1–7) — making TxDownloaderPro's state objects reachable through the
  tenant's Data API and giving a key access to them.
- **Operation** (sections 8–12) — once they are reachable, reading and correcting the
  configuration rows and the in-flight transaction rows through the CLI, editing the field
  mapping that lives in their JSON- and XML-bearing columns, and knowing exactly which
  filter operators are available to decide what reaches the source system.

**Before you reach for row-level CRUD, read section 14.** The control plane now carries a
dedicated **`txdownloaderpro_*`** admin tool set that drives the product's own configuration
API — processes, both mapping documents, ERP flags, the master and per-process switches, and
three live-CRM helpers. That is the right tool for *configuring* writeback; the exposure and
row-level work in sections 1–12 is for *inspecting and correcting state* the configuration API
does not reach.

It does **not** cover installing, scheduling or upgrading the TxDownloaderPro service itself —
that is done on the customer's own host, outside the `dlake` CLI.

The shape of it:

```
list_exposed_entities        → what does the Data API publish today?
set_entity_exposure  × N     → add each TxDownloaderPro object to the scope
create_api_key / set_key_entity_scopes
                             → give a key access to exactly those entities
restart_dab --confirm true   → ONE restart, after all of the above
list_exposed_entities        → verify `served`, not merely present
```

---

## 1. When you need this

A Data Lake tenant is perfectly usable **without** TxDownloaderPro. Plenty of tenants only ever
receive data. You need this skill when the customer wants changes made in the CRM to travel back
to their source system — order entry from the CRM, status updates, record creation that must land
in the ERP.

**Write the name in full.** TxDownloaderPro is a distinct product from TxDownloader; the shortened
form names something else.

TxDownloaderPro is a scheduled service that runs against the customer's gateway database. It
keeps its working state in a small family of tables in that database's `dbo` schema:

| Object | Role |
|---|---|
| `TxDownloaderPro` | The per-object sync configuration rows that drive each run |
| `TxDownloaderProTrans` | The transaction/state rows — one per record in flight |
| `TxDownloaderProBlockDuplicateId` | Duplicate suppression |
| `TimeStampRepository` | Per-key cursor/timestamp bookmarks written by some platform handlers |
| `TimeStampRepositoryHistory` | History alongside the above |

Errors from a run are written to a `Commercient_Error_Log` table, and per-customer configuration
is key/value rows in a `CommercientFlags` table. Both live in `dbo` as well.

TxDownloaderPro itself reaches these tables over a **direct database connection**, not over the
Data API. Exposing them is what makes them reachable to *everything else* — the CLI, MCP tools,
the SQL Editor's by-key paths, dashboards, and any app you build that needs to see or steer
writeback state. Exposure is what lets you operate writeback through the Data Lake; the service
itself does not depend on it.

## 2. What you actually expose

Those tables sit in `dbo`, outside the tenant's working (platform) schema — usually `DLO`, not
always. The platform handles that for you: when a tenant is seeded it builds **same-named,
explicit-column, single-table views in the working schema** over the gateway tables it finds,
including all five above. A view is created only for a table the tenant actually has, so a
tenant missing one simply has no view for it.

So the thing you expose is the **working-schema view**, addressed by its bare name
(`TxDownloaderProTrans`, not `dbo.TxDownloaderProTrans`). Because each wraps a single table,
SQL Server treats it as updatable — reads and writes both work, and a write body omits identity
columns exactly as it would against the base table.

Two consequences worth knowing before you start:

- **Raw SQL already sees them.** They are ordinary objects in the active schema, so
  `dlake query` / the `query` tool / the SQL Editor can read them the moment the tenant is
  seeded, with no exposure step at all. Exposure is specifically about the **Data API** (DAB)
  entity endpoints and the record-level tools built on them.
- **`CommercientFlags` and `Commercient_Error_Log` are not in that view set.** Do not assume a
  working-schema view exists for them. Check with `dlake admin list_views` before planning
  around either.

Confirm what a given tenant has before you expose anything:

```bash
dlake login --domain <tenant>
dlake tool get_active_schema          # the working schema these views live in
dlake admin list_views                # which TxDownloaderPro* views this tenant actually got
```

## 3. See the current Data API scope

```bash
dlake admin list_exposed_entities
```

Takes no arguments. It lists every table and view in this tenant's Data API scope with its type,
key fields and per-entity settings — and, importantly, tells **persisted scope apart from what is
actually being served**:

| Field | Meaning |
|---|---|
| `served` | `true` = in the running config, `false` = persisted but not served, `null` = served config unknown |
| `servedConfig` | Version of the served config, when it was generated, how long it has been serving |
| `pendingChanges` / `warning` | Names the difference between the two |

Trust `served`, not mere presence. An entity that is listed but `served:false` answers
`EntityNotFound` on a read — that is a missing restart, not a broken entity.

## 4. Add each object to the scope

```bash
dlake admin set_entity_exposure --entity TxDownloaderPro            --expose true
dlake admin set_entity_exposure --entity TxDownloaderProTrans       --expose true
dlake admin set_entity_exposure --entity TxDownloaderProBlockDuplicateId --expose true
dlake admin set_entity_exposure --entity TimeStampRepository        --expose true
dlake admin set_entity_exposure --entity TimeStampRepositoryHistory --expose true
```

| Argument | Type | Notes |
|---|---|---|
| `entity` | string | **Required.** Table/view name, **unqualified**, in the active schema |
| `expose` | boolean | **Required.** `true` adds to scope, `false` removes |
| `keyFields` | string array | The addressable key column(s); composite keys allowed. Required when the object has no derivable key |
| `confirm` | boolean | Required (`true`) **only** when removing (`expose:false`) |

Unknown properties are **rejected**, not silently ignored — a typo'd argument fails the call
rather than half-applying it.

**About `keyFields`.** Adding an entity validates that it exists *and* that it has an addressable
key. A view whose key cannot be inherited from a single base table — or a table with no primary
key — is **refused** unless you pass `keyFields`, because DAB cannot start with a keyless entity
and exposing one would take the tenant's whole Data API down. The gateway views each wrap one
base table, so their key is usually inherited; if a call is refused, supply the key explicitly:

```bash
dlake admin set_entity_exposure --entity TxDownloaderProTrans \
    --expose true --keyFields TxDownloaderProTransId
```

`keyFields` is validated case-insensitively against the entity's live columns. Use
`dlake admin describe_table` / `describe_entities` to find the right column rather than guessing;
`TxDownloaderProTransId` above is an illustration of the shape, not a promise about your tenant.

> Some clients cache the tool list for the life of a session. If `keyFields` is missing from the
> schema your client shows, reconnect.

**Removal is destructive.** `--expose false` withdraws all API access to that entity and requires
`--confirm true`.

## 5. Scope a key to them

This is the step people conflate, so be precise about the two layers:

| Layer | Tool | What it controls |
|---|---|---|
| **Entity exposure** | `set_entity_exposure` | Whether the Data API publishes the entity **at all**, tenant-wide. One switch for the whole tenant. |
| **Key scope** | `create_api_key --scope` / `set_key_entity_scopes` | Which of the published entities **one key** may touch, and with which verbs. Per key. |

They are AND-ed. Scoping a key to an entity that is not exposed gets you nothing, and exposing an
entity does not by itself narrow any key. An unscoped key is full-access.

### Mint a key scoped to writeback

Write the scope to a file — on Windows especially, inline JSON gets mangled by the shell, and
`@file.json` is the reliable path on every platform:

```json
[
  { "entityName": "TxDownloaderPro",                 "canRead": true },
  { "entityName": "TxDownloaderProTrans",            "canRead": true, "canCreate": true, "canUpdate": true },
  { "entityName": "TxDownloaderProBlockDuplicateId", "canRead": true },
  { "entityName": "TimeStampRepository",             "canRead": true },
  { "entityName": "TimeStampRepositoryHistory",      "canRead": true }
]
```

```bash
dlake admin create_api_key --keyName "<tenant>-writeback" --expirationDays 180 --scope @scope.json
```

| `create_api_key` argument | Type | Notes |
|---|---|---|
| `keyName` | string | **Required.** Display name |
| `expirationDays` | integer | **Required.** Must be `7`, `30`, `90` or `180` — no other value |
| `targetUserId` | integer | Own the key to a different user (admin-only; owner/dbo-admin targets need matching privilege) |
| `project` | string | Grouping label only, not a rights axis. Get-or-create by name, max 100 chars |
| `scope` | array | Per-entity scope. **Omit for a full-access key** |
| `allowRawSql` | boolean | Scoped keys only: allow read-only raw-SQL tools |

Each `scope` entry: `entityName` (**required**), optional `schemaName`, and the verb booleans
`canCreate`, `canRead`, `canUpdate`, `canDelete`, `canExecute`. Omit `schemaName` for the working
schema — that is what you want here, since you are scoping the working-schema views.

**The raw key is returned exactly once.** Only its hash is persisted. Store it before the call
scrolls away.

**A scoped key cannot drive the admin control plane.** Keep a full-scope admin key for the tools
in this skill; the writeback key is a data-plane credential.

**`allowRawSql`** is an opt-in for scoped keys, and when it is on, row-level security is the only
boundary left on those reads. Leave it off unless something concretely needs free-form SELECTs.

### Adjust an existing key instead

```bash
dlake admin list_api_keys                                     # find the id (metadata only)
dlake admin set_key_entity_scopes --apiKeyId <id> --scope @scope.json
```

| `set_key_entity_scopes` argument | Type | Notes |
|---|---|---|
| `apiKeyId` | integer | **Required.** From `list_api_keys` |
| `scope` | array | **Required.** The **FULL** scope list — it **replaces** the existing rows. `[]` makes the key full-access again |
| `allowRawSql` | boolean | Scoped keys only, default `false` |

`scope` is a replace, not a merge: to add writeback access to a key that already has other
entities, send the existing rows **plus** the new ones, or you will silently drop the rest.
Read the current state first with `list_api_keys` (which reports a scope-row count) and the key's
own scope before you overwrite it. Changes take effect on the key's next token exchange or
refresh.

`list_api_keys` takes an optional `project` (case-insensitive) and returns **metadata only** —
id, name, prefix, owner email, project, created/expiry/last-used, revoked state, active flag,
scope-row count. Key hashes and secrets are never returned.

### Rights are a separate, subtract-only axis

```bash
dlake admin get_key_rights --apiKeyId <id>
dlake admin set_key_rights --apiKeyId <id> --suppressOwner true --deniedPermissionKeys @denied.json
```

`get_key_rights` (`apiKeyId`, required) reports the key's rights model: the inherited baseline —
the owning user's live owner/dbo flags and full effective permissions — plus the key's current
removals.

`set_key_rights` sets **removals only, never grants**:

| Argument | Type | Notes |
|---|---|---|
| `apiKeyId` | integer | **Required** |
| `suppressOwner` | boolean | Suppress the owner axis on this key (default `false`) |
| `suppressDboAdmin` | boolean | Suppress the dbo-admin axis on this key (default `false`) |
| `deniedPermissionKeys` | string array | Permission keys this key may not use — **replaces** the current denylist; empty clears it |

It is a denylist subtracted from the owning user's live rights at every exchange and refresh, and
it replaces the current removals rather than adding to them. Use it when the writeback key is
owned by a privileged user and you want it to stop inheriting that privilege. It is not how you
grant entity access — that is `set_key_entity_scopes`.

## 6. Make it live — one deliberate restart

Neither `set_entity_exposure` nor `set_entity_settings` restarts the Data API. They persist the
intent. The running DAB container keeps serving the configuration it was started with until:

```bash
dlake admin restart_dab --confirm true
```

`confirm` (boolean) is the only argument and **must** be `true`.

**A DAB restart briefly interrupts the tenant's live Data API.** Treat it as an operator-initiated
event, not a reflex after every change:

- **Batch everything first.** Expose all five entities, set their keys, mint or re-scope the key,
  *then* restart once.
- **Do it when the customer is ready.** If a change is applied while other traffic is live, say
  "a restart is needed" and let the operator pick the moment rather than firing one yourself.
- **A scoped key needs the restart too.** After creating a scoped key — or re-saving an existing
  key's scope — DAB's running config only learns that key's per-key role on regeneration. Skip it
  and every request with that key returns `403 AuthorizationCheckFailed` even though the key and
  its scope are entirely correct. Unscoped keys are exempt. This is why the exposure work and the
  key work belong in the *same* batch: one restart covers both.
- **Only a full-scope admin key can restart DAB.** A scoped key is locked out of the admin control
  plane, so a writeback key can never restart DAB for itself.

## 7. Verify

```bash
# 1. Are the entities SERVED, not merely persisted?
dlake admin list_exposed_entities        # check `served: true` on each of the five

# 2. Does the Data API actually answer for them?
dlake tool read_records --help           # confirm this build's argument names
dlake tool read_records ...              # read one of the five

# 3. Does the KEY see them? Log in with the writeback key and repeat the read.
dlake login --domain <tenant>            # supply the writeback key
dlake tool read_records ...
```

Read the failures precisely — they mean different things:

| Symptom | Almost always |
|---|---|
| `EntityNotFound` on a listed entity | Missing or failed `restart_dab` — check `served` |
| `403 AuthorizationCheckFailed` on a correct scoped key | No restart since the key was created or re-scoped |
| Entity refused at exposure time | No derivable addressable key — pass `keyFields` |
| No such view on the tenant | That gateway table does not exist there; nothing to expose |

## 8. Operating it: the verbs

Everything from here on assumes section 1–7 is done: the objects are exposed, `served: true`, and
your key can reach them. Two families of verb do the work.

```
dlake tool read_records   --entity <E> [--filter <odata>] [--select <cols>]
                          [--orderby <list>] [--first <n>] [--after <cursor>]
dlake tool create_record  --entity <E> --data  @row.json
dlake tool update_record  --entity <E> --keys  @keys.json  --fields @fields.json
dlake tool delete_record  --entity <E> --keys  @keys.json
dlake tool query          --sql "SELECT ... FROM <schema>.<Entity> WHERE ..."
```

| Verb | Required arguments | Optional | Use it for |
|---|---|---|---|
| `read_records` | `entity` | `filter` (OData: `eq ne gt ge lt le and or not`), `select`, `orderby` (e.g. `['CreateDate desc']`), `first`, `after` | Everyday inspection of a handful of rows |
| `create_record` | `entity`, `data` (object of field/value) | — | Seeding a configuration row |
| `update_record` | `entity`, `keys` (object), `fields` (object) | — | Correcting one row, addressed by key |
| `delete_record` | `entity`, `keys` (object) | — | Removing a configuration row (see the warnings below) |
| `query` | `sql` | — | Read-only analysis: joins, counts, state breakdowns |

Three things about these that matter here:

- **`query` is read-only.** A single `SELECT` (or `WITH … SELECT`). It will not `UPDATE`. Every
  correction below therefore goes through `update_record`, one row at a time — there is no
  bulk-update verb, and that is a feature when you are editing sync state.
- **Schema-qualify in `query`, do not qualify in the record verbs.** `query` needs
  `<active schema>.TxDownloaderProTrans`; `--entity` takes the bare name. Get the schema from
  `dlake tool get_active_schema`.
- **Big text columns.** The mapping and payload columns are large text. Use `--select` to keep
  reads readable, and pull one row's full column with a targeted `query` when you actually need
  the body.

Confirm argument names against your build before scripting anything:

```bash
dlake tool read_records --help
dlake tool update_record --help
```

## 9. CRUD on the setup tables

`TxDownloaderPro` is the configuration table. One row is one **process**: one CRM object being
watched, with the query that finds records, the mapping that shapes them, and the flags that
steer the run. The service reads the active rows at the start of every run, so an edit here takes
effect on the **next run** — no restart of anything, no redeploy.

### The columns you actually operate

| Column | What it is | Editing it changes behaviour |
|---|---|---|
| `TxDownloaderProId` | Identity key for the process. Every transaction row carries it | Never edit |
| `ProcessName` | Human label for the process | Cosmetic |
| `CRMName` | Which CRM this process belongs to. Runs can be filtered to one CRM by this value | **Yes** — a wrong value takes the process out of the run |
| `IsActive` | Only active rows are picked up | **Yes, immediately on next run.** This is the safe on/off switch |
| `Query` | What to fetch from the CRM. **Shape depends on the CRM** — see below | **Yes** — the single biggest lever |
| `SFFlagNameField` | Name of the CRM field the service writes the "completed" marker into | **Yes** |
| `SFMessageNameField` | Name of the CRM field the service writes the status message into | **Yes** |
| `ProcessStructure` | JSON. Carries `{{Object.Field}}` template paths read against the retrieved record | **Yes** — this is field mapping (section 11) |
| `ResultStructure` | JSON. The four-part mapping used when pushing the source system's response back to the CRM | **Yes** — field mapping (section 11) |
| `IsDoNotUpdateFieldsInSF` | Steers the hold/immediate-advance behaviour of the state machine | **Yes, and it is load-bearing** — see section 10 |
| `FieldSynchronize` | Text. Written by the service; on at least one download path it is also read as a filter input (a created-date window, defaulting to the last hour, for one CRM's email objects) | **Yes on that path.** Treat as service-owned elsewhere |
| `IsTrackingFieldsCreated` | Records that the tracking fields have been created on the CRM side | Service-owned. Do not modify |
| `IsStreaming` | Selects a streaming variant of the fetch path | Service-owned. Do not modify |
| `TxDownloaderDllName`, `IsDownloadDll`, `AdditionalFile` | A plug-in/download mechanism | Service-owned. Do not modify |
| `DownloadError`, `IsLogEnable` | Diagnostics | Read them; do not rely on undocumented effects |

Read the configuration:

```bash
dlake tool read_records --entity TxDownloaderPro \
    --select "TxDownloaderProId,ProcessName,CRMName,IsActive,SFFlagNameField,SFMessageNameField" \
    --orderby "['TxDownloaderProId asc']"
```

Disable one process without deleting it — the correct way to stop a misbehaving mapping:

```bash
dlake tool update_record --entity TxDownloaderPro \
    --keys   '{"TxDownloaderProId": 12}' \
    --fields '{"IsActive": false}'
```

**Prefer `IsActive: false` to `delete_record`.** Transaction rows reference the configuration row
by `TxDownloaderProId`; deleting the configuration leaves those rows pointing at nothing, and
several of the service's paths join the two. Deactivate, observe a run, and only then consider
deletion.

### `Query` does not have one shape

This is the trap that costs the most time. The column is free text and each CRM handler
interprets it differently:

| CRM family | What `Query` holds |
|---|---|
| Salesforce-shaped paths | A SOQL statement — root fields, parent dot-fields, child subqueries, `WHERE` / `ORDER BY` / `LIMIT` |
| Microsoft Dynamics | A FetchXML document — so this cell is **XML**, not JSON |
| HubSpot | A **JSON** object: a list of selected fields, the module/object name, and a `Where` string that is itself a JSON array of filter clauses (section 12) |
| Monday.com | GraphQL query text |
| Amazon Seller | A plain object name |
| **Any CRM** | A value beginning with `https` switches the process into **webhook mode**: the service stops polling and instead drains deliveries for the registration identified by the `id` query-string parameter in that URL |

So before you edit `Query`, read the existing value and match its shape. Pasting SOQL into a
process whose handler expects JSON does not fail loudly at edit time — it fails at the next run.

```bash
dlake tool query --sql "SELECT TxDownloaderProId, CRMName, Query
                        FROM DLO.TxDownloaderPro WHERE TxDownloaderProId = 12"
```

(Substitute your tenant's active schema for `DLO`.)

### The supporting tables

| Table | Operator's view |
|---|---|
| `TxDownloaderProBlockDuplicateId` | Holds process ids (`TxBlockProcessID`) whose records must not be re-imported once a record with the same `UniqueID` already exists. Adding a row here is a suppression switch; removing one re-opens re-import |
| `TimeStampRepository` | Key/bookmark rows — a name and an id value — written by some handlers as a cursor. Read it to see where a handler thinks it is. Editing it moves a cursor: treat as a deliberate, one-off intervention |
| `TimeStampRepositoryHistory` | History alongside the above. Read-only in practice |

Per-customer configuration also lives as key/value rows in `CommercientFlags`, and errors land in
`Commercient_Error_Log`. Both are outside the gateway view set (section 2) — check with
`dlake admin list_views` before assuming you can reach either through the Data API. Where they
are not exposed, `dlake tool query` against the `dbo` objects is still the read path.

**`CommercientFlags` holds credentials.** The flag rows are how the service decides which CRM it
is talking to, and several of them are API keys, client secrets and access tokens (stored
encrypted). Do not dump the whole table into a ticket, a log or a chat window. Read named flags,
not `SELECT *`.

## 10. CRUD on the transaction tables

`TxDownloaderProTrans` is the state machine. One row is one record in flight. This is where you
diagnose "the order never arrived" — and where you can do real damage.

### The columns

| Column | Meaning |
|---|---|
| `TxDownloaderProTransId` | Identity key of the transaction row |
| `TxDownloaderProId` | Which configuration row (section 9) produced it |
| `UniqueID` | The CRM-side record identifier. Duplicate suppression keys on this |
| `Version` | Version counter carried on the row |
| `Object` | The CRM object/module name the row came from |
| `XMLResult` | **XML.** The retrieved CRM record (section 11) |
| `SFUpdated` | The state value. See below — handle with care |
| `SFMessage` | Status text. Set to an "imported" message when the row is first written |
| `IsERPCompleted` | Whether the source system has finished with the row. Half of every state transition's condition |
| `IsSuccess` | Outcome marker |
| `JsonRequest` | **JSON.** Request payload |
| `ERPResponse` | **JSON.** The source system's response — the substitution source for the writeback mapping (section 11) |
| `JsonResponse` | **JSON.** The shaped writeback instruction the service reads when pushing to the CRM (section 11) |
| `CreateDate`, `ERPToCommercientDate`, `CommercientToSFDate` | Timestamps for each leg |
| `LogInformation` | Per-row log text |
| `IsRecordReadyToProcess` | Readiness marker |
| `IsRepeat` | When set, the record is eligible to be downloaded again for processing |

### `SFUpdated` — read it, do not redefine it

| Value | Meaning |
|---|---|
| `0` | Imported from the CRM, not yet acknowledged back to the CRM. This is the value every new row is written with |
| `1` | Acknowledged; ready to push to the CRM |
| `2` | Synced to the CRM. Terminal for the push |
| `3` | A suppression sentinel. The import path refuses to insert a new row for a `UniqueID` that already has a row at `3`. **Nothing in the writeback service writes `3`** — it is set elsewhere, so treat a `3` as a deliberate block and establish why before clearing it |
| `4` | On hold — the row was pulled aside during the push and is waiting for the retry step |

The transitions the service makes, in the order it makes them: `0 → 1` (acknowledge back to the
CRM), `1 → 4` (put aside), `4 → 2` (retry succeeded), `1 → 2` (main push succeeded). The order is
load-bearing: holds are taken before retries, and retries before the main push, so the main push
never touches a row that is still on hold.

> **The standing warning:** these columns and their state
> values are load-bearing. Renaming a column breaks every handler. **Changing the meaning of an
> `SFUpdated` value silently corrupts sync state** — there is no error, records simply move
> wrongly. Exposing the entity through the Data API makes it writeable; that does not make
> hand-editing it safe.

### What inspection looks like

State breakdown for a tenant — the first thing to run when writeback "stopped":

```bash
dlake tool query --sql "SELECT SFUpdated, IsERPCompleted, COUNT(*) AS Rows
                        FROM DLO.TxDownloaderProTrans
                        GROUP BY SFUpdated, IsERPCompleted
                        ORDER BY SFUpdated"
```

A pile at `4` means the retry step is not clearing holds. A pile at `0` means the acknowledge step
is not reaching the CRM. A pile at `1` with `IsERPCompleted` false means the source system has not
finished with those rows — the problem is upstream of TxDownloaderPro, not in it.

One record's history:

```bash
dlake tool read_records --entity TxDownloaderProTrans \
    --filter "UniqueID eq 'DEMO-000123'" \
    --select "TxDownloaderProTransId,TxDownloaderProId,SFUpdated,IsERPCompleted,SFMessage,CreateDate" \
    --orderby "['CreateDate desc']" --first 20
```

### Correcting state — the narrow, deliberate cases

There are only a few edits that are defensible, and all of them are one row at a time:

```bash
dlake tool update_record --entity TxDownloaderProTrans \
    --keys   '{"TxDownloaderProTransId": 4711}' \
    --fields '{"SFUpdated": 1}'
```

| Situation | Defensible edit | Why |
|---|---|---|
| A row stuck at `4` whose conflict has been resolved by hand in the CRM | Set `SFUpdated` back to `1` so the main push picks it up | Returns the row to the normal path rather than inventing a state |
| A row that must be re-downloaded | Set `IsRepeat` | The service already has a path for this |
| A row that must never be pushed | Leave it. Deactivate the process instead | Editing state to suppress one row is how meanings drift |

And the edits to refuse outright:

- **Do not invent a new `SFUpdated` value**, and do not reuse an existing one for a new purpose.
- **Do not bulk-move rows between states.** The verbs are single-row for a reason.
- **Do not set `SFUpdated` to `2` to "clear a backlog".** `2` asserts the CRM was updated. If it
  was not, the record is silently lost and nothing will ever retry it.
- **Do not hand-edit `XMLResult`, `JsonRequest`, `JsonResponse` or `ERPResponse` on a row already
  in flight.** Fix the mapping on the configuration row (section 11) and let the next run produce
  a correct payload.
- **Do not delete transaction rows to tidy up.** The service has its own housekeeping step, and
  duplicate suppression reads existing rows: deleting one can cause a re-import you did not want.

## 11. Field mapping: the JSON and XML columns

Field mapping in TxDownloaderPro is **template substitution over a document**, in both directions.
Knowing which column is JSON and which is XML is most of the job.

| Column | Table | Format | What it holds |
|---|---|---|---|
| `XMLResult` | `TxDownloaderProTrans` | **XML** | The retrieved CRM record, converted to XML and wrapped under a single `result` root element |
| `ERPResponse` | `TxDownloaderProTrans` | **JSON** | The source system's response object — the value source for writeback templates |
| `JsonRequest` / `JsonResponse` | `TxDownloaderProTrans` | **JSON** | The request and the shaped writeback instruction |
| `ProcessStructure` | `TxDownloaderPro` | **JSON** | Template paths read against `XMLResult` |
| `ResultStructure` | `TxDownloaderPro` | **JSON** | The four-part writeback mapping, filled from `ERPResponse` |
| `Query` | `TxDownloaderPro` | **JSON, XML or plain text** | Depends on the CRM — see section 9 |

### Inbound: the XML document

Every retrieved CRM record is converted from the CRM's JSON to XML and stored under one root
element named `result`. So the document for a record of object `DemoOrder` looks like this
(entirely synthetic):

```xml
<result>
  <DemoOrder>
    <Id>a0X000000DEMO01</Id>
    <Demo_Ref__c>DEMO-000123</Demo_Ref__c>
    <Customer>
      <Code>CUST-0001</Code>
    </Customer>
    <Lines>
      <Sku>WIDGET-1</Sku>
      <Qty>2</Qty>
    </Lines>
    <Lines>
      <Sku>WIDGET-2</Sku>
      <Qty>5</Qty>
    </Lines>
  </DemoOrder>
</result>
```

Two properties of that shape are load-bearing and catch people out:

- **The root is always `result`.** Template paths are written as `{{DemoOrder.Demo_Ref__c}}` —
  the root is *not* part of the path, the resolver supplies it.
- **A child collection is a repeated element, not an array element.** Two lines means two
  `<Lines>` elements side by side. There is no index in the path.

Read one document:

```bash
dlake tool query --sql "SELECT XMLResult FROM DLO.TxDownloaderProTrans
                        WHERE TxDownloaderProTransId = 4711"
```

### Template paths

A mapping value is a string containing one or more `{{Path.To.Field}}` placeholders. Resolution is
**dotted traversal, and nothing else**:

- Each dot-separated segment is looked up in turn from the document root.
- A segment that does not exist resolves the whole placeholder to the **empty string**. It does
  not error, and it does not leave the placeholder in place.
- Text around the placeholder is preserved, so `REF-{{DemoOrder.Demo_Ref__c}}` is a valid value.

**That silent empty string is the most common field-mapping mistake.** A mistyped path and a genuinely
blank CRM field produce byte-identical output. Verify a path by reading the actual `XMLResult` of
a real transaction row, not by reading the CRM's field list.

### Outbound: the four-part writeback mapping

`ResultStructure` is a JSON object with up to four parts. Each part is optional; omit the ones you
do not need. Synthetic example:

```json
{
  "Part1": {
    "Demo_Status__c":  "{{status}}",
    "Demo_DocNum__c":  "{{document.number}}"
  },
  "Part2": {
    "ObjectAPIName":     "DemoOrderLine",
    "LoopFieldTagName":  "lines",
    "LoopFieldIDName":   "lineId",
    "FieldName": { "{{lines.price}}": "Demo_Price__c" }
  },
  "Part3": {
    "ObjectAPIName":            "DemoNote",
    "NewObjectExternalIDField": "Demo_ExtId__c",
    "FieldName": { "{{document.number}}": "Demo_Ref__c" }
  },
  "Part4": {
    "ObjectAPIName":         "DemoAudit",
    "AnotherObjectFieldId":  "{{document.id}}",
    "FieldName": { "{{status}}": "Demo_Result__c" }
  }
}
```

| Part | What it does |
|---|---|
| `Part1` | Update fields on the record the run is already working with |
| `Part2` | Update the child/line records under it. `ObjectAPIName` names the CRM child object; `LoopFieldTagName` and `LoopFieldIDName` locate the repeating collection and its id |
| `Part3` | Create a **new** record, matched on `NewObjectExternalIDField` |
| `Part4` | Update a **different** record, addressed by `AnotherObjectFieldId` |

The `FieldName` map is written **source-path first, CRM-field second**. Getting that round the
wrong way produces a mapping that resolves to nothing — the same silent empty string as above.
`ObjectAPIName` and the id fields are themselves template-resolved, so they may contain `{{ }}`
placeholders.

Edit one safely — read it, change it in a file, write it back:

```bash
dlake tool query --sql "SELECT ResultStructure FROM DLO.TxDownloaderPro WHERE TxDownloaderProId = 12"
#   ... save to result-structure.json, edit, validate it is still valid JSON ...
dlake tool update_record --entity TxDownloaderPro \
    --keys '{"TxDownloaderProId": 12}' --fields @fields.json
```

where `fields.json` is `{ "ResultStructure": "<the JSON, as a string>" }`. The column is text: the
JSON goes in as a **string value**, not as a nested object. Take a copy of the old value before
you overwrite it — there is no version history on this column.

## 12. What you can do to retrieved CRM data before it reaches the source

This is the question people mean when they ask what "functions" are available. The honest answer
in two parts:

> **There is no user-invokable transformation function library.** No `UPPER()`, no `TRIM()`, no
> `CONCAT()`, no date formatting, no default-value fallback, no conditional expression. The
> template mechanism (section 11) is dotted path substitution only. Anyone who tells you they
> wrote `{{UPPER(Account.Name)}}` in a mapping wrote a path that resolves to empty string.

What **is** available, and is genuinely configurable per process, is a **filter-operator
vocabulary** — a set of named comparisons evaluated against each retrieved record, deciding
whether that record is written at all — plus a small set of structural operations the service
performs on the way in.

### 12a. Filter operators (the general set)

These are the operators of the JSON filter clauses carried in the `Where` part of a `Query`
(section 9). Each clause is an object; all clauses in the array must pass or the record is not
imported. They are evaluated against the retrieved record's XML.

Clause shape:

| Key | Meaning |
|---|---|
| `propertyName` | Dotted path to the value inside the retrieved document. A missing path fails the clause — **except** under `NOT_HAS_PROPERTY`, which it satisfies |
| `operator` | One of the names below. Matched case-insensitively |
| `value` | The comparison value |
| `highValue` | Upper bound. `BETWEEN` only |
| `values` | Array form of the comparison value. Used by `IN` / `NOT_IN`; joined into a comma-separated list before comparison |

| Operator | What it does | Arguments | Comparison |
|---|---|---|---|
| `EQ` | Equal | `value` | String, case-insensitive |
| `NEQ` | Not equal | `value` | String, case-insensitive |
| `GT` | Greater than | `value` | Numeric |
| `GTE` | Greater than or equal | `value` | Numeric |
| `LT` | Less than | `value` | Numeric |
| `LTE` | Less than or equal | `value` | Numeric |
| `BETWEEN` | Within an inclusive range | `value` (low) **and** `highValue` (high) | Numeric |
| `IN` | Value is one of a list | `values` array, or `value` as a comma-separated string | String, case-insensitive, list items trimmed |
| `NOT_IN` | Value is not one of a list | as `IN` | String, case-insensitive |
| `HAS_PROPERTY` | The field is present and non-empty | none | Presence |
| `NOT_HAS_PROPERTY` | The field is absent or empty | none | Presence |
| `CONTAINS_TOKEN` | The field contains the substring | `value` | Substring, case-insensitive |
| `NOT_CONTAINS_TOKEN` | The field does not contain the substring | `value` | Substring, case-insensitive |

Behaviour at the edges — all of it worth knowing before you write a filter:

- **A numeric operator on a non-numeric value fails the clause.** `GT`, `GTE`, `LT`, `LTE` and
  `BETWEEN` parse both sides as numbers; if either side does not parse, the answer is "no match"
  and the record is filtered out. An empty CRM field is not a number.
- **An unrecognised operator name fails the clause too** — it does not error, it does not pass.
  A typo in an operator name silently stops the process importing anything.
- **`IN` / `NOT_IN` compare as text**, on a comma-separated list. A value that itself contains a
  comma cannot be expressed.
- Only the operators in that table exist. There is no regex, no `STARTS_WITH`, no null-versus-empty
  distinction, no case-sensitive variant.

Example clause array (synthetic), as the string stored in the `Where` part of `Query`:

```json
[
  { "propertyName": "DemoOrder.Status",           "operator": "IN",         "values": ["Approved", "Released"] },
  { "propertyName": "DemoOrder.Total",            "operator": "GTE",        "value": "100" },
  { "propertyName": "DemoOrder.Customer.Code",    "operator": "HAS_PROPERTY" }
]
```

### 12b. Filter operators (Monday.com board rules)

Monday.com processes use a **different, board-native vocabulary**, because the values are typed by
the board column rather than being text. Rules are grouped with a logical `and` or `or`. Do not mix
the two vocabularies: a `Where`-style `EQ` in a Monday rule is unrecognised and evaluates false.

The column's type decides which operators apply:

| Board column types | Treated as |
|---|---|
| Numbers, Rating, AutoNumber, Progress, TimeTracking, Vote, ItemID | Number |
| Date, Timeline, CreationLog, LastUpdated | Date |
| Checkbox | Boolean |
| Formula | Number if the value parses as one, otherwise text |
| everything else | Text |

| Operator | Number | Date | Boolean | Text | Argument |
|---|---|---|---|---|---|
| `greater_than` | yes | yes | — | — | scalar |
| `greater_than_or_equals` | yes | yes | — | — | scalar |
| `lower_than` | yes | yes | — | — | scalar |
| `lower_than_or_equals` | yes | yes | — | — | scalar |
| `between` | yes | yes | — | — | two-element array `[low, high]`, inclusive |
| `any_of` | yes | yes | — | yes | array |
| `not_any_of` | yes | yes | — | yes | array |
| `is_empty` | — | yes | yes | yes | none |
| `is_not_empty` | — | yes | yes | yes | none |
| `contains_text` | — | — | — | yes | scalar, case-insensitive |
| `not_contains_text` | — | — | — | yes | scalar, case-insensitive |

- **A rule naming a column the item does not have fails.**
- **On a checkbox, `is_empty` means unticked and `is_not_empty` means ticked.**
- **On a date, `is_empty` is always false and `is_not_empty` is always true** — dates are never
  treated as empty. Do not use them to find records with no date.
- Any operator outside the table, or applied to the wrong type, evaluates false.

### 12c. Structural operations on the way in

These are not operators you invoke; they are what the service does to a retrieved record, and each
is steered by a column you can edit.

| Operation | Steered by | What it does |
|---|---|---|
| **JSON-to-XML conversion** | none — always | Converts the retrieved CRM record to XML under a single `result` root, and stores it in `XMLResult`. This is the only transformation applied to the record body |
| **Token selection** | `DataTag` on a webhook registration | Selects which part of an incoming payload to import, as a plain key or a dotted/nested path. If it selects an **array**, one transaction row is written per element. If the path is not found, the whole body is imported instead |
| **Unique-ID resolution** | `UniqueIDFieldName` | Names the field, as a path, whose value becomes the row's `UniqueID`. When it is blank or resolves to nothing, the delivery's own log identifier is used instead — with an index suffix per element when the payload was an array |
| **Provenance injection** | none — always, for object payloads | The originating registration and delivery identifiers are added into the record before conversion, so they are visible in the resulting XML. Payloads that are not objects have nowhere to attach them and are stored as-is |
| **Duplicate suppression** | `TxDownloaderProBlockDuplicateId` | A record whose `UniqueID` already exists is not re-inserted when its process is listed in the block table, or when an existing row for that `UniqueID` sits at `SFUpdated = 3` |
| **Re-download eligibility** | `IsRepeat` on the transaction row | Makes a record eligible to be downloaded again for processing |

And the safety net worth knowing: **if the payload cannot be parsed at all, the service falls back
to importing the whole body** rather than dropping the delivery. A row you did not expect, with a
whole-payload document and a log-identifier `UniqueID`, is that fallback firing — read it as
"the `DataTag` or the payload shape is wrong", not as a lost record.

### 12d. Where the real transformation happens

Because there is no transformation library, anything beyond "select a path and filter on it" has
to happen somewhere else. The three honest options, in order of preference:

1. **In the query.** Shape the data on the CRM side — the SOQL / FetchXML / GraphQL / JSON query
   in `Query` is the most expressive tool you have, and it runs before anything is stored.
2. **In the filter clauses.** Use section 12a/12b to decide what is worth importing at all.
3. **In the source system.** The record arrives as XML under a `result` root; mapping it into the
   ERP's own shape is the ERP side's job, and that is where value manipulation belongs.

## 13. Where this sits

Setting up an integration is a longer journey and this is one step of it:

| Skill | Covers |
|---|---|
| `dlake-integration-setup` | Registering a tenant, verification, seeding, then the wizard — server IP, CRM choice and OAuth, ERP connector and provisioning |
| **`dlake-txdownloaderpro`** (this) | The writeback leg's Data API surface and its operation: exposing the TxDownloaderPro objects, scoping a key to them, then CRUD on the configuration and transaction rows, field mapping, and the filter operators |
| `dlake` | Operating an existing tenant generally — schema, queries, exports, keys, the REST/GraphQL contract |

Do this **after** the tenant exists and has been seeded — the gateway views are created at seed
time, so there is nothing to expose before then.

---

## 14. Configuring it through its own API — the `txdownloaderpro_*` tools

Sections 1–12 reach the writeback tables as ROWS, over the Data API. That is the right tool for
inspecting and correcting in-flight state. It is the wrong tool for *configuring* writeback,
because the product's own rules — the unique process name, the ERP process id the DLL column
really wants, the webhook row that must be created in the same transaction, the mapping
validation, the circular-sync warning — live in the configuration API, not in the table.

The control plane carries a dedicated **`txdownloaderpro_*`** tool set for exactly that, and it
is the **preferred way to configure TxDownloaderPro**. Everything runs from anywhere, over the
tenant's API key:

```bash
dlake admin <tool> --profile <tenant> [--arg value ...]
dlake admin list | grep txdownloaderpro_        # the whole group, with schemas
dlake admin txdownloaderpro_create_process --help
```

`--profile` is **always explicit**. There is deliberately no default profile — naming the tenant
on every call is what stops a command landing on the wrong customer.

**Every `txdownloaderpro_*` tool is Admin-only, reads included.** The key you call with must
belong to a tenant user who holds the **Admin** role; a key minted from a non-admin user is
refused with a message naming that role. Keys inherit the roles of the user who minted them, so
mint operating keys from an Admin account — a refusal here is an account question, not a scope
you can widen after the fact. The same rule governs the `crmpro_*` and `registration_*` groups.

No `txdownloaderpro_*` tool takes a user id: the platform resolves the customer from the key, so
a key can only ever drive its own tenant.

### Reads — always safe

| Tool | What it does | Key args |
|---|---|---|
| `txdownloaderpro_list_processes` | Every Phase 2 process with its 30-day unresolved-error count — the grid | — |
| `txdownloaderpro_get_process` | One process in its edit shape, including the four webhook fields | `processId` |
| `txdownloaderpro_sync_status` | The master switch. Pure read | `oldPortal` (default `false`) |
| `txdownloaderpro_resolve_erp` | An ERP name or alias to its catalogue `erpId` | `erpName` |
| `txdownloaderpro_create_options` | Everything a create needs: resolved ERP + process versions + DLL list | `erpName` \| `erpId` |
| `txdownloaderpro_erp_flags` | Every flag defined for the ERP with this customer's value | `erpId` |
| `txdownloaderpro_destination_points` | The customer's Generic-API destination points | — |
| `txdownloaderpro_field_mapping` | The Process Mapping grid | `processId` |
| `txdownloaderpro_result_mapping` | The stored Result Mapping (null data = not configured) | `processId` |
| `txdownloaderpro_additional_files` | The ERP's optional files, each marked enabled | `processId`, `erpId` |
| `txdownloaderpro_circular_sync_check` | Names CRM objects that ALSO have an active Phase 1 config | `processId` |
| `txdownloaderpro_source_fields` | The source fields a mapping can draw on (live CRM) | `processId` |
| `txdownloaderpro_preview_xml` | Runs the SAVED query and renders one record as the sync engine's XML | `processId` |
| `txdownloaderpro_test_query` | Tests a query you have NOT saved yet (live CRM) | `query`, `processId` |
| `txdownloaderpro_destination_systems` | The shared destination-system catalogue | — |
| `txdownloaderpro_destination_actions` | The actions on one destination point | `destinationPoint` |
| `txdownloaderpro_destination_schema` | The bound action's API config + simplified schema | `processId` |

### Mutations — these change what the next run does

| Tool | What it does | Key args |
|---|---|---|
| `txdownloaderpro_create_process` | Create a process. **This is what flips a customer to "Phase 2 ready."** | `processName`, `erpProcessId` + see `--help` |
| `txdownloaderpro_update_process` | Change a process. **Read-merge-write**: send only the fields you are changing and everything else is preserved | `processId`, `fields`, `erpProcessId` \| `erpId` |
| `txdownloaderpro_delete_process` | Delete a process AND its transaction log | `processId`, `confirm` |
| `txdownloaderpro_set_sync_enabled` | The master switch, by VALUE. Reports before / after / changed | `enabled`, `oldPortal` |
| `txdownloaderpro_set_process_flag` | One grid checkbox, **idempotently** | `processId`, `fieldName`, `enabled` |
| `txdownloaderpro_save_erp_flags` | Save the ERPFlag screen. Reads first and merges, so an unnamed flag keeps its value | `erpId`, `flags` |
| `txdownloaderpro_save_field_mapping` | Replace the Process Mapping — **complete row set** | `processId`, `rows`, `confirm` |
| `txdownloaderpro_save_result_mapping` | Replace the Result Mapping — **complete document** | `processId`, `structure`, `confirm` |
| `txdownloaderpro_delete_result_mapping` | Clear the Result Mapping | `processId`, `confirm` |
| `txdownloaderpro_save_additional_files` | Replace the enabled-file set. Nothing checked CLEARS it | `processId`, `files`, `confirm` |

### The four traps these tools defuse for you

1. **The save is a whole-row rewrite.** A partial post to the underlying endpoint blanks the
   process name, the CRM binding, the destination ids and every webhook field — including the
   plaintext `SecurityKey` a running webhook sync depends on. `txdownloaderpro_update_process`
   reads the record, overlays only what you sent, and posts the complete entity.
2. **The ERP process is read as a NAME and written as an ID.** The same save refuses anything
   but an `ERPProcessId` in that column, so pass `erpProcessId` when you are changing which ERP
   process runs, or `erpId` so the tool can resolve the stored name back to its id. Supply
   neither and it refuses *before* writing rather than failing halfway.
3. **The per-process checkbox endpoint is a blind toggle** — it flips the stored value and
   ignores what you posted, so a retry after an ambiguous failure flips the customer's process
   the wrong way. `txdownloaderpro_set_process_flag` reads first and flips only on a genuine
   mismatch. (`IsStreaming` is accepted by that endpoint but is not returned by the read, so it
   cannot be verified and is refused here rather than toggled blind.)
4. **A create needs three catalogue answers.** `txdownloaderpro_create_options` composes them:
   the resolved ERP, its process versions (the seed defaults) and its DLL list (where
   `erpProcessId` comes from).

### Two more things worth knowing

- Turning the master switch ON stores the literal `separated`, **not** `true` — writing `true`
  would silently move the customer onto the old combined code path. The tool writes it; the raw
  value is never an argument.
- `txdownloaderpro_test_query` treats **zero rows as a PASS**. `invalid_query` carries the CRM's
  own rejection text (that is the answer you want); `crm_unavailable` means their CRM is down or
  the credential expired, which is an outage, not a bad query.

Four Phase 2 endpoints stay REST-only on purpose: registering a shared destination-action
template (admin plane, service key only), the raw single-process read (a subset of
`txdownloaderpro_get_process`), the process-structure preview, and the line-item template lookup.

---

## Quick reference

| Tool | Arguments | Purpose |
|---|---|---|
| `list_exposed_entities` | *(none)* | Current Data API scope, with `served` vs persisted |
| `set_entity_exposure` | `entity`\*, `expose`\*, `keyFields`, `confirm` | Add/remove one entity. Does **not** restart DAB |
| `create_api_key` | `keyName`\*, `expirationDays`\*, `targetUserId`, `project`, `scope`, `allowRawSql` | Mint a key; raw value shown once |
| `list_api_keys` | `project` | Key metadata only — never secrets |
| `set_key_entity_scopes` | `apiKeyId`\*, `scope`\*, `allowRawSql` | **Replace** a key's per-entity scope |
| `get_key_rights` | `apiKeyId`\* | Inherited baseline + current removals |
| `set_key_rights` | `apiKeyId`\*, `suppressOwner`, `suppressDboAdmin`, `deniedPermissionKeys` | Rights **removals** only |
| `restart_dab` | `confirm`\* | Regenerate config + restart. Brief interruption |

\* = required.

### Operating (data plane)

| Tool | Arguments | Purpose |
|---|---|---|
| `read_records` | `entity`\*, `filter`, `select`, `orderby`, `first`, `after` | Inspect configuration or transaction rows |
| `create_record` | `entity`\*, `data`\* | Add a configuration row |
| `update_record` | `entity`\*, `keys`\*, `fields`\* | Correct one row, addressed by key |
| `delete_record` | `entity`\*, `keys`\* | Remove a row. Prefer `IsActive: false` on configuration |
| `query` | `sql`\* | Read-only `SELECT` — state breakdowns, full text columns, joins |

\* = required. `query` needs schema-qualified table names; the record verbs take the bare entity name.

### The mapping and filter surface at a glance

| Where | Format | Carries |
|---|---|---|
| `TxDownloaderPro.Query` | JSON, XML, GraphQL or plain text — **per CRM** | What to fetch, and the `Where` filter clauses |
| `TxDownloaderPro.ProcessStructure` | JSON | `{{Object.Field}}` paths read against the retrieved XML |
| `TxDownloaderPro.ResultStructure` | JSON | The four-part writeback mapping |
| `TxDownloaderProTrans.XMLResult` | XML under a `result` root | The retrieved CRM record |
| `TxDownloaderProTrans.ERPResponse` | JSON | Value source for writeback templates |
| `TxDownloaderProTrans.JsonRequest` / `.JsonResponse` | JSON | Request, and the shaped writeback instruction |

## Things that bite

- **Changing what an `SFUpdated` value means.** `TxDownloaderProTrans` is the sync state machine:
  its `SFUpdated` values (`0` / `1` / `2` / `4`) plus `IsERPCompleted` and
  `IsDoNotUpdateFieldsInSF` are what every writeback handler steers on. These columns and state
  values are load-bearing: renaming a column breaks all handlers, and **changing the meaning of an
  `SFUpdated` value silently corrupts sync state** — no error, just wrong records moving. Exposing
  the entity through the Data API makes it
  *writeable*. Read it freely; do not hand-edit state values, and do not "tidy" the columns.
- **Conflating exposure with key scope.** Exposing an entity publishes it tenant-wide; scoping a
  key restricts *that key* to a subset. Both must line up. A key scoped to an unexposed entity
  reaches nothing, and exposing an entity narrows no key.
- **Forgetting `restart_dab`.** Exposure and scope changes are persisted intent. Until a restart
  the container serves the old config — the entity is listed but `served:false` and reads answer
  `EntityNotFound`.
- **Restarting after every single change.** It interrupts the live Data API each time. Batch the
  exposure and key work, then restart once, when the operator is ready.
- **Forgetting the restart after minting a *scoped* key.** Correct key, correct scope, `403`. The
  per-key role only enters DAB's config on regeneration.
- **Sending a partial `scope` to `set_key_entity_scopes`.** It replaces, not merges — omitted
  entities are dropped.
- **Trying to restart DAB with the writeback key.** Scoped keys are locked out of the admin plane;
  use the full-scope admin key.
- **Qualifying the entity name.** Expose the working-schema view by its bare name; `dbo.`-prefixed
  names are not what the scope takes.
- **Assuming every tenant has all five objects.** A view exists only where the gateway table does.
  Check `list_views` first.
- **Assuming `CommercientFlags` or `Commercient_Error_Log` have working-schema views.** They are
  not part of the gateway view set. Verify before planning around them.

### Operating traps

- **Expecting transformation functions that do not exist.** There is no `UPPER`, `TRIM`,
  `CONCAT`, date format, default-value or conditional syntax in a mapping. `{{Path.To.Field}}`
  is dotted path substitution and nothing more. Shape the data in the query or in the source
  system instead.
- **A mistyped template path is indistinguishable from an empty field.** An unresolvable
  `{{ }}` becomes the empty string, silently. Validate every path against a real `XMLResult`
  from a real transaction row before you trust a mapping.
- **Forgetting the `result` root.** Paths are written from the object down —
  `{{DemoOrder.Field}}` — because the resolver supplies the root. Including `result` in the path
  makes it unresolvable, which means empty string, which means silent.
- **Writing `FieldName` the wrong way round.** In the writeback mapping the key is the source
  path and the value is the CRM field. Reversed, it resolves to nothing.
- **Assuming `Query` has one format.** It is SOQL on one CRM, FetchXML on another, a JSON object
  on a third, GraphQL on a fourth — and a value starting with `https` switches the process into
  webhook mode entirely. Read the existing value before you replace it.
- **A typo in a filter operator name silently imports nothing.** An unrecognised operator
  evaluates false, per clause, per record. So does a numeric operator applied to a value that
  does not parse as a number — including an empty one.
- **Mixing the two filter vocabularies.** The general `EQ` / `GT` / `CONTAINS_TOKEN` set and the
  Monday.com `any_of` / `contains_text` set are not interchangeable. Wrong vocabulary, false,
  no import.
- **Using `is_empty` on a Monday date column to find records with no date.** It is always false
  there, and `is_not_empty` is always true.
- **Hand-editing `SFUpdated` to clear a backlog.** Setting rows to `2` asserts the CRM was
  updated. If it was not, those records are lost and nothing retries them. The only routinely
  defensible move is returning a resolved hold from `4` to `1`.
- **Treating a `3` as junk.** Nothing in the writeback service writes it, and its presence blocks
  re-import of that `UniqueID`. Find out who set it before you clear it.
- **Bulk-editing transaction state.** `query` is read-only and the record verbs are single-row.
  That is the guardrail, not an inconvenience to script around.
- **Editing payload columns on a row already in flight.** Fix the mapping on the configuration
  row and let the next run produce a correct payload.
- **Deleting a configuration row instead of deactivating it.** Transaction rows reference it by
  `TxDownloaderProId`. Set `IsActive: false`, watch a run, then decide.
- **Deleting transaction rows to tidy up.** Duplicate suppression reads existing rows, so a
  deletion can trigger a re-import you did not want. The service has its own housekeeping step.
- **Overwriting a mapping column with no copy of the old value.** There is no version history on
  `ProcessStructure`, `ResultStructure` or `Query`. Read and save before you write.
- **Passing a mapping as a nested object.** The mapping columns are text — the JSON goes in as a
  **string** value.
- **Dumping `CommercientFlags`.** Those rows include API keys, client secrets and access tokens.
  Read named flags; never paste the table into a ticket or a chat.
