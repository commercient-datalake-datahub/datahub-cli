---
name: dlake-txdownloaderpro-setup
description: >-
  Set up the TxDownloaderPro writeback leg of a Commercient integration with the `dlake` CLI
  (npm `@commercient/dlake`): make the gateway sync-state objects — `TxDownloaderPro`,
  `TxDownloaderProTrans`, `TxDownloaderProBlockDuplicateId`, `TimeStampRepository`,
  `TimeStampRepositoryHistory` — reachable through the tenant's Data API, then scope an API key
  to them and apply it with a single deliberate DAB restart. Use this skill whenever the task
  mentions TxDownloaderPro, writeback, CRM-to-ERP sync state, `SFUpdated`, or "expose the
  TxDownloaderPro tables / add them to a key". This is ONE STEP of standing up an integration —
  for registration, CRM choice and the ERP connector use `dlake-integration-setup`; for general
  tenant operation use `dlake`.
---

# Setting up TxDownloaderPro writeback with `dlake`

A Commercient integration has two legs. The **inbound** leg pulls the customer's ERP data into
the Data Lake. **TxDownloaderPro** is the **writeback** leg going the other way: it picks up
records a user flagged in the CRM, carries them back toward the source/ERP system, and reports
the outcome back to the CRM.

This skill covers the Data Lake side of that: making TxDownloaderPro's state objects reachable
through the tenant's Data API and giving a key access to them. It does **not** cover installing
or scheduling the TxDownloaderPro service itself, or its per-customer configuration — those are
handled outside the `dlake` CLI.

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

**Write the name in full.** TxDownloaderPro is its own product — not an edition or a nickname of
TxDownloader, which is a different thing entirely. Shortening it in a ticket, a commit message or a
prompt points at the wrong product, and the tenant tables here are the only place the two look
alike.

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
writeback state. Do not describe exposure as a precondition for the service to run; it is a
precondition for **you** to operate it through the Data Lake.

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

> MCP clients cache `tools/list` per session. If `keyFields` is missing from the schema your
> client shows, reconnect — direct JSON-RPC callers can pass it immediately.

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

## 8. Where this sits

Setting up an integration is a longer journey and this is one step of it:

| Skill | Covers |
|---|---|
| `dlake-integration-setup` | Registering a tenant, verification, seeding, then the wizard — server IP, CRM choice and OAuth, ERP connector and provisioning |
| **`dlake-txdownloader-setup`** (this) | The writeback leg's Data API surface: exposing the TxDownloaderPro objects and scoping a key to them |
| `dlake` | Operating an existing tenant generally — schema, queries, exports, keys, the REST/GraphQL contract |

Do this **after** the tenant exists and has been seeded — the gateway views are created at seed
time, so there is nothing to expose before then.

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

## Things that bite

- **Changing what an `SFUpdated` value means.** `TxDownloaderProTrans` is the sync state machine:
  its `SFUpdated` values (`0` / `1` / `2` / `4`) plus `IsERPCompleted` and
  `IsDoNotUpdateFieldsInSF` are what every writeback handler steers on. The product's own contract
  notes warn that these columns and state values are load-bearing, that renaming a column breaks
  all handlers, and that **changing the meaning of an `SFUpdated` value silently corrupts sync
  state** — no error, just wrong records moving. Exposing the entity through the Data API makes it
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
