---
name: dlake-apisync
description: >-
  Set up Generic API Sync — the sync product for a source that is an API rather than a database —
  with the `dlake` CLI and the `apisync_*` admin tools: enable the product (schema provision +
  flag), write the endpoint configurations the sync agent calls, read the shared per-ERP
  endpoint-template catalogue, and read a hosted customer's real ERP table columns. Use it when
  the task is "enable API sync", "describe which API endpoints sync", or "why does the agent call
  nothing?". Authentication configurations (HOW to authenticate) are the Connection Manager's, a
  different surface — this half stores WHAT TO CALL. For ERP table selection use
  `dlake-normalsync`; for standing a tenant up use `dlake-integration-setup`.
---

# Setting up Generic API Sync, with `dlake`

## 1. When you need this

The customer's source system exposes an **API, not a database**. Generic API Sync teaches the
sync agent which endpoints to call and how. Two halves make the product, and only one is here:

| Half | Stores | Surface |
|---|---|---|
| Connection Manager | **how to authenticate** (encrypted) | its own routes, PortalWeb plane — NOT reachable from these tools |
| **This skill** | **what to call** — endpoint configs + the shared template catalogue | `apisync_*` admin tools |

All six verbs are the generic admin passthrough:

```bash
dlake --profile <slug> admin <tool> [--arg value ...]
dlake --profile <slug> admin <tool> --help          # the live argument schema
```

Everything is **Admin only, reads included**, and every tool acts on the profile's own tenant.

## 2. The working path

```bash
# 1. Is the product on?  (data 1 = enabled, 0 = not; both are success — safe to poll)
dlake --profile <slug> admin apisync_status

# 2. Turn it on: provisions 3 tables + up to 4 procedures IN THE TENANT DB, then sets the flag
dlake --profile <slug> admin apisync_enable --confirm true

# 3. What templates exist for this ERP? (the shared catalogue — a starting point, not required)
dlake --profile <slug> admin apisync_templates --erpName SYSPRO
dlake --profile <slug> admin apisync_template_by_id --id 7

# 4. (hosted customers only) the real columns of an ERP table, for building the config
dlake --profile <slug> admin apisync_table_schema --tableName ArCustomer

# 5. Describe an endpoint — TWO calls, never one (see section 3).
#    objectName is required on BOTH, including the JSON call.
dlake --profile <slug> admin apisync_save_endpoint_config --configId 0 \
    --objectName AR_CUSTOMER --authConfigId 12 --isConfigForFastSync true \
    --syncOrder 1 --isDeleted false --confirm true
dlake --profile <slug> admin apisync_save_endpoint_config --configId <id> \
    --objectName AR_CUSTOMER --configurationJson @endpoint.json --confirm true
```

Reading `apisync_enable`'s result precisely: `skipped` naming `usp_GetGenericAPIData` on a
PostgreSQL tenant is **normal** (the read path is inline there). The three TABLE entries under
`created` mean "the script ran" (their DDL is self-guarding), so on a re-enable judge what
changed by `alreadyPresent` and the PROCEDURE entries. **On failure the product is left OFF** —
`provisioning_failed` means nothing was enabled, and `created` shows how far it got. Enabling is
idempotent and safe to re-run.

**Enabled ≠ syncing.** The agent still needs an authentication configuration (Connection
Manager) that `authConfigId` points at, at least one endpoint configuration, and it has to be
running.

## 3. The one trap: JSON and metadata are separate writes

The upstream save procedure switches mode on whether the configuration JSON is empty — non-empty
writes **the JSON and nothing else** (name, auth id, order and the soft-delete flag are silently
ignored); empty writes **the metadata including `isDeleted`** and leaves the JSON alone. Sending
both would lose the metadata and report success — so the tool **refuses the mix**. Always two
calls:

1. **Metadata leg** — `authConfigId`, `isConfigForFastSync`, `syncOrder`, and `isDeleted`
   **which is required**: the procedure defaults an omitted flag to `1`, i.e. a soft delete
   nobody asked for. A soft delete IS this leg with `--isDeleted true`.
2. **JSON leg** — `configurationJson`.

**`objectName` sits outside that split and is required on both.** The API refuses a blank
`APISyncObjectName` before the mode logic runs (blank rows were found in production, so the
validation was added), which means a JSON call without it can never succeed. On the JSON leg it
is validation only: the procedure ignores it, so the stored name does not change. Send the row's
real object name in both calls.

`configId` `0` creates; a real id updates. The response cannot tell you which happened (the value
means different things on SQL Server vs PostgreSQL) — re-read the row if you need to know.
**The JSON is stored plaintext and the agent reads it directly: describe the endpoint, never put
credentials in it** — they belong in the Connection Manager configuration (encrypted), referenced
by `authConfigId`.

## 4. Templates are shared, append-only, and read-only from here

The template catalogue lives in `Commercient_WWW` and is **shared by every customer of the same
ERP**. From these tools it is read-only — template *authoring* is deliberately an operator-plane
action (the API refuses tenant-scoped callers), because the catalogue is **append-only**: no
update, no delete, and a "correction" is a duplicate row every customer of that ERP then sees.
If a template needs adding or fixing, hand it to the operator rather than looking for a save verb
here.

An empty catalogue is ambiguous by design: it means "no templates for this ERP" **or** "the
`sql/013` script has not been applied to that environment". On live production the script is
applied; on a fresh environment check it before concluding a data gap. Do not confuse this
catalogue with the Connection Manager's **Auth**Template catalogue, one word apart — the wrong
one answers with the fields you wanted empty, not an error.

## 5. `apisync_table_schema` — hosted customers only

The four `ERP_SQL*` flags say *how to reach* the ERP, not that this service has a route to it.
A customer whose ERP lives on their own premises is **structurally unreachable** from the
platform — `erp_unreachable` is the **correct answer** for them, not a defect: do not raise it as
a bug, do not "fix" the flags, do not ask for a firewall change. For those customers the table
and column names in an endpoint configuration are typed by hand. `erp_not_configured` names which
flags are missing — they are set by ERP connector setup (`dlake-integration-setup` step 5), not
here. `columnSize: 0` means "not applicable" (non-character types), and an unknown table is an
empty array with success.

## 6. Failure codes worth recognizing

| Code | Meaning / action |
|---|---|
| `not_provisioned` | the tenant has no gateway database yet — finish provisioning first |
| `provisioning_failed` | the tenant DB rejected a schema object; the product is OFF; `created` shows progress |
| `save_failed` | the stored procedure reported no write |
| `not_found` | no template with that id |
| `erp_not_configured` | the message names the missing `ERP_SQL*` flags |
| `erp_unreachable` | flags set but no route/credentials — CORRECT for on-premise customers (section 5) |

Business failures are HTTP 200 envelopes — read `issuccess` and `code`, never the HTTP status.
