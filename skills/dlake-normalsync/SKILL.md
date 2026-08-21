---
name: dlake-normalsync
description: >-
  Choose which of a customer's ERP tables the Normal Sync change-tracking agent clones into the
  CommercientGateway tenant database, with the `dlake` CLI and the `normalsync_*` admin tools — the
  available-tables dropdown, the two-call add sequence, per-table sync toggles and row filters, and
  the prerequisites this surface CANNOT set (so finishing it does not mean the customer syncs). Use
  it when the task is "sync this ERP table", "stop syncing that one", "filter which rows sync", or
  "why is this table not in the clone tables?". Normal Sync is the ERP-side EXTRACT only. For the
  CRM push that consumes the clone tables use the `dlake-crmpro` skill; for the CRM-to-source
  writeback leg use `dlake-txdownloaderpro`; for standing a new tenant up from scratch use
  `dlake-integration-setup`. When the source is NOT Microsoft SQL Server, the ODBC agent stacks
  underneath this one — configure it with `dlake-odbcsync`; when the source is an API rather than
  a database, use `dlake-apisync`.
---

# Choosing what Normal Sync clones, with `dlake`

## 1. When you need this

The customer's tenant already exists and the sync agent is (or is about to be) installed, and the
question is **which ERP tables get cloned into their gateway database**. That is one screen's worth
of decisions and six endpoints, reachable as eight `normalsync_*` admin tools.

**Use the `dlake normalsync` shorthand group** — it covers all eight, and the examples in this skill
are written in it. The full form of any verb is the generic passthrough,
`dlake admin normalsync_<tool>`, which prints each tool's live argument schema with `--help`; reach
for it when you want the schema, or an argument the shorthand does not expose.
`dlake normalsync --help` lists the group.

You do NOT need this skill to stand a tenant up (`dlake-integration-setup`), to change what reaches
the CRM (`dlake-crmpro`), or to push CRM edits back to the ERP (`dlake-txdownloaderpro`).

Everything here is **Admin only, reads included**. A key minted from a non-admin user is refused
with a message naming the role.

---

## 2. Where Normal Sync sits — three products, two agents

Get this section right and the rest of the skill is mechanical. Get it wrong and you will aim
perfectly good instructions at the wrong product.

### Normal Sync — the ERP-side extract

**Normal Sync is the on-prem CHANGE-TRACKING sync agent for Microsoft SQL Server 2008 R2 and
later.** It runs on the customer's premises, watches their ERP source database, and writes the
changed rows into the **`dbo` clone / change-tracking tables in their CommercientGateway tenant
database**. That is its entire job.

The clone tables are named with a prefix — `SF_ERP_Salesforce_Clone_<TABLE>` — while the agent, this
surface and the customer all talk in logical table names (`AR_CUSTOMER`). Worth knowing before you
go looking for a table called `AR_CUSTOMER` in the tenant database and conclude nothing synced.

### The ODBC agent stacks UNDERNEATH it — it does not replace it

For a source that is **not** Microsoft SQL Server, a separate **ODBC sync agent** syncs into an
**intermediary database**, and **Normal Sync then moves that into the same `dbo` clone tables**.

So on an ODBC customer **both agents run**, and Normal Sync is still the leg that fills the clone
tables. Configuring Normal Sync is never optional, and "they're on ODBC" is never a reason to skip
this surface. `normalsync_readiness` reports **ODBC Sync** and **Normal Sync** as two separate
phases for exactly this reason.

### CRM Pro — not Normal Sync — consumes the clone tables

**CRM PRO reads the clone tables through views and pushes to the CRM.** Nothing in this skill moves
a single row towards a CRM.

This is the most useful diagnostic boundary on the platform. If the complaint is "the CRM is missing
records":

- rows missing from the **clone table** → an extract problem → **here**;
- rows present in the clone table but missing in the **CRM** → a push problem → **`dlake-crmpro`**.

### TxDownloaderPro — the writeback leg

**TxDownloaderPro is the CRM → source writeback leg**, a third distinct product. Never abbreviate
it, and never conflate it with **TxDownloader** (a different, older product) or with **CRM Pro**.

### The pipeline, in one line

```
source DB → [ODBC agent → intermediary DB →] Normal Sync → dbo clone tables (gateway DB)
          → views → CRM Pro → CRM
```

…with **TxDownloaderPro** running the arrow back the other way, CRM → source.

---

## 3. See the current scope

```bash
dlake normalsync selected
# full form: dlake admin normalsync_list_selected_tables
```

One row per table this customer has selected (the shorthand's table shows the id, name, state,
filter and index hint; `--json` gives every field, including the two below):

| Field | What it means |
|---|---|
| `id` | The **catalogue row id**. Every per-table tool takes this as `tableId`. |
| `tableName` | The ERP table's logical name, e.g. `AR_CUSTOMER`. |
| `description` | The catalogue's description of the table. Informational. |
| `syncStatus` | **The state.** `true` = this table is syncing. |
| `commandName` | **A BUTTON LABEL, NOT A STATE** — the label of the button that *toggles* the row, so it is the **opposite** of `syncStatus`. A table that IS syncing reads `DeActivate`. |
| `whereClause` | The row filter appended to this table's sync query. Empty = the whole table. |
| `sqlIndex` | Index hint for the agent's query. Usually empty. |
| `aiName` | **Always empty.** The AI-description feature was never migrated; an empty value here means nothing at all. |

Read `syncStatus` for the state. Every operator who has read `commandName` as the state has reported
the exact opposite of the truth.

---

## 4. Add a table — two calls, in order

```bash
# 1. What COULD be added: this ERP's catalogue minus what is already selected
dlake normalsync tables

# 2. Select one for this customer — the name and the TABLE ID come from step 1
dlake normalsync select AR_CUSTOMER 41

# full form:
#   dlake admin normalsync_available_tables
#   dlake admin normalsync_select_table --tableName AR_CUSTOMER --tableId 41
```

`normalsync_available_tables` returns `{text, value}` pairs. `text` is the table name **upper-cased
by the API**; `value` is the catalogue row id that `normalsync_select_table`,
`normalsync_set_sync_enabled` and `normalsync_set_row_filter` all take as `tableId`.

**`normalsync_select_table` is the write that makes a customer Normal Sync ready** — readiness
counts exactly these rows. It is idempotent, so a retry is a no-op and no confirmation is required.

### When the table is not in the catalogue yet

If the table you want is not in `normalsync_available_tables`, nobody has catalogued it for that ERP
yet. Cataloguing it is a real, supported action — it is the mechanism for syncing a table nobody
foresaw, without waiting on Commercient — but read the next paragraph before you use it.

```bash
# Step one only: put the name in the ERP's catalogue and get its id back
dlake normalsync catalog SO_HEADER --confirm

# Or both steps at once
dlake normalsync add SO_HEADER --confirm

# full form:
#   dlake admin normalsync_catalog_table --tableName SO_HEADER --confirm true
#   dlake admin normalsync_add_table     --tableName SO_HEADER --confirm true
```

**The catalogue is SHARED by every customer on that ERP.** The row you write appears in every other
customer's available-tables dropdown, and **nothing in this API can delete a catalogue row** —
removing a mistake is a DBA task. So:

- **check `normalsync_available_tables` first.** An existing row is reused rather than duplicated,
  which means a near-miss spelling (`AR_CUSTOMR`) is exactly how junk gets in permanently;
- spell the name **exactly** as the ERP does;
- both tools are confirm-gated for this reason, not because they delete anything. The CLI refuses
  without `--confirm` **before** it calls, so a forgotten flag costs nothing.

`normalsync_catalog_table` does **not** select the table for the customer and does not change what
syncs. Follow it with `dlake normalsync select`, or use `dlake normalsync add`, which does both.
The catalogue leg is authoritative and runs first, so if `add` fails at the select step the
catalogue row already exists — re-run `dlake normalsync select <tableName> <tableId>` with the id it
names, **not** `add`, which would touch the shared catalogue a second time.

---

## 5. Per-table settings

### Turn one table's sync on or off

```bash
dlake normalsync disable AR_CUSTOMER 41
dlake normalsync enable  AR_CUSTOMER 41

# full form (equivalent):
#   dlake admin normalsync_set_sync_enabled --tableName AR_CUSTOMER --tableId 41 --enabled false
#   dlake admin normalsync_set_sync_enabled --tableName AR_CUSTOMER --tableId 41 --command deactivate
```

The shorthand sends the INTENT as a boolean; on the full form pass `enabled` (a boolean) or
`command` (`activate` / `deactivate`, accepted in any case). The tool constructs the exact literal
the API demands. **Do not hand-build that literal against the raw
endpoint** — see §7, first entry; it is the sharpest edge on this surface.

Disabling is **silent downstream**: the clone table simply stops updating, and CRM Pro keeps pushing
whatever is already there, with no error anywhere. Nothing will tell you later that you did this.

### Row filters and index hints

```bash
# Sync only part of a table
dlake normalsync filter 41 --where "CustomerNo > '1000'" --confirm

# Set just the index hint (the filter is preserved)
dlake normalsync filter 41 --index IX_CUSTOMER --confirm

# Remove both the filter and the index hint
dlake normalsync filter 41 --clear --confirm

# full form:
#   dlake admin normalsync_set_row_filter --tableId 41 --whereClause "CustomerNo > '1000'" --confirm true
#   dlake admin normalsync_set_row_filter --tableId 41 --clear true --confirm true
```

Two things to hold in mind:

1. **The filter is EXECUTED BY THE ON-PREM AGENT** against the customer's ERP database, under the
   agent's own ERP credentials, and it is **not validated as SQL anywhere** — not by the API, not by
   this tool. Write it without the `WHERE` keyword, and do not paste anything you have not read.
   A malformed filter breaks that table's sync at the next tick, with the error on the agent side.
2. **The two columns are written together upstream**, so a partial write would silently clear the
   other one. The tool therefore reads the row first and overlays only what you supply — an omitted
   value is **preserved**. To remove them, pass `--clear` (`clear: true` on the full form), which
   cannot be combined with a new value.

Both are why this tool is confirm-gated.

---

## 6. ⚠ Selecting tables is necessary but NOT sufficient

A customer with ten selected tables and none of the following syncs nothing at all. **None of these
five things can be set from this surface** — they come from the agent install and the ERP connector
setup:

| Prerequisite | Where it comes from |
|---|---|
| The sync agent installed (`SyncAgentProduct` type 10, `IsInstalled = 1`) | The on-prem agent installer |
| `ERP_SQLHOSTNAME` | ERP connector setup (`dlake-integration-setup`) |
| `ERP_SQLUSERNAME` | ERP connector setup |
| `ERP_SQLPASSWORD` | ERP connector setup |
| `ERP_SQLDATABASE` | ERP connector setup |

```bash
dlake normalsync readiness
# full form: dlake admin normalsync_readiness
```

**This is the honest answer to "is this customer set up?"** — all four phases (ODBC Sync, Normal
Sync, Phase 1, Phase 2), each with `ready`, a count of what is configured, and the **reasons** when
it is not ready. A bare table count is not readiness, and reporting one as readiness is the single
most common way this surface is misused.

A customer with no gateway database yet is reported not-ready on every phase: provisioning has not
finished, so there is nowhere for the prerequisites to exist.

---

## 7. Verify

In order, after any change here:

1. `dlake normalsync selected` — read the grid back. The change is there, and the SYNCING column
   (`syncStatus`) says what you think it says.
2. `dlake normalsync readiness` — Normal Sync ready, with no issues listed.
3. Confirm rows are landing in the **clone table** in the tenant database (the
   `SF_ERP_Salesforce_Clone_<TABLE>` table) on the next agent tick.

**Step 3 is the boundary of this skill.** Once the clone table is filling up, Normal Sync has done
its job. Anything still missing further along is a CRM Pro question — go to `dlake-crmpro`.

---

## 8. Quick reference

| CLI verb | Tool | What it does |
|---|---|---|
| `normalsync tables` | `normalsync_available_tables` | The dropdown: catalogue rows this customer has NOT selected. `value` is the `tableId`. Read-only. |
| `normalsync selected` | `normalsync_list_selected_tables` | The grid: what this customer HAS selected, with state, filter and index hint. Read-only. |
| `normalsync readiness` | `normalsync_readiness` | All four phases with per-phase reasons — the honest "is this set up?". Read-only. |
| `normalsync catalog <table> --confirm` | `normalsync_catalog_table` | Step one: add a name to the ERP's **SHARED** catalogue, return its id. Confirm required. |
| `normalsync select <table> <tableId>` | `normalsync_select_table` | Step two: select a catalogued table for this customer. **This is what flips readiness.** |
| `normalsync add <table> --confirm` | `normalsync_add_table` | Both steps at once, for a table nobody has catalogued. Same shared-catalogue blast radius. Confirm required. |
| `normalsync enable`/`disable <table> <tableId>` | `normalsync_set_sync_enabled` | Turn one selected table's sync on or off. |
| `normalsync filter <tableId> …--confirm` | `normalsync_set_row_filter` | Set (`--where`, `--index`) or clear (`--clear`) one table's row filter and index hint. Confirm required. |

Every verb has a full form, `dlake admin <tool_name>`, and
`dlake admin list | grep normalsync_` prints all eight with their live argument schemas. Use the
full form when you want the schema, or `--command activate` instead of the boolean.

---

## 9. Things that bite

- **`"activate"` is not `"Activate"`.** The underlying endpoint compares the command
  **case-sensitively** against the exact literal `Activate`, and **anything else DISABLES the
  table** — then answers `"Disable AR_CUSTOMER To sync"` as though that had been the request. Use
  `dlake normalsync enable`/`disable` (or the full form with `--enabled` / `--command`) and let the
  tool build the literal. If you ever call the raw endpoint by hand, one wrong capital turns a paying
  customer's data sync off.

- **The row filter is executed by the on-prem agent** against the customer's ERP database, and is
  not validated as SQL by anything in the path. Do not paste a filter you have not read.

- **`normalsync_catalog_table` writes a catalogue every customer on that ERP reads**, and **nothing
  in this API can delete a catalogue row** — cleanup is a DBA task. Check
  `normalsync_available_tables` first; a misspelling is permanent.

- **Table selection is not proof of anything.** Readiness needs the agent installed and four
  `ERP_SQL*` flags populated, and none of them can be set from here. Use `normalsync_readiness`.

- **`commandName` in the grid is the OPPOSITE of `syncStatus`** — it is the toggle button's label,
  not the state.

- **`aiName` is always empty.** It is not a sign that something failed to populate.

- **Disabling a table is silent.** The clone stops updating, and CRM Pro keeps serving the stale rows
  with no error anywhere. Nothing raises an alarm; only the grid remembers.

- **A table missing from the clone tables is not automatically an extract bug.** Check in this order:
  selected (§3) → enabled (`syncStatus`) → not filtered out (`whereClause`) → readiness (§6) → the
  clone table itself. Four of those five are visible from this surface.

- **ODBC customers still need Normal Sync configured.** The ODBC agent fills an intermediary
  database; Normal Sync is what moves it into the clone tables.

- **`plumbsupply` is a live customer.** Never exercise a mutation against it to see what happens —
  every write here changes a production data flow, and disabling a table is invisible from the CRM
  side until someone notices missing records.

---

## 10. Where this sits

| | |
|---|---|
| **This skill** | Which ERP tables Normal Sync clones into the gateway database |
| `dlake-crmpro` | The clone tables → CRM push (CRM Pro), and its field mappings |
| `dlake-txdownloaderpro` | The CRM → source writeback leg (TxDownloaderPro) |
| `dlake-integration-setup` | Standing a new tenant up: registration, CRM connect, ERP connector |
| `dlake` | Operating the lake itself: schema, exposure, keys, the Data API |

Full CLI reference: `dlake guide cli`.
