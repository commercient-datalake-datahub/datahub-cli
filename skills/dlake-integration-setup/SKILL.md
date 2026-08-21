---
name: dlake-integration-setup
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

Steps 3–5 are **admin-plane tools** (`dlake admin registration_*`). They authenticate with the
tenant's API key and are safe to re-run — the wizard state lives on the server, so you can stop and
resume at any point.

---

## 1. Register the account

```bash
printf '%s' "$PW" | dlake register start \
  --email <email> --company "<Company Name>" --phone <tel> --password-stdin \
  --consent-crm-backup --consent-erp-backup --consent-phone
```

**The three affirmation flags are the registrant's decision, not yours.** The two backup flags are
the registrant's guarantee that THEY have made their own backups of their CRM and ERP data (they do
not ask Commercient to make any backup); `--consent-phone` consents to being contacted by phone.
The server refuses `start` unless all three are present. Ask the human and pass the flags only
after they have agreed — never pass them unprompted.

**FIRST, ask the customer what type of setup this is.** There are two answers, and the instance
type follows from it — never guess:

1. **An integration** (their CRM and/or ERP will sync through the platform): pass their real ERP as
   `--instance-type` (e.g. `SAGEINTACCT`), so the CRM and connector setup match their ERP from the
   first step. The Data Lake and the bootstrap API key arrive exactly the same.
2. **A standalone Data Lake** (no integration): omit `--instance-type` — the default (Commercient
   Data Lake) is the correct instance type for that, not a fallback.

If the customer does not know their ERP yet, treat it as case 2 for now; the ERP can still be
declared at step 5 via `--erpName` on the connector step.

**Keep the password.** `dlake register login` needs that exact string to resume from another machine
or after the local token expires. Generating one inline and piping it straight in leaves it
unrecoverable:

```bash
PW=$(openssl rand -base64 18)     # store $PW somewhere durable BEFORE the shell exits
```

`--email`, `--company` and `--phone` are all required. The response gives the `userId` and the
**application slug** (derived from the company name) — that slug is the tenant name from here on.

## 2. The human clicks the verification link

Nothing progresses until they do. The link is in the inbox you registered, opens a verification
page, needs no sign-in, and is safe to click twice. There is no CLI verify command by design.

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
dlake admin create_api_key --keyName "<tenant>-setup" --expirationDays 90
```

Leaving this until after expiry creates a chicken-and-egg: minting a key requires a key.

## 4. Wizard step 3 — capture the server IP

```bash
dlake admin registration_state                                  # where am I?
dlake admin registration_capture_server_ip --ipAddress <ip>
```

`registration_state` is the authoritative answer to where a registration got to, and the right first call
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

**Deferring the OAuth handshake is fully supported — but it is the user's decision, not yours.** The
customer may not have their CRM admin available, or the authorizing person may be someone else
entirely. Offer the handshake first; defer only when the user chooses to. If deferring: select the
CRM, finalize step 4, carry on to step 5, and complete the authorization whenever the customer is
ready — the tenant remembers the CRM choice. A deferred handshake must be named in your summary:
CRMPro cannot push to the CRM until it is completed.

**There are two legs, and the difference is where the provider sends the callback.** Check the CRM's
`callbackMode` in the catalog:

| `callbackMode` | Callback goes to | Use when |
|---|---|---|
| `loopback` | A listener the CLI binds on `127.0.0.1:8801-8803` | The browser is on the same machine as the CLI |
| `website` | The platform's own public callback endpoint | The provider will not redirect to a loopback address |
| `both` | Either — you choose per handshake | HubSpot; see below |

**Pick the leg BEFORE running anything.** The default command shape is the loopback leg, and on the
wrong provider it fails only after the customer is already staring at an error:

- **HubSpot: always the website leg** (`--no-browser`). Its OAuth apps are per-ERP, and they register
  the platform's callback URL, not the CLI's loopback ports. A loopback attempt fails **pre-consent**
  with HubSpot's "redirect URL doesn't match" page — if you see that page, you picked the wrong leg;
  re-run with `--no-browser`, do not retry the same command.
- **Loopback only when both are true:** the CRM's app is known to register the loopback URIs *and* the
  browser is on the machine running `dlake`.
- When in doubt on a `both` CRM, the website leg always works — it uses the redirect every provider app
  already registers.

```bash
dlake registration oauth --crm <Crm> --no-browser         # website leg: print the URL, poll, complete
dlake registration oauth --crm <Crm>                      # loopback leg: listen on 127.0.0.1, complete
dlake registration oauth --crm <Crm> --fields @start.json # when the CRM needs start inputs
```

**On a `both` CRM, `--no-browser` selects the website leg** — the start call sends no redirect URI, so
the platform authorizes against its own public callback and the code is delivered server-side. The
consent can happen in any browser, on any network — right for HubSpot (above) and whenever the person
authorizing is not sitting at the machine running `dlake`; the CLI picks the result up by polling.

**On the loopback leg** the same command without `--no-browser` binds the listener on 127.0.0.1:8801
(falling back to 8802 then 8803), starts the flow against the port it actually bound, opens the
browser, verifies the returned `state`, and completes the exchange itself.

A callback whose `state` does not match is refused and nothing is exchanged — re-run the command for a
fresh handshake. State tokens are single-use and expire 30 minutes after the start.

The individual tools remain available, and are what to use on the website leg or when the browser and
the terminal are on different machines. To connect at any point, including long after step 4 is
finalized:

```bash
# 1. Where does this CRM's handshake stand? (none / pending / awaiting PIN / connected)
dlake admin registration_crm_oauth_status --crmName <Crm>

# 2. Begin — returns the authorization URL for the customer to open in a browser,
#    plus the `state` to keep for step 3.
#    LOOPBACK LEG: --redirectUri is REQUIRED, and only ports registered with the
#    providers are accepted: 8801, 8802, 8803.
dlake admin registration_crm_oauth_start --crmName <Crm> \
    --redirectUri http://127.0.0.1:8801/callback/

#    WEBSITE LEG: omit --redirectUri entirely. Allowed for callbackMode
#    "website" and "both"; the platform supplies its own registered callback.
dlake admin registration_crm_oauth_start --crmName <Crm>

# 3a. Finish. On the loopback leg pass the code your listener caught; on the
#     website leg pass only --state — the code was delivered to the platform and
#     is read from the parked attempt (status never returns it).
dlake admin registration_crm_oauth_complete --crmName <Crm> --state <state> [--code <code>]

# 3b. …then, for providers whose flow ends in a PIN, confirm it
dlake admin registration_crm_oauth_confirm_pin --crmName <Crm> --pin <pin>
```

`registration_crm_oauth_status` is safe to call at any time and is the right way to check whether a
deferred connection has since been completed. On the website leg it is also how you learn the browser
leg finished: poll until it reports `code_received`, then run `oauth_complete`. Which of
`oauth_complete` / `oauth_confirm_pin` applies is visible in the CRM's `stages` in the catalog — a
`pin` stage means the flow ends with a PIN.

**`code_received` is NOT success — it is a countdown.** It means the provider's authorization code is
parked and waiting, and provider codes are single-use and expire within minutes. Run `oauth_complete`
**immediately**; do not finalize the step, move on to other wizard work, or wait for anything first.
A code left sitting expires, the exchange then fails, and the only recovery is a full re-authorize —
the customer has to consent all over again. The handshake is done only at `connected`.

**The PIN is emailed, not displayed — and only `oauth_complete` sends it.** For the four PIN providers
(Salesforce, Zoho, HubSpot, Klaviyo) `oauth_complete` parks the tokens and sends a confirmation PIN to
the registration account's email address; nothing shows it on screen, and no email exists until the
complete call runs. Do not go looking for it on the callback page — that page only says the
authorization was recorded — and do not go looking for a "send/resend PIN" command: **there is none.**
PIN delivery is a side effect of `oauth_complete`; the only step after it is
`oauth_confirm_pin --pin <pin>` with the code from the email.

**Two different refusals, in this order.** Read the code, not the prose:

| Code | Status | Meaning |
|---|---|---|
| `missing_redirect_uri` | 400 | You sent no callback URL to a `loopback`-only CRM. Supply `--redirectUri` as above. A `website` or `both` CRM never answers this — omitting the redirect is how you ask for the website leg. |
| `invalid_redirect_uri` | 400 | The URL you sent is not an http loopback address. It is used verbatim, so the guard applies whenever you supply one. |
| `provider_app_not_configured` | 424 | The platform holds no OAuth app credentials for this provider. Nothing you can fix from here. |

The first two are yours to correct; the third is platform-side. You reach it only *after* supplying a
valid redirect URI, so a run that stops at `missing_redirect_uri` has not yet learned anything about
the provider's configuration.

On `provider_app_not_configured`: report it, carry on with the rest of the setup, and complete the
authorization later with the same commands. Nothing about the integration is lost by finishing that
part afterwards — and you do **not** need to substitute a different CRM to close step 4. Selecting the
CRM and running `crm_finalize` completes the step with the handshake still at `status: none`.

## 6b. Provider sub-actions — Salesforce package install, Monday workspace

Two CRMs need one extra call after the authorization, and the connection is INCOMPLETE without it
even though every other tool reports success:

```bash
# Salesforce: the Commercient managed-package install URLs. The customer must install
# the package in their org, or the sync has nothing to talk to.
dlake registration action --crm Salesforce --action SalesforceSetup

# Monday: list the workspaces, then pick one.
dlake registration action --crm Monday --action MondayListWorkspaces
dlake registration action --crm Monday --action MondayPickWorkspace --fields @workspace.json
```

The server validates the action by name; only those three exist. The same call is
`dlake admin registration_crm_action --crmName <Crm> --action <Action> [--fields @f.json]`.
The catalog advertises which stages need it as `SubmitAction: "CRMAction:<name>"`.

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

### Placeholder ERP details are supported — but ASK before using them

Customers frequently do not have their ERP server, database and credentials to hand at this point.
The platform supports submitting placeholders: neither `connector_submit` nor `connector_provision`
opens a connection to the ERP — submit parks the configuration, provisioning sets up the tenant
around it, and a provisioning run completes normally with placeholder values.

**Placeholders are the user's decision, not yours.** Always ask for the real ERP credentials first;
submit placeholders only when the user says they are not available (or explicitly tells you to defer).
Never silently substitute placeholders and move on — a setup that looks finished but can never sync is
worse than one that visibly waits on an answer. When placeholders ARE used, say so out loud in your
summary and record that **no data will flow until the real values are re-submitted**.

Do not read a completed provisioning run as "the ERP connection works" — it means the tenant is
configured. The credentials are exercised when data actually syncs.

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
| `registration_status` | any | Account verification and provisioning progress |
| `registration_capture_server_ip` | 3 | Record the customer's server IP |
| `registration_crm_catalog` | 4 | CRMs, their auth modes and fields |
| `registration_crm_select` | 4 | Choose the CRM |
| `registration_crm_connect` | 4 | Submit credential/apikey fields |
| `registration_crm_oauth_start` | 4/any | Begin an OAuth handshake |
| `registration_crm_oauth_complete` | 4/any | Finish with the provider's `code`/`state` |
| `registration_crm_oauth_confirm_pin` | 4/any | Confirm an out-of-band PIN |
| `registration_crm_oauth_status` | 4/any | Where a handshake stands |
| `registration_crm_action` | 4/any | Provider sub-actions: `SalesforceSetup`, `MondayListWorkspaces`, `MondayPickWorkspace` |
| `registration_crm_finalize` | 4 | Close step 4 |
| `registration_connector_catalog` | 5 | ERP connectors and their fields |
| `registration_connector_submit` | 5 | Submit config; `--erpName` declares the ERP |
| `registration_connector_provision` | 5 | Start provisioning |
| `registration_provisioning_status` | 5 | Poll the provisioning run |

## Things that bite

- **Guessing the instance type instead of asking.** Ask whether this is an integration (pass the
  real ERP) or a standalone Data Lake (omit the flag — the default is the correct choice there,
  not a fallback). Either way the Data Lake is created; what differs is whether the CRM and
  connector setup match the customer's ERP from the first step.
- **A generated password piped straight into `register start`.** Keep it; `register login` needs it.
- **Assuming the wizard steps are independent.** They are ordered and guarded; a 409 naming a step is
  an instruction, not a failure.
- **Letting the 7-day bootstrap key expire mid-setup.** Mint a durable key early.
- **Skipping the ERP credentials without asking.** Ask for the real values first. If the user says
  they aren't available, placeholders let setup finish (provision now, re-submit real values later) —
  but that is the user's call to make, and a placeholder setup must be flagged as "will not sync yet"
  in your summary.
- **Reading a completed provisioning run as proof the ERP connection works.** It isn't; the
  credentials are exercised when data syncs.
- **`--fields` as a JSON string.** It takes an object; use `@file.json`.
- **Treating a deferred OAuth connection as a blocked setup.** It isn't — finish the rest and
  connect the CRM when the customer is ready.
