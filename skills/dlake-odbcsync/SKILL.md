---
name: dlake-odbcsync
description: >-
  Configure ODBC Sync — the sync agent for a source that is NOT Microsoft SQL Server — with the
  `dlake` CLI and the `odbcsync_*` admin tools: the S3 bucket registry, the bucket's IAM IP
  allow-list, the agent's BridgeClient configuration file, and the two verification reads (staged
  objects, last-24h errors). Use it when the task is "set up ODBC sync", "the agent isn't
  uploading", "edit the agent's config", or "allow-list this customer's IP". ODBC Sync stages data
  into an intermediary database; Normal Sync still moves it into the clone tables — for table
  selection use `dlake-normalsync`, for the CRM push use `dlake-crmpro`, for standing a tenant up
  use `dlake-integration-setup`.
---

# Configuring ODBC Sync, with `dlake`

## 1. When you need this

The customer's ERP source is **not Microsoft SQL Server**, so the plain change-tracking agent
cannot read it. The ODBC agent bridges that: it runs on the customer's premises, reads their
source over ODBC, and stages the data **through an S3 bucket into an intermediary database** —
and Normal Sync then moves that into the `dbo` clone tables like any other customer. **Both
agents run on an ODBC customer.** This skill configures the ODBC half only: *where* the agent
writes (the bucket), *who* may reach the bucket (the IAM policy's IP condition), and *how* the
agent behaves (its `BridgeClient.exe.config`, stored in the bucket).

All eight verbs are the generic admin passthrough:

```bash
dlake --profile <slug> admin <tool> [--arg value ...]
dlake --profile <slug> admin <tool> --help          # the live argument schema
```

Everything here is **Admin only, reads included**, and every tool acts on the profile's own
tenant — there is no userId argument anywhere.

## 2. The mental model — three stores, one agent

| Store | Holds | Who writes it |
|---|---|---|
| The bucket **registry** | which bucket names this customer registered | `odbcsync_register_bucket` |
| The bucket's **IAM policy** | which IPs may upload | `odbcsync_allow_ip` |
| The **config file** in the bucket | how the agent behaves (`BridgeClient.exe.config`) | the agent installer uploads it; `odbcsync_save_agent_config` edits it |

Two things this surface deliberately does NOT have:

- **No delete, anywhere.** A registered bucket name cannot be removed through the API, and there
  is no way to remove an allow-listed IP or empty a bucket. Removals are operator/DBA actions.
- **No AWS verification on registration.** `odbcsync_register_bucket` records the name AS TYPED.
  A typo produces a bucket that reads and writes nothing, silently — the config read then says
  `not_uploaded`. **Check the spelling before anything else.**

## 3. The working path

```bash
# 1. What does this customer have? (record, selected bucket, MERGED bucket registry)
dlake --profile <slug> admin odbcsync_status

# 2. Register the bucket — ONLY if it is not already in the registry
dlake --profile <slug> admin odbcsync_register_bucket --bucketName sync-acme --confirm true

# 3. What is already allow-listed on it?
dlake --profile <slug> admin odbcsync_ip_allowlist --bucketName sync-acme

# 4. Admit the customer's sync server — WITHOUT THIS THE AGENT CANNOT UPLOAD A SINGLE FILE
dlake --profile <slug> admin odbcsync_allow_ip --bucketName sync-acme \
    --ipAddress 203.0.113.9 --confirm true

# 5. Read the agent configuration — check the per-bucket `status`
dlake --profile <slug> admin odbcsync_read_agent_config

# 6. Edit it — send ONLY the settings you are changing (see section 5)
dlake --profile <slug> admin odbcsync_save_agent_config --bucketName sync-acme \
    --settings @settings.json --confirm true

# 7. Verify: is anything actually landing, and is anything erroring?
dlake --profile <slug> admin odbcsync_bucket_data --bucketName sync-acme
dlake --profile <slug> admin odbcsync_errors
dlake --profile <slug> normalsync readiness      # the four-phase honest answer
```

Reading `odbcsync_status`: `bucketList` is TWO registries concatenated (portal-side + the
customer's own gateway `BucketList`, which most customers don't have) — **a name appearing twice
means the registries disagree**, which an operator should see. `ipList` in that payload is always
empty by legacy shape; the real allow-list is step 3.

## 4. Necessary but not sufficient

A perfectly configured bucket still does not sync. ODBC Sync also needs the four `ERP_SQL*`
connector flags (set by ERP connector setup, `dlake-integration-setup` step 5), the agent
installed and running on the customer's machine, and Normal Sync's table selection. Selecting and
configuring here changes none of that — `dlake normalsync readiness` reports ODBC Sync and Normal
Sync as **two separate phases** and names what is missing. Its ODBC phase checks the database
side only (a bucket recorded + the flags); whether the config file exists is
`odbcsync_read_agent_config`'s `status`, and whether the IP is admitted is
`odbcsync_ip_allowlist` — readiness deliberately does not call AWS.

## 5. The config editor — read `status`, and never echo secrets back

`odbcsync_read_agent_config` returns one entry per registered bucket with a `status`:

| `status` | Meaning |
|---|---|
| `loaded` | read and parsed; `settings` holds the contents |
| `not_uploaded` | no config in the bucket — normal before the agent install, **or a typo at registration** |
| `unreadable` | exists but unreadable — permissions, wrong region, AWS down. S3 answers 403 for a MISSING key too, so this cannot be narrowed further |
| `unparseable` | read, but not valid XML |

Saving: each entry is `{settingType, settingName, originalSettingName, settingValue,
settingOperation}` with operation `A` add, `U` update, `D` delete, empty = leave alone; a rename
is a `U` whose `originalSettingName` is the old name. **Send only what you are changing** —
omitted settings are preserved. An operation naming a setting not in the file fails with nothing
written and `errors[]` naming it. Pass `readFileName` exactly as the read reported it (a config
found under `Uploaded.exe.config` is promoted to `BridgeClient.exe.config` on save).

> ### ⚠ The one trap: the encrypted echo
> Ten sensitive settings (AWS keys, SMTP, `ODBCSrcConnectionString`, the Commercient credentials)
> come back from the read **encrypted**, and the save writes **verbatim** — so echoing an
> untouched sensitive value back stores ciphertext where the agent expects plaintext, and the
> agent stops connecting. The tool refuses a batch naming a sensitive setting unless you pass
> `--allowSensitive true`, which states you are supplying a **new plaintext value** on purpose.
> Otherwise: leave sensitive settings out of the batch entirely — omitted is preserved.

## 6. Failure codes worth recognizing

All business failures are HTTP 200 envelopes with `issuccess:false` — read `code`:

| Code | Meaning / action |
|---|---|
| `www_db_unavailable` | Commercient_WWW unreadable — outage **or a missing SQL grant**; the server log has the real reason. Report, don't retry-loop |
| `bucket_not_found` | that bucket is not in THIS customer's registry — register it, or fix the name |
| `bucket_already_registered` | another customer holds that name (409) — pick another |
| `bucket_policy_not_understood` | the IAM policy is not a shape the service will modify — **nothing was written**; operator action |
| `odbc_config_not_uploaded` | nothing to edit yet — agent installer hasn't uploaded, or the bucket name is a typo |
| `odbc_settings_not_applied` | an operation named a setting not in the file — nothing written, `errors[]` names them |
| `gateway_db_unavailable` (errors read) | the tenant isn't provisioned — not "no errors" |

No code ever carries SQL Server's or AWS's own message; the correlation id in the log is where
the provider text lives.
