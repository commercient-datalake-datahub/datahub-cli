---
name: dlake-onboarding
description: >-
  Set up a NEW Commercient integration end-to-end with the `dlake` CLI (npm `@commercient/dlake`):
  register a tenant, verify by email, wait for the Data Lake to seed, then drive the setup wizard —
  capture the server IP, choose and connect a CRM (HubSpot, Salesforce, Dynamics, Zoho, Trello,
  WooCommerce, Shopify and more), and declare the ERP connector (Syspro, QuickBooks, SQL Server,
  ODBC…) so data syncs between them. Use this skill whenever the task is onboarding a NEW customer
  or standing up a NEW ERP↔CRM integration, resuming a half-finished registration, or connecting a
  CRM's OAuth later. For operating an EXISTING tenant — schema, queries, exports, API keys — use the
  `dlake` skill instead. It encodes the wizard's ORDERING, its guards, and the credential lifetimes
  that otherwise cost repeated failed calls.
---

# Setting up an integration with `dlake`

This skill covers the **one-time journey** that turns a new sign-up into a working ERP↔CRM
integration. It is a different job from operating a lake you already have: the steps are ordered,
the server enforces that order, and two of the waits need a human.

The shape of it:

```
register start → [human clicks email link] → seed + bootstrap key
    → step 3  capture server IP
    → step 4  choose CRM   (connect now, or connect later — both supported)
    → step 5  declare ERP connector → provision
```

Steps 3–5 are **admin-plane MCP tools** (`dlake admin registration_*`). They authenticate with the
tenant's API key, work from any network, and are safe to re-run — the wizard state lives on the
server, so you can stop and resume at any point.

---

## 1. Register the account

```bash
printf '%s' "$PW" | dlake register start \
  --email <email> --company "<Company Name>" --phone <tel> --password-stdin
```

**Do NOT pass `--instance-type`.** Register as the default (Commercient Data Lake) account. That is
what makes the platform seed a Data Lake and issue the bootstrap API key, and **every later step
needs that key**. The customer's real ERP is declared at step 5 — that is the designed place for it,
and it is what `--erpName` on the connector step is for. Passing an ERP at registration instead
produces an account the seeder does not pick up, and the run stalls with nothing to act on.

**Keep the password.** `dlake register login` needs that exact string to resume from another machine
or after the local token expires. Generating one inline and piping it straight in leaves it
unrecoverable:

```bash
PW=$(openssl rand -base64 18)     # store $PW somewhere durable BEFORE the shell exits
```

`--email`, `--company` and `--phone` are all required. The response gives the `userId` and the
**application slug** (derived from the company name) — that slug is the tenant name from here on.

## 2. The human clicks the verification link

Nothing progresses until they do. The link is in the inbox you registered, opens a page hosted by
the Registration API, needs no sign-in, and is safe to click twice. There is no CLI verify command
by design.

## 3. Wait for the seed, and catch the key

```bash
dlake register status --watch
```

Progression is `emailVerified` → `isProvisioned` → `dataLakeSeeded`, typically a couple of minutes
after the click. On the first poll **after** seeding the response carries a **once-only bootstrap API
key**, which the CLI prints and saves to a profile automatically. It is never shown again.

A second "welcome" email also arrives carrying the tenant owner's temporary **web app** password.
That is for the human and the browser; agents use the API key.

Two numbers appear in status output and they measure different things — `setupProcess` is account
provisioning (8 = gateway ready), `currentStep` is the position in the setup wizard below. They are
not the same scale and do not track each other.

### Mint a longer-lived key now

The bootstrap key expires in **7 days**. If setup will span longer — waiting on ERP credentials is
common — create a durable key while the bootstrap one is still valid:

```bash
dlake admin create_api_key --name "<tenant>-setup"
```

Leaving this until after expiry creates a chicken-and-egg: minting a key requires a key.

## 4. Wizard step 3 — capture the server IP

```bash
dlake admin registration_state                                  # where am I?
dlake admin registration_capture_server_ip --ipAddress <ip>
```

`registration_state` is the authoritative answer to "where did we get to" and the right first call
whenever you resume. Step 3 is a **prerequisite for the CRM step** — attempt step 4 first and the
server answers `"Please complete step 3 first. [registration status 409]"`. That 409 is the wizard
telling you the order; satisfy it rather than working around it.

## 5. Wizard step 4 — choose the CRM

```bash
dlake admin registration_crm_catalog                      # what's available, and how each connects
dlake admin registration_crm_select --crmName <Crm>
```

The catalog lists every supported CRM with its `authMode`, which tells you how it connects:

| `authMode` | How it connects | Examples |
|---|---|---|
| `oauth` | Browser authorization, sometimes ending in a PIN | HubSpot, Salesforce, Zoho, Shopify |
| `credentials` | A form of fields you submit | Dynamics CRM, Magento |
| `apikey` | One or more keys | Trello, WooCommerce |
| `none` | Nothing to connect | IOT Pulse |

For a **credentials/apikey** CRM, submit the fields the catalog lists for it:

```bash
dlake admin registration_crm_connect --crmName <Crm> --fields @fields.json
```

For an **OAuth** CRM, either connect now (below) or **defer it** and come back — see the next
section. Then close the step:

```bash
dlake admin registration_crm_finalize
```

## 6. Connecting an OAuth CRM — now or later

**Deferring the OAuth handshake is fully supported and often the right call.** The customer may not
have their CRM admin available, the authorizing person may be someone else entirely, or you may
simply want the ERP side configured first. Select the CRM, finalize step 4, carry on to step 5, and
complete the authorization whenever the customer is ready — the tenant remembers the CRM choice.

To connect at any point, including long after step 4 is finalized:

```bash
# 1. Where does this CRM's handshake stand? (none / pending / awaiting PIN / connected)
dlake admin registration_crm_oauth_status --crmName <Crm>

# 2. Begin — returns the authorization URL for the customer to open in a browser
dlake admin registration_crm_oauth_start --crmName <Crm>

# 3a. Finish with what the provider hands back
dlake admin registration_crm_oauth_complete --crmName <Crm> --code <code> --state <state>

# 3b. …or, for providers whose flow ends in a PIN the user reads off the screen
dlake admin registration_crm_oauth_confirm_pin --crmName <Crm> --pin <pin>
```

`registration_crm_oauth_status` is safe to call at any time and is the right way to check whether a
deferred connection has since been completed. Which of `oauth_complete` / `oauth_confirm_pin` applies
is visible in the CRM's `stages` in the catalog — a `pin` stage means the flow ends with a PIN.

Some OAuth providers require the platform's redirect URI to be registered on their side before the
handshake can complete. If `oauth_start` reports a problem with the redirect URI, that is a
platform-side app registration matter rather than anything wrong with the tenant or the run: report
it, continue with the rest of the setup, and complete the authorization afterwards using the same
commands. Nothing about the integration is lost by finishing that part later.

## 7. Wizard step 5 — declare the ERP connector

This is where the customer's real ERP is declared.

```bash
dlake admin registration_connector_catalog                # connectors + the fields each needs
dlake admin registration_connector_submit \
    --connectorType SQL2008ABOVE --erpName SYSPRO7 --fields @fields.json
```

`--erpName` **overrides** the ERP recorded at registration. The response echoes the result, e.g.
`{"erpName":"SYSPRO7","erpChanged":true}` — check it. From that point the provisioning branch, the
sync-agent choice and the Connection Manager record all follow the declared ERP.

`--fields` takes a JSON **object**, not a JSON string. On Windows write it to a file and pass
`@fields.json` rather than fighting shell quoting. For `SQL2008ABOVE` the fields are
`SqlServerName`, `DataBaseName`, `SqlUserName`, `SqlPassword`.

Step 5 is guarded on step 4 being finalized — same 409 shape as before.

Then run it:

```bash
dlake admin registration_connector_provision
dlake admin registration_provisioning_status              # poll until completed or failed
```

### Placeholder ERP details are fine — this is the normal case

Customers frequently do not have their ERP server, database and credentials to hand at this point,
and waiting for them blocks nothing. **Submit placeholders and move on.** Neither `connector_submit`
nor `connector_provision` opens a connection to the ERP: submit parks the configuration, and
provisioning sets up the tenant around it. A provisioning run completes normally with placeholder
values.

Do not read that completion as "the ERP connection works" — it means the tenant is configured. The
credentials are exercised when data actually syncs.

When the real details arrive, submit them again. There is no separate update verb and no need to
redo anything else:

```bash
dlake admin registration_connector_submit \
    --connectorType SQL2008ABOVE --erpName SYSPRO7 --fields @real-fields.json
```

Re-submitting is accepted at any point, including after provisioning has completed. `erpChanged`
reports whether the **ERP itself** changed, so it is `false` when you re-submit the same ERP with
corrected connection values — that is success, not a rejection.

## 8. Resuming later

The wizard is stateful server-side, so a setup can span days. To pick up:

```bash
dlake admin registration_state        # authoritative: current step, ERP, CRM, IP
```

That needs only a valid API key — which is why minting a durable one at step 3 matters.

If the local registration token has lapsed (72h) and you need `register status` or `register resend`
again, mint a fresh one with the password from step 1:

```bash
dlake register login --email <email> --password-stdin
```

That refreshes only the registration-side token. The wizard tools do not use it — they run on the
API key — so the two are independent and you rarely need both.

---

## Quick reference

| Tool | Step | Purpose |
|---|---|---|
| `registration_state` | any | Full wizard state — always the first call when resuming |
| `registration_status` | any | Account/verification/progress from the Registration API |
| `registration_capture_server_ip` | 3 | Record the customer's server IP |
| `registration_crm_catalog` | 4 | CRMs, their auth modes and fields |
| `registration_crm_select` | 4 | Choose the CRM |
| `registration_crm_connect` | 4 | Submit credential/apikey fields |
| `registration_crm_oauth_start` | 4/any | Begin an OAuth handshake |
| `registration_crm_oauth_complete` | 4/any | Finish with the provider's `code`/`state` |
| `registration_crm_oauth_confirm_pin` | 4/any | Confirm an out-of-band PIN |
| `registration_crm_oauth_status` | 4/any | Where a handshake stands |
| `registration_crm_finalize` | 4 | Close step 4 |
| `registration_connector_catalog` | 5 | ERP connectors and their fields |
| `registration_connector_submit` | 5 | Submit config; `--erpName` declares the ERP |
| `registration_connector_provision` | 5 | Start provisioning |
| `registration_provisioning_status` | 5 | Poll the provisioning run |

## Things that bite

- **`--instance-type` at registration.** Don't. The ERP belongs at step 5.
- **A generated password piped straight into `register start`.** Keep it; `register login` needs it.
- **Assuming the wizard steps are independent.** They are ordered and guarded; a 409 naming a step is
  an instruction, not a failure.
- **Letting the 7-day bootstrap key expire mid-setup.** Mint a durable key early.
- **Waiting on the customer's ERP credentials before finishing setup.** Don't — submit placeholders,
  provision, and re-submit the real values later.
- **Reading a completed provisioning run as proof the ERP connection works.** It isn't; the
  credentials are exercised when data syncs.
- **`--fields` as a JSON string.** It takes an object; use `@file.json`.
- **Treating a deferred OAuth connection as a blocked setup.** It isn't — finish the rest and
  connect the CRM when the customer is ready.
