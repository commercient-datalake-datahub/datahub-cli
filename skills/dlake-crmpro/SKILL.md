---
name: dlake-crmpro
description: >-
  Operate **CRMPro**, Commercient's forward sync agent, through the `dlake` CLI (npm
  `@commercient/dlake`) — remotely, with the `crmpro_*` admin tools (`dlake admin crmpro_list_processes`,
  `crmpro_get_process`, `crmpro_update_process`, `crmpro_set_sync_enabled`, `crmpro_field_mapping`,
  `crmpro_sync_history`, `crmpro_errors`, `crmpro_flags`, `crmpro_templates` and the rest): list and
  edit sync processes, enable or disable sync, map fields, apply templates and diagnose a run from
  anywhere. It also covers the underlying tables when a tool does not reach them — `CRM_Configuration`,
  `CRM_FieldList`, the `CommercientFlags` flag table, `CRM_Parameters`, the Connection Manager records,
  and the per-run tables `TimeStampRepository`, `CRMProRunHistory`, `CRMPRO_DASHBOARD_ERROR`,
  `CRMPRO_ERROR_LOG`, `CRMPRO_DeleteRecordInfo`, `CRMBackupObjectList`. Use this skill whenever the task
  mentions CRMPro, a `crmpro_` tool, the forward/ERP-to-CRM leg, a sync process or config row, a
  `Sync_Order` / `Sync_Operation_Type` / `TimeStamp_Prefix` value, a CRMPro flag, or field mapping
  through the JSON- and XML-bearing columns. For the writeback leg use `dlake-txdownloaderpro`;
  for standing an integration up use `dlake-integration-setup`; for general tenant operation use `dlake`.
---

# Operating CRMPro with `dlake`

**CRMPro** is Commercient's **forward** sync engine. It reads a customer's ERP data out of their
gateway database and pushes it into the CRM or e-commerce platform, covering the whole lifecycle:
schema creation in the destination, incremental change detection by timestamp, transformation,
destination API calls, error logging and summary reporting. It runs **per customer on a schedule**,
and supports **16+ destination platforms** (CRM and e-commerce alike).

Its counterpart is **TxDownloaderPro**, which runs the other way — CRM back toward the source. They
are **different products**. Write both names in full, always; abbreviating either points at the wrong
thing, and their tables sit side by side in the same database.

**CRMPro is flag-driven.** Runtime behaviour comes from the flag table in the database rather than
from static configuration files. That is the single most important fact for an operator: you change
what a run does by changing **state in the database**, not files —
which is exactly why this is a `dlake` skill. You reach that state with the `crmpro_*` tools in §2,
and directly with the CLI's data plane for the rest.

> **Flag table name, worth having up front.** The flag table is **`CommercientFlags`** — the plural
> form. The singular `Commercient_Flag` is never a table name. Use `CommercientFlags`, and confirm
> against the tenant before acting.

---

## 1. When you touch it

- A destination object is not syncing, or is syncing the wrong rows → a `CRM_Configuration` row.
- A field is missing or landing in the wrong destination field → `CRM_FieldList`, or a mapping column.
- Sync stopped entirely, or a test run "turned it off" → a flag in `CommercientFlags`.
- "Did it run? What broke?" → the transaction/state tables in section 4, or `crmpro_errors`.

You do **not** install, schedule or upgrade the agent from here. This skill is the **operating**
surface: what an operator reads and changes about a running CRMPro. Reach for the `crmpro_*` tools in
§2 first; drop to the tables in §3–§4 for what those tools do not cover.

## 2. Operating CRMPro remotely — the `crmpro_*` tools

The control plane carries a dedicated **`crmpro_*`** tool set, and it is the **preferred way to
operate CRMPro**. These tools drive the product's own logic — the connection pair, the delete guard,
the view that belongs to a process, the customer's activity trail — instead of writing rows behind
it. Everything runs from anywhere, over the tenant's API key:

```bash
dlake admin <tool> --profile <tenant> [--arg value ...]
```

`--profile` is **always explicit**. There is deliberately no default profile — naming the tenant on
every call is what stops a command landing on the wrong customer. `DLAKE_PROFILE` in the environment
is the only shorthand.

**Every `crmpro_*` tool is Admin-only.** The key you call with must belong to a tenant user who holds
the **Admin** role; a key minted from a non-admin user is refused with a `403` naming that role. Keys
inherit the roles of the user who minted them, so **mint operating keys from an Admin account** — a
403 here is an account question, not a broken tool or a scope you can widen after the fact. The same
rule governs the `registration_*` wizard tools.

No `crmpro_*` tool takes a user id: the platform resolves the customer from the key. Each returns its
outcome directly, so you read the result rather than inferring it from a follow-up call.

### Quick reference

**Reads — always safe.**

| Tool | What it does | Key args |
|---|---|---|
| `crmpro_list_processes` | Every sync process — the grid an operator works from | `isActiveOnly` (default `false`) |
| `crmpro_get_process` | One process in full: all 39 fields, including the stored connection id | `recordId` |
| `crmpro_get_process_sql` | The `CREATE VIEW` script behind that process's view | `recordId` |
| `crmpro_field_mapping` | The mapping grid — one row per view column, with the saved CRM field | `recordId` |
| `crmpro_sync_status` | The master sync flag. Pure read, no side effect | — |
| `crmpro_sync_history` | The per-record sync rows (key, saved timestamp, upsert time, destination id) | see `--help` |
| `crmpro_sync_history_options` | The object picker that scopes a history query | — |
| `crmpro_errors` | Sync errors for the customer | — |
| `crmpro_agent_info` | The sync agent's own self-report | — |
| `crmpro_flags` | The sync flags for this customer's ERP/CRM pair, with current values | — |
| `crmpro_user_flags` | The customer's extra-info toggles | — |
| `crmpro_connections` | Connection-manager entries and the ids every connection-aware call wants | — |
| `crmpro_crm_objects` | Objects the live CRM offers | — |
| `crmpro_crm_object_fields` | Fields of one CRM object, live | see `--help` |
| `crmpro_tables_views` | Source tables (`false`) or source views (`true`) to build a process from | `isNeedtoCreateFromView` |
| `crmpro_table_columns` | Columns of one table or view | `tableName` |
| `crmpro_templates` | The default-template catalog | see `--help` |
| `crmpro_selected_templates` | Which templates are currently enabled (where present) | — |

**Mutations — these change what the next run does.**

| Tool | What it does | Key args |
|---|---|---|
| `crmpro_set_sync_enabled` | Turn the customer's master sync on or off. **Idempotent**: it reads the current state, toggles only if the state differs, and reports before and after | `enabled` (bool) |
| `crmpro_update_process_field` | Flip one field on one process — the everyday `Is_Active` style lever | `recordId`, `fieldName`, `checked` |
| `crmpro_update_process` | Change a process. Send `recordId` plus only the fields you are changing; the merge happens server-side and omitted fields are preserved | `recordId` + the fields to change |
| `crmpro_create_process` | Create a process (and, for a table-sourced one, its view) | see `--help` |
| `crmpro_delete_process` | Delete a process and attempt to drop its view | `recordId`, `confirm` (must be `true`) |
| `crmpro_apply_template` | Import default templates | see `--help` |

### Worked examples

```bash
# What is this customer syncing, and is sync even on?
dlake admin crmpro_list_processes --profile <tenant>
dlake admin crmpro_list_processes --profile <tenant> --isActiveOnly true
dlake admin crmpro_sync_status    --profile <tenant>

# One process in full — this is where the stored connection id lives
dlake admin crmpro_get_process --profile <tenant> --recordId 123

# Stop one object syncing, without touching anything else
dlake admin crmpro_update_process_field --profile <tenant> \
    --recordId 123 --fieldName Is_Active --checked false

# Master switch. Safe to run when it is already in the wanted state
dlake admin crmpro_set_sync_enabled --profile <tenant> --enabled true

# Remove a process. Refused without the confirmation
dlake admin crmpro_delete_process --profile <tenant> --recordId 123 --confirm true
```

For `crmpro_update_process` and `crmpro_create_process`, take the exact field names from
`dlake admin crmpro_update_process --help` (and from a `crmpro_get_process` read of the row you are
about to change) rather than from memory.

### What these tools know that a raw row write does not

- **A process targets either a connection or the customer's registered CRM — never both.** Either it
  points at a named connection-manager entry (its `apiAuthConfigID` is set, and that entry's name
  decides which CRM it talks to), or that id is null and the process targets the CRM the customer
  registered. On the wire the authoritative signal is **`apiAuthConfigID`**, not the CRM name: reads
  apply a display fallback, so a row with no stored name still shows one. Get valid ids from
  `crmpro_connections`. And although `crmpro_update_process` preserves what you omit, still send the
  connection pair deliberately whenever changing the target is the point of the edit.
- **Live CRM providers are Salesforce, HubSpot and ZohoCRM**, each with automatic token refresh.
  DynamicCRM is recognised but answers `crm_not_supported` — that is the platform telling you it
  cannot talk to that CRM, not a misconfigured tenant.
- **The delete-records flag has a prerequisite.** Turning `Is_Active_Delete_Records` on is refused
  until the process has a non-empty `Delete_SQL_Query`. Non-empty is not the same as valid: a source
  table without a primary key produces a malformed delete query that the guard cannot catch, so read
  the query before you enable the flag.
- **The master sync toggle is customer-wide.** `crmpro_set_sync_enabled` affects every process in the
  tenant. To stop one object, use `crmpro_update_process_field` on that process instead.

## 3. Where these tables live, and how you reach them

Everything from here down is the **fallback path** — the direct-table surface for what the tools in
§2 do not reach: the flag and parameter tables, the cursor tables, and the XML-bearing mapping
columns in §6.

CRMPro's tables are ordinary tables in the **`dbo`** schema of the customer's gateway database —
outside the tenant's working (platform) schema, which is usually `DLO` but not always. The agent
itself reaches them over a **direct database connection**; it does not use the Data API.

When a tenant is seeded, the platform sweeps a **named allowlist** of gateway tables into
**same-named, explicit-column, single-table views in the working schema**. Because each view wraps
one table, SQL Server treats it as **updatable** — you read *and* write through it, omitting identity
columns exactly as you would against the base table.

The CRMPro objects in that allowlist:

| View created for | Kind |
|---|---|
| `CRM_Configuration` | setup |
| `CRM_FieldList` | setup |
| `CRMProRunHistory` | transaction / state |
| `CRMPRO_DASHBOARD_ERROR` | transaction / state |
| `CRMPRO_ERROR_LOG` | transaction / state |
| `CRMPRO_DeleteRecordInfo` | transaction / state |
| `CRMBackupObjectList` | transaction / state |
| `CRM_Prompt_History` | present in the lake view set; inspect before use |
| `TimeStampRepository`, `TimeStampRepositoryHistory` | transaction / state (shared with TxDownloaderPro) |
| `GenericAPISyncConfiguration`, `GenericAPIAuthenticationTemplateLink` | setup (Connection Manager siblings) |

**A view exists only where the tenant actually has the table.** The allowlist is enumerated against
the database at seed time, so a tenant missing one simply has no view for it.

**Three things are deliberately NOT in that set** — plan around them rather than assuming:

- **`CommercientFlags`** — the flag table. No working-schema view is created for it.
- **`CRM_Parameters`** — the generic-REST mapping table. Same.
- **`GenericAPIAuthonticationConfiguration`** — the Connection Manager **credential store**. Its
  exclusion is by design: its JSON blobs are encrypted on save, and a write through a view would
  bypass that encrypt-on-save path. **Do not add it back, and do not try to route around it.** Its non-credential
  siblings (`GenericAPISyncConfiguration`, `GenericAPIAuthenticationTemplateLink`) are included.

Note the spelling: the credential table's name contains the long-standing typo
`...Authontication...`, while the template-link table is spelled `...Authentication...`. Both are
load-bearing. Copy them, don't retype them.

Confirm what a given tenant has before planning anything:

```bash
dlake login --domain <tenant>
dlake tool get_active_schema          # the working schema these views live in
dlake admin list_views                # which CRMPro views this tenant actually got
```

`list_views` takes no arguments and is read-only; it reports `schemaName`, `viewName`, creation date
and an `isSystem` flag per view.

### Getting one onto the Data API

Reads through **raw SQL** work the moment the tenant is seeded — the views are ordinary objects in
the active schema. **Record-level CRUD is different**: `read_records` / `create_record` /
`update_record` / `delete_record` only see entities the Data API publishes, so each view needs the
normal two-step exposure and then one restart:

```bash
dlake admin list_exposed_entities                                        # what is served today
dlake admin set_entity_exposure --entity CRM_Configuration --expose true # repeat per object
dlake admin set_entity_exposure --entity CRM_FieldList     --expose true
dlake admin restart_dab --confirm true                                   # ONE restart, at the end
dlake admin list_exposed_entities                                        # check `served: true`
```

| `set_entity_exposure` argument | Type | Notes |
|---|---|---|
| `entity` | string | **Required.** Unqualified name in the active schema — `CRM_Configuration`, never `dbo.CRM_Configuration` |
| `expose` | boolean | **Required.** `true` adds, `false` removes |
| `keyFields` | string array | The addressable key column(s). Required when a key cannot be derived |
| `confirm` | boolean | Required (`true`) **only** when removing |

`restart_dab` takes `confirm` (boolean, must be `true`) and nothing else. **It briefly interrupts the
tenant's live Data API** — batch every exposure change first, restart once, and do it when the
customer is ready rather than as a reflex. Trust the `served` field over mere presence: a listed but
`served:false` entity answers `EntityNotFound`, which is a missing restart, not a broken entity.

## 4. The tables

### 3a. Setup — what an operator changes

**`CRM_Configuration`** is the heart of it: **one row per destination object per run**, and the row
is what drives that object's behaviour. Columns (as the agent creates them):

| Column | What it does |
|---|---|
| `ID` | Identity PK. Referenced by `CRM_Parameters.CRM_Configuration_ID` |
| `Is_Active` | Push/sync this object to the destination |
| `Is_Active_Get_Records` | Fetch records **from** the destination for this object |
| `Is_Active_Delete_Records` | Delete records in the destination for this object |
| `Sync_Order` | Integer execution order. Rows are processed in ascending `Sync_Order` |
| `CRM_Object_Display_Name` | Human label used throughout the logs |
| `SQL_Query` | The source query/view the rows come from |
| `CRM_Object_API_Name` | Destination object/endpoint name. May carry `{placeholder}` tokens (see §6) |
| `CRM_PK_API_Name` | The destination-side key field |
| `Prefix_OF_Field_OR_Object`, `Postfix_OF_Field_OR_Object` | Prefix/suffix applied to field or object names |
| `TimeStamp_Prefix` | **Load-bearing.** The prefix that namespaces this object's rows in `TimeStampRepository` |
| `Is_Create_Entity` | Create the object/table in the destination |
| `Is_Create_Fields` | Create columns/fields in the destination |
| `Is_In_Commercient_CRM` | Module-specific filter |
| `View_Name_For_Field_Creation` | Source view consulted when creating destination fields |
| `Delete_SQL_Query` | Source query for the delete pass |
| `Get_SOQL_Query`, `Get_SOQL_Query_Type` | Query used by the get-records pass, and its mode |
| `Get_Run_Count`, `Get_Current_Run_Count` | Interval + counter for the get-records pass (see below) |
| `Developer_Comment` | Free text |
| `Sync_Batch_Size` | Rows per destination API batch (default `200`) |
| `Sync_Operation_Type` | `1` upsert · `2` create · `3` update · `5`/`11` skip (default `'1'`) |
| `Document_Source_Path` | Long-text path used by the document-sync path |
| `IsAccountMatching` | Account-matching toggle |
| `Is_Clear_SOQL_Cache_Everytime` | Clear the get-records cache each run |
| `Delete_Records_Limit`, `IsOverwriteDeleteLimit` | Cap on deletes per run, and the override |
| `File_Search_Pattern`, `File_Name_Separator` | File-matching settings for the document path |
| `Last_Run_Date`, `File_History_Date` | Timestamps maintained by the agent |
| `IsViewNeedsToCreate`, `CreateViewQuery` | Whether to build the source view, and its body |
| `CRMName` | Legacy per-row destination-platform name |
| `APIAuthConfigID` | Nullable FK into the Connection Manager. **NULL means "fall back to the flags"** |

The last several columns are added by the agent to an existing table when they are missing, so an
older tenant may genuinely lack some of them. **Read the live shape before you write** — never assume
this list matches the tenant in front of you.

`Get_Run_Count` / `Get_Current_Run_Count` are an interval pair: when the counter reaches the interval
the get-records pass runs for that row and the counter resets; otherwise the counter is incremented
and the pass is skipped. So a row with a large `Get_Run_Count` looks "broken" for several runs by
design.

**`CRM_FieldList`** is the plain relational field map — `ID`, `Object_Name`, `View_Field_Name`
(the source column), `CRM_API_Name` (the destination field), `DateCreated`. This is the first place
to look when a field is landing in the wrong destination field.

**`CommercientFlags`** — the flag table. Two columns matter: **`FlagName`** and **`Value`**, both
read as strings; the agent reads the whole table at startup. The documented flags:

| Flag | Purpose | Default |
|---|---|---|
| `CRM_NAME` | Target platform identifier (fallback dispatch) | detected from DB |
| `FirstTimeSync` | Create database objects on this run, then **exit without syncing** | `True` |
| `IS_CRMPRO_SYNC_ENABLED` | Master enable/disable | `1` |
| `IS_SYNC_RUNNING` | Concurrency lock | `0` |
| `CRM_PRO_CHECK_NORMAL_SYNC_LAST_RUN` | Enable timestamp-based change detection | `0` |
| `CRM_PRO_FORCE_SYNC_RUN` | Force a sync, ignoring change detection | `0` |
| `IS_TOP_10_RECORDS_SYNC_REQUIRED` | Limit the run to top records, for testing | `0` |
| `SF_PARTNER_API_VERSION` | Destination API version | `54.0` |
| `CRM_SQL_CONNECTION_TIMEOUT` | SQL command timeout override | not set |
| `Drop_SP_From_DB` | Recreate stored procedures | `False` |

Many more flags exist per destination platform (credentials, batch sizes, per-object hour offsets,
retention intervals). Enumerate the live table rather than working from this list — it is the
documented subset, not the whole surface. **Flags are tenant-global**: one flag row changes behaviour
for every config row in that database.

**`CRM_Parameters`** drives the generic REST path — see §6. Columns: `ID`,
`CRM_Configuration_ID` (→ `CRM_Configuration.ID`), `GroupName`, `Key`, `Value`,
`IsDefaultForAllRequest`, `IsActive`.

**Connection Manager** (newer tenants). `GenericAPIAuthonticationConfiguration` holds one row per
named connection: `APIAuthConfigID` (identity PK), `APIAuthConfigName`, `APIAuthConfigurationJSON`,
`APIAuthConfigurationJSON_Backup`, `APIAuthResponseJSON`, `IsDeleted`, `LastModifiedDate`.
`GenericAPIAuthenticationTemplateLink` links a connection to an auth template. A
`CRM_Configuration` row points at a connection through `APIAuthConfigID`; when that is NULL — or
points at a row that no longer exists — the agent falls back to `CRM_NAME` + the flag table.

### 3b. Transaction / state — what an operator inspects

| Object | Columns | Read it when |
|---|---|---|
| `TimeStampRepository` | `Key` (PK, ≤900 chars), `SavedTimeStamp` (8-byte binary), `UpSertTime`, `SFDCID` | You need the per-record cursor. `Key` is `TimeStamp_Prefix` + the record's source key; `SFDCID` is the destination-side id |
| `TimeStampRepositoryHistory` | `ID`, then the same four columns | You need the history behind a cursor |
| `CRMProRunHistory` | `ID`, `RunInfo` (long text), `RunDateTime` | "Did it run, and when?" |
| `CRMPRO_DASHBOARD_ERROR` | `id`, `SyncDate`, `RecordType`, `ObjectAPIName`, `TimeStampPrefix`, `ErrorKey`, `ErrorDescription`, `NoOfRecordsCount` | Dashboard errors for the **latest** run — this table is truncated each run |
| `CRMPRO_ERROR_LOG` | `id`, `SyncDate`, `RecordType`, `ObjectAPIName`, `TimeStampPrefix`, `ErrorKey`, `ErrorDescription`, `RunID` | Persistent errors, grouped by `RunID` |
| `CRMPRO_DeleteRecordInfo` | `ID`, `SObjectName`, `ViewName`, `DeletedDate`, `DeleteRecordXML`, `DeleteStatus`, `SFDCID`, `ExternalKey` | Auditing what the delete pass removed |
| `CRMBackupObjectList` | `ID`, `CRMObjectName`, `Source`, `EntryDate`, `IsCompleted`, `ViewName`, `TableName`, `IsDeleted`, `IsNeverDeleted`, `DeletedTableDate` | Tracking the backup/field-tracking objects |

The agent also writes local log files on the customer's sync server (a rotating run log, an error
log, a run summary, and a binary last-run-timestamp file). Those are **not** in the database and not
reachable from `dlake` — ask for them if the database tables do not explain a failure.

## 5. Direct table CRUD — the fallback path

Use this **when a `crmpro_*` tool does not cover what you need**: the flag and parameter tables, the
cursor tables, `CRM_FieldList` rows you want to edit relationally, and anything you are reading
purely to diagnose. For listing, reading, creating, changing and deleting a **process**, for the
**master sync flag**, and for the **mapping grid**, prefer §2 — those tools carry guards and a
connection-pair discipline that a row write does not.

Argument names below are exact. Getting one wrong is worse than not acting: confirm against
`dlake tool <tool> --help` / `dlake admin <tool> --help` on the tenant in front of you before a
write, since the tool set evolves.

### Read — always safe

```bash
# Shape first: works for VIEWS exactly as for tables
dlake tool describe_entities --entities CRM_Configuration,CRM_FieldList

# The config rows, in the order the agent will process them
dlake tool read_records --entity CRM_Configuration \
    --select ID,CRM_Object_Display_Name,Is_Active,Sync_Order,Sync_Operation_Type,TimeStamp_Prefix \
    --orderby "Sync_Order asc" --first 100

# One object's field map
dlake tool read_records --entity CRM_FieldList \
    --filter "Object_Name eq 'Account'" --first 200

# Latest errors
dlake tool read_records --entity CRMPRO_DASHBOARD_ERROR --orderby "SyncDate desc" --first 50
```

| Tool | Arguments |
|---|---|
| `describe_entities` | `entities` (string array), `nameOnly` (boolean). Never pass both |
| `read_records` | `entity`\*, `select` (comma-separated string), `filter` (OData: `eq ne gt ge lt le and or not`), `orderby` (**array** of `"col asc"` strings), `first` (integer page size), `after` (cursor) |
| `query` | `sql`\* — a single read-only `SELECT` (or `WITH … SELECT`) |

\* = required.

The page cap is **`first`** — not `limit`, `top` or `pageSize`. It is the most common mis-guess
against this API.

`query` reaches anything in the active schema without exposure, which makes it the tool for
joining a config row to its parameters or its cursor rows. **Qualify with the active schema**
(`FROM DLO.CRM_Configuration` when the active schema is `DLO`) — unqualified names resolve against
the login's default schema and error. `INFORMATION_SCHEMA.*` and `sys.*` are refused by design; use
`describe_entities` for shape. Results cap at 10,000 rows, and `query` is refused for
scope-restricted keys unless the key was minted with the raw-SQL opt-in.

### Write — these change the next run

```bash
# Turn one object off for the next run
dlake tool update_record --entity CRM_Configuration \
    --keys '{"ID": 12}' --fields '{"Is_Active": false}'

# Correct one field mapping
dlake tool update_record --entity CRM_FieldList \
    --keys '{"ID": 88}' --fields '{"CRM_API_Name": "Customer_Ref__c"}'

# Add a mapping row (omit the identity column)
dlake tool create_record --entity CRM_FieldList \
    --data '{"Object_Name":"Account","View_Field_Name":"CUST_NO","CRM_API_Name":"Customer_Ref__c"}'

# Remove a mapping row
dlake tool delete_record --entity CRM_FieldList --keys '{"ID": 88}'
```

| Tool | Arguments |
|---|---|
| `create_record` | `entity`\*, `data`\* (object: field → value) |
| `update_record` | `entity`\*, `keys`\* (object: key column → value), `fields`\* (object: field → new value) |
| `delete_record` | `entity`\*, `keys`\* (object: **all** key columns) |

On Windows especially, pass object/array arguments as **`@file.json`** — inline JSON gets mangled by
the shell, and `@file.json` is the reliable path on every platform.

**Omit identity columns from a `create_record` body** — `ID` on `CRM_Configuration`,
`CRM_FieldList`, `CRMBackupObjectList`; `id` on the error tables. The views are updatable precisely
because they wrap a single table, and they behave like the base table in this respect.

### What is safe, and what is not

| Operation | Verdict |
|---|---|
| Any read of any of these objects | **Safe.** |
| Editing `Developer_Comment` | **Safe.** Free text, no behaviour |
| Editing `CRM_FieldList` rows | **Changes the next run** — the intended, ordinary field-mapping edit |
| Toggling `Is_Active` / `Is_Active_Get_Records` / `Is_Active_Delete_Records` | **Changes the next run.** The everyday lever — but do it with `crmpro_update_process_field`, which carries the delete-query guard. Also see the flag caveats in §8: a few destination modules do not honour these |
| Changing `Sync_Order`, `Sync_Batch_Size`, `Delete_Records_Limit` | **Changes the next run.** Reversible, low blast radius |
| Changing `Sync_Operation_Type` | **Changes the next run**, and changes *semantics* — upsert vs create vs update vs skip |
| Changing `TimeStamp_Prefix` | **Dangerous.** It namespaces this object's rows in `TimeStampRepository`. Change it and the object's entire cursor history is orphaned; the next run behaves like a first run for every record |
| Editing `TimeStampRepository` rows | **Dangerous.** This is the change-detection cursor. Deleting a row re-sends that record; editing `SavedTimeStamp` by hand silently mis-scopes what the next run picks up |
| Changing a flag row | **Tenant-global.** One row changes behaviour for every config row in the database. See §8 |
| Deleting a `CRM_Configuration` row | **Destructive**, and it leaves the process's view behind. Prefer `Is_Active = false`, which is reversible and leaves the cursor history intact; when the process really must go, use `crmpro_delete_process --confirm true`, which also attempts the view drop |
| Anything in `GenericAPIAuthonticationConfiguration` | **Out of bounds from here.** Not view-wrapped, deliberately. Use the Admin Portal's Connection Manager |

## 6. Field mapping — the JSON- and XML-bearing columns

Three mapping surfaces, in ascending order of how careful you need to be.

### Relational: `CRM_FieldList`

The plainest one, and the one you will use most: `View_Field_Name` → `CRM_API_Name`, per
`Object_Name`. Read the mapping a process actually resolves with
`dlake admin crmpro_field_mapping --profile <tenant> --recordId <id>` — one row per view column, with
the saved CRM field alongside it, which is the view an operator wants before changing anything. Where
you then need row-level edits the tools do not offer, these are ordinary rows with ordinary CRUD,
per §5.

### Placeholder tokens + `CRM_Parameters` (generic REST destinations)

For destinations driven by the generic REST path, request shape comes from `CRM_Parameters` rows
keyed to a `CRM_Configuration_ID`, grouped by `GroupName`. The group names the agent uses:

`APIRequest` · `APIHeaders` · `APIParameters` · `APISuccess` · `APIFailure` · `Paging` ·
`SkipFieldsfromBody` · `SkipFieldsforPrePostFix` · `Oauth`

Within a group, `Key`/`Value` carry the settings — for example an `APIRequest` key of `Body Type`
whose value mentions `array` makes the request body a JSON array, and `APISuccess` keys such as
`Success JArray Object Name`, `Success Key`, `Success RecordID`, `Success Name` and `Success Value`
tell the agent where to find the created record's id in the response. `SkipFieldsfromBody` holds a
comma-separated list of columns to strip from the body.

**Where the field mapping happens:** a `Value` (and `CRM_Object_API_Name` itself) may contain
`{ColumnName}` tokens, which the agent substitutes from the **source view's columns** at run time,
per row. A token prefixed `Multiple_` — `{Multiple_ColumnName}` — is substituted with that column's
values across the whole batch, comma-joined. If a token names a column the view does not have, the
agent logs that the view does not contain it and **skips the object for that run**. So a mapping
break here shows up as a silently skipped object, not an error from the destination.

`CRM_Parameters` is **not** in the seeded view allowlist (§3). Check `list_views` before planning to
edit it through the Data API.

### XML-bearing columns

Two genuinely different things wear the name "XML" here. Do not conflate them.

**1. Source-view columns whose names end in `QBCUSTOMFIELDSXML`, or are named
`ADDITIONALCONTACTREFXML`.** These carry a real XML document in the *data*, and the agent expands it
into destination custom fields — this is what makes name/value extension data mappable at all. Three
shapes appear, each a flat list of name/value pairs. Synthetic examples:

```xml
<ArrayOfDataExtRet>
  <DataExtRet>
    <OwnerID>0</OwnerID>
    <DataExtName>Sample Field</DataExtName>
    <DataExtType>STR255TYPE</DataExtType>
    <DataExtValue>sample value</DataExtValue>
  </DataExtRet>
</ArrayOfDataExtRet>
```

```xml
<ArrayOfAdditionalContactRef>
  <AdditionalContactRef>
    <ContactName>Sample Contact</ContactName>
    <ContactValue>sample value</ContactValue>
  </AdditionalContactRef>
</ArrayOfAdditionalContactRef>
```

```xml
<ArrayOfEntityPropertyOfString>
  <EntityPropertyOfString>
    <Name>Sample Property</Name>
    <Value>sample value</Value>
  </EntityPropertyOfString>
</ArrayOfEntityPropertyOfString>
```

The `DataExtName` / `ContactName` / `Name` element is the mapping key — it is what becomes the
destination field. The expansion is **conditional**, not universal: it is gated on the run's source
ERP and on a per-call switch, and not every destination module performs it. Establish that a given
tenant's run actually does it before promising a customer a mapping through this route. The XML lives
in the **customer's source view**, not in a CRMPro setup table, so you change it by changing the view
or the data behind it — not by editing a config row.

**2. `CRMPRO_DeleteRecordInfo.DeleteRecordXML`.** Declared as an `xml` column, but the current
delete-pass writers store the literal marker `"1"` in it. **Do not expect a document there**, and do
not build tooling that parses it. It is a flag wearing an XML column's clothes.

### JSON-bearing columns (Connection Manager) — read-only in practice

`GenericAPIAuthonticationConfiguration` carries three long-text JSON columns:

| Column | Holds |
|---|---|
| `APIAuthConfigurationJSON` | The connection's auth configuration. Written encrypted by the portal |
| `APIAuthConfigurationJSON_Backup` | The previous value, kept before an update |
| `APIAuthResponseJSON` | The cached auth response — tokens, refreshed by the agent after an OAuth refresh |

The configuration JSON's top level carries `authonticationType` (note the typo — `0` NoAuth,
`1` BasicAuth, `2` OAuth1.0, `3` OAuth2.0, `4` APIKey, `5` NTLM) alongside the typed sub-objects
`oAuth2_0Request`, `oAuth2_0Response`, `basicAuthRequest`, `apiKeyRequest`. Synthetic shape:

```json
{
  "authonticationType": 3,
  "oAuth2_0Request": {
    "OAuth2_0AccessTokenURL": "https://example.invalid/oauth/token",
    "OAuth2_0HttpRequestMethod": "POST",
    "OAuth2_0HttpContentType": "application/x-www-form-urlencoded",
    "OAuth2_0TokenExpireMin": 30,
    "OAuth2_0HttpHeaderList": [{ "HeaderKey": "Accept", "HeaderValue": "application/json" }],
    "OAuth2_0HttpParamsList": [{ "ParamsKey": "client_id", "ParamsValue": "<redacted>" }],
    "FormUrlEncodedContentList": [
      { "FormUrlEncodedContentKey": "grant_type", "FormUrlEncodedContentValue": "refresh_token" }
    ]
  },
  "oAuth2_0Response": {
    "access_token": "<redacted>",
    "refresh_token": "<redacted>",
    "expires_in": 1800,
    "token_type": "bearer",
    "TokenLastCreateDate": "2026-01-01T00:00:00Z"
  },
  "basicAuthRequest": { "UserName": "<redacted>", "UserPassword": "<redacted>" },
  "apiKeyRequest": { "ApiKeyPrefix": "Bearer", "ApiKeyName": "<redacted>" }
}
```

`APIAuthResponseJSON` is the response object on its own — the same `access_token` /
`refresh_token` / `expires_in` / `token_type` / `TokenLastCreateDate` keys, and the agent prefers it
over the copy embedded in the configuration because it is the fresher one. Both readers tolerate
either camelCase or snake_case token keys.

**You do not edit these through the Data Lake.** The credential table is excluded from the view
sweep on purpose (§3), and the JSON is credential-bearing. Change connections in the Admin Portal's
Connection Manager, which owns the encrypt-on-save path. What you *may* usefully do from here is
observe the **link**: read `CRM_Configuration.APIAuthConfigID` to see which connection a config row
resolves through, and whether it is NULL (flag fallback) or set.

## 7. Verifying a change, and what a run does next

**Verify the write itself.** Read the record back:

```bash
dlake admin crmpro_get_process --profile <tenant> --recordId 12          # after a crmpro_* write
dlake tool  read_records --entity CRM_Configuration --filter "ID eq 12" --first 1   # after a row write
```

Read-back matters most after `crmpro_update_process_field`: an unrecognised `fieldName` reports
success and changes nothing, and the record is the only place that shows it.

**Nothing takes effect until the next scheduled run.** CRMPro is a scheduled per-customer agent; it
reads its configuration and flags at **startup**. An edit made mid-run does not affect the run in
flight, and an edit made between runs takes effect at the next one. There is no "apply now" from the
`dlake` side — do not tell a customer a change is live because the row updated.

**What a run does afterwards:** it loads the flags and the configuration rows, creates destination
schema where the create flags allow it, detects changes by timestamp, transforms, calls the
destination API, and writes errors and a summary. Configuration rows are processed sequentially in
`Sync_Order`, each row resolving its own connection.

**Where to look after the run:**

Start with `crmpro_errors` and `crmpro_sync_history` — the fastest read of what broke and what moved.
Then, for the detail those do not carry:

1. `CRMProRunHistory` — a row with `RunDateTime` proves the agent ran.
2. `CRMPRO_DASHBOARD_ERROR` — errors from the **latest** run only (it is truncated each run).
3. `CRMPRO_ERROR_LOG` — persistent errors; filter by `RunID` to isolate one run.
4. `TimeStampRepository` — `UpSertTime` on rows whose `Key` starts with the object's
   `TimeStamp_Prefix` shows the cursor actually moved.
5. `CRMPRO_DeleteRecordInfo` — what the delete pass removed.

If none of those moved, the question is whether the agent ran at all — a scheduling/host question,
answered from the customer's sync server, not from here.

## 8. Things that bite

- **The `crmpro_*` tools are Admin-only.** The calling key must belong to a tenant user holding the
  **Admin** role; anything else gets a `403` naming that role. Keys inherit the user's roles, so the
  fix is a key minted from an Admin account — not a scope change, not a retry. Same for the
  `registration_*` wizard tools.
- **`--profile` is never optional.** There is no default profile. Omit it and the command does not
  quietly pick a tenant for you; `DLAKE_PROFILE` is the only shorthand.
- **An unknown `fieldName` on `crmpro_update_process_field` reports success and changes nothing.**
  Only whitelisted columns are writable through it. Spell the field exactly as the record does, and
  read the record back with `crmpro_get_process` when it matters.
- **`crmpro_delete_process` requires `--confirm true`.** Without it the call is refused. With it the
  process is gone — prefer switching `Is_Active` off first, and reserve deletion for a process that
  should not exist.
- **A process targets a connection OR the registered CRM, never both.** `apiAuthConfigID` set means
  the connection-manager entry decides the CRM; null means the customer's registered CRM. Reads apply
  a display fallback, so the CRM name on the wire is not proof of anything — trust `apiAuthConfigID`,
  and take valid ids from `crmpro_connections`. `crmpro_update_process` preserves fields you omit, but
  send the connection pair deliberately when retargeting is the point of the edit.
- **`Is_Active_Delete_Records` needs a `Delete_SQL_Query` first**, and the guard only checks that the
  query is non-empty. A source table with no primary key yields a malformed delete the guard cannot
  catch — read the query yourself before enabling the flag.
- **The master sync toggle is customer-wide.** `crmpro_set_sync_enabled` is idempotent and reports the
  state before and after, but it moves the whole customer. One object belongs in
  `crmpro_update_process_field`.
- **DynamicCRM answers `crm_not_supported`.** Salesforce, HubSpot and ZohoCRM are the live providers,
  each refreshing its own tokens. That refusal is the platform being clear, not a tenant fault.
- **`TimeStamp_Prefix` is the cursor namespace.** `TimeStampRepository.Key` is built as
  `TimeStamp_Prefix` + the record's source key. Rename the prefix and every existing cursor row for
  that object is orphaned in place — no error, and the next run re-sends the whole object. Treat this
  column as immutable once an object has synced.
- **Editing cursor rows by hand silently changes what syncs.** `SavedTimeStamp` is an 8-byte binary
  change-detection value, not a date you can eyeball. Deleting a row re-sends that record; a wrong
  value mis-scopes the run. Read this table freely; write to it only with a reason you can state.
- **`Sync_Operation_Type` is semantics, not a toggle.** `1` upsert · `2` create · `3` update ·
  `5`/`11` skip. Setting it to `2` on an object that already exists in the destination means the run
  stops updating and starts creating.
- **A "test" run can leave sync switched off.** After a run made with `IS_TOP_10_RECORDS_SYNC_REQUIRED`
  set, the agent sets **`IS_CRMPRO_SYNC_ENABLED` to `0`** as well as clearing the test flag. That is
  deliberate — but if you turn the test flag on and walk away, the customer's sync is disabled and
  the flag that disabled it is not the flag you set. Check `IS_CRMPRO_SYNC_ENABLED` afterwards.
- **`CRM_PRO_FORCE_SYNC_RUN` is one-shot.** The agent resets it to `0` at the end of a run. If you
  find it at `0`, that does not mean nobody set it.
- **`FirstTimeSync = True` costs you a run.** That run creates database objects and **exits without
  syncing**, then sets the flag to `False`. This two-phase startup is intended and must be preserved.
  Do not set it back to `True` to "refresh schema" unless you intend to burn a run.
- **`IS_SYNC_RUNNING` can strand.** It is a concurrency lock, and a crash mid-sync can leave it at
  `1`, blocking future runs until it is manually reset to `0`. That reset is the supported recovery;
  the enforcement lives outside the tables this skill covers, so verify the agent is genuinely not
  running before you clear it.
- **Flags are tenant-global; config rows are per-object.** If a change should affect one object,
  it belongs in `CRM_Configuration`, not in a flag.
- **Not every destination module honours the flags.** A few modules act without checking `Is_Active`
  (and some without checking `Is_Active_Get_Records` either) — **Trello**, **Asana** and **Shopify**
  among them. If setting `Is_Active = false` does not stop a sync, this is the first thing to check,
  not a Data Lake problem. Re-verify against the tenant's own build.
- **`DeleteRecordXML` is not XML.** See §6.
- **The credential JSON is off-limits from here.** No view is created for the Connection Manager
  credential table, by decision, because a write through a view would bypass encryption on save.
- **`CommercientFlags` and `CRM_Parameters` have no seeded view either.** Do not plan an edit through
  the Data API without checking `list_views` first.
- **Exposure vs restart.** An entity in the scope but `served: false` answers `EntityNotFound`. That
  is a missing `restart_dab`, not a broken view. And restart once, at the end — each restart briefly
  interrupts the tenant's live Data API.
- **Spelling is load-bearing.** `GenericAPIAuthonticationConfiguration` and `authonticationType`
  carry a long-standing typo; `GenericAPIAuthenticationTemplateLink` does not. The agent's flag table
  is `CommercientFlags`, not the singular form. Copy names; do not correct them.

## 9. Where this sits

| Skill | Covers |
|---|---|
| `dlake-integration-setup` | Standing an integration up: registration, verification, seeding, then the wizard — CRM choice and the ERP connector |
| **`dlake-crmpro`** (this) | Operating the **forward** leg: the `crmpro_*` tools, and the setup and transaction tables behind them — processes, sync control, field mapping, diagnostics |
| `dlake-txdownloaderpro` | The **writeback** leg: exposing the TxDownloaderPro objects and scoping a key to them |
| `dlake` | Operating a tenant generally — schema, queries, exports, keys, the REST/GraphQL contract |

Do this **after** the tenant exists and has been seeded: the gateway views are created at seed time,
so there is nothing to read or expose before then.
