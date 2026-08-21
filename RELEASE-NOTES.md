# dlake release notes

## 0.5.18 (2026-08-21)

- **New: ODBC Sync setup from the CLI.** Eight `odbcsync_*` admin tools configure
  the sync agent for sources that are not Microsoft SQL Server: the S3 bucket
  registry, the bucket's IP allow-list (the step without which the agent cannot
  upload), the agent's configuration file, and two verification reads (what the
  agent has staged, and the last 24 hours of errors).

  ```
  dlake admin odbcsync_status
  dlake admin odbcsync_register_bucket --bucketName <name> --confirm true
  dlake admin odbcsync_allow_ip --bucketName <name> --ipAddress <ip> --confirm true
  dlake admin odbcsync_read_agent_config
  ```
- **New: Generic API Sync setup from the CLI.** Six `apisync_*` admin tools turn
  the API-source sync product on and describe the endpoints it calls: enable
  (idempotent), endpoint configurations, the per-ERP endpoint-template catalogue,
  and a hosted customer's real ERP table columns.

  ```
  dlake admin apisync_status
  dlake admin apisync_enable --confirm true
  dlake admin apisync_templates --erpName <erp>
  ```
- Every new tool is Admin-only (reads included), acts only on the key's own
  tenant, and shows its argument schema with `dlake admin <tool> --help`.
- **Two new bundled agent skills** — `dlake-odbcsync` and `dlake-apisync` —
  teach an AI agent both surfaces, including the ordering and the traps (the
  config editor's encrypted settings; the endpoint save's two modes).
- The Help and API guides gain matching sections. Upgrade:
  `npm install -g @commercient/dlake` (or `npm install -g datalake`).
- If your MCP client was already connected, restart it (or re-add the connector)
  to see the new tools — the tool list is cached per session.
## 0.5.17 (2026-08-21)

- **Updated bundled agent skills.** The integration-setup skill now gives an AI
  agent much firmer guidance through the CRM OAuth step:
  - which callback style to use per CRM (HubSpot always uses the website
    flow - no local listener needed, and the browser can be on any machine);
  - completing the token exchange promptly once the authorization lands
    (authorization codes are short-lived);
  - that confirmation PINs arrive **by email** after the exchange completes -
    they are never shown on screen;
  - asking you before deferring a CRM connection or submitting placeholder ERP
    credentials, and saying clearly when a setup will not sync yet.
- The npm packages (`@commercient/dlake` and the `datalake` alias) now link
  the GitHub repository and the browsable skills folder from their pages.
- No CLI command changes. Upgrade: `npm install -g @commercient/dlake`
  (or `npm install -g datalake`).
## 0.5.16 (2026-08-19)

- **New: `dlake normalsync` — choose which of your ERP tables get synced, from the
  command line.** Normal Sync is the on-premises agent that copies changed rows
  out of your ERP database and into your Data Lake; this group is the table
  selection for it:

  ```
  dlake normalsync tables                 tables you could add
  dlake normalsync selected               tables you have added, and their settings
  dlake normalsync readiness              is the sync actually set up, and if not why
  dlake normalsync select <table> <id>    add a table to your sync
  dlake normalsync enable | disable <table> <id>   turn one table's sync on or off
  dlake normalsync filter <id> --where "<sql>" --confirm   sync only some rows
  dlake normalsync filter <id> --clear --confirm          sync the whole table again
  dlake normalsync catalog <table> --confirm   register a table name for your ERP
  dlake normalsync add <table> --confirm      register it and add it in one step
  ```

  `catalog`, `add` and `filter` need `--confirm`, and the command stops before
  doing anything if it is missing. Two of those are worth the extra keystroke:
  the table register is shared by everyone using the same ERP and nothing can
  remove an entry from it, and a row filter is run against your ERP database by
  the on-premises agent exactly as you wrote it.
- **The same operations are still available in full.** Every verb has a longer
  form as `dlake admin normalsync_<name>`, with `--help` for its arguments;
  `dlake admin list` names them.
- **Normal Sync commands need the Admin role.** The API key has to belong to a
  user who holds Admin on the tenant. If it doesn't, the command says so plainly
  and exits 3; nothing is changed.
- Adding tables is not the same as syncing them — `dlake normalsync readiness`
  reports what is still missing, including the parts (the agent install and your
  ERP connection details) that these commands cannot set.
- The bundled Normal Sync guide (`dlake skills show dlake-normalsync`) and the
  CLI reference ship updated in this version. Upgrade:
  `npm install -g @commercient/dlake`

## 0.5.15 (2026-08-19)

- Maintenance: corrected the download-host URL in the bundled guide's install
  example and refreshed package links. No functional changes. Upgrade:
  `npm install -g @commercient/dlake` (or `npm install -g datalake`)

## 0.5.14 (2026-08-19)

- Package metadata refresh: the npm package now carries a proper description,
  keywords, and links to the GitHub repository and the official downloads
  site. No functional changes. Upgrade: `npm install -g @commercient/dlake`

## 0.5.13 (2026-08-18)

- **Updated guidance in the bundled guides and skills.** When registering,
  answer one question first: is this an **integration** (your CRM and/or ERP
  syncing through the platform) or a **standalone Data Lake**? For an
  integration, pass your real ERP as `--instance-type` (e.g. `SAGEINTACCT`)
  so the CRM and connector setup match your ERP from the first step; for a
  standalone Data Lake, omit it — the default is the correct choice there.
  Your Data Lake is created either way.
- The bundled guide now covers the TxDownloaderPro management tools available
  through `dlake admin`, and the TxDownloaderPro skill gained a section on
  them.
- No behaviour changes. Upgrade: `npm install -g @commercient/dlake`

## 0.5.12 (2026-08-18)

- **Corrected the wording of the two backup statements on `dlake register start`.**
  `--consent-crm-backup` and `--consent-erp-backup` confirm that **you have made
  your own backups** of your CRM and ERP data — they were previously described
  as consenting to Commercient making backups, which is not what they mean.
  The flags, their behaviour, and the sign-up flow are unchanged;
  `--consent-phone` (consent to phone contact) is unchanged. The help text and
  bundled guides now state the correct meaning. Upgrade:
  `npm install -g @commercient/dlake`

## 0.5.11 (2026-08-17)

**Please upgrade if you use `dlake register` — sign-up now requires consent flags.**

- **Changed: `dlake register start` requires three affirmation flags.**
  Registration now asks for the same three statements as the sign-up website,
  and the server refuses to create an account without them. The backup flags
  confirm that **you have made your own backups** of your CRM and ERP data
  (they do not ask Commercient to make any backup). Pass each flag to affirm
  its statement:

  ```
  --consent-crm-backup    I confirm I have made my own backup of my CRM data
  --consent-erp-backup    I confirm I have made my own backup of my ERP data
  --consent-phone         I consent to being contacted by phone
  ```

  Older CLI versions cannot complete `register start` any more — upgrade:
  `npm install -g @commercient/dlake`. There is also a new optional
  `--document-sync` flag for File-Sync-only accounts.
- **New: `dlake registration oauth` — connect an OAuth CRM in one command.**
  It opens your browser at the provider's authorization page, catches the
  redirect on this machine, and completes the exchange automatically — no more
  copying `code` and `state` out of the address bar. `--no-browser` keeps the
  manual flow if you prefer it, and the command falls back to it by itself when
  a browser can't be opened.
- **New: `dlake registration action` — the provider-specific setup steps.**
  Salesforce's managed-package install links and Monday's workspace picker
  (`SalesforceSetup`, `MondayListWorkspaces`, `MondayPickWorkspace`) are now
  reachable from the CLI; previously these steps could not be completed outside
  the portal.
- The bundled guides and the integration-setup skill are updated to match.

## 0.5.10 (2026-08-15)

- Refreshed the built-in guides and documentation. No functional changes.
- Downloads for older versions have been retired; installs of pinned versions
  below 0.5.10 will no longer fetch binaries. Upgrade:
  `npm install -g @commercient/dlake`

## 0.5.9 (2026-08-15)

- **New: `dlake crmpro` — run CRMPro from the command line.** The everyday verbs
  for the forward ERP→CRM sync, without opening the portal:

  ```
  dlake crmpro processes [--active]     the sync grid
  dlake crmpro process <id>             one process in full
  dlake crmpro status                   is the sync on or off
  dlake crmpro enable | disable         turn it on or off
  dlake crmpro history <id>             what was pushed, when, to which CRM record
  dlake crmpro errors                   why records are missing in the CRM
  dlake crmpro connections              the CRM connections a process can use
  dlake crmpro mapping <id>             a process's field mapping
  ```

  `enable` and `disable` set the state you asked for — run either twice and
  nothing moves — and each prints what the setting was before, what it is now,
  and whether anything changed, so a no-op no longer looks like a success.
- **Everything else CRMPro can do is available too.** Creating, editing and
  deleting processes, importing templates, listing CRM objects and fields, the
  sync flags, the sync agent's own version report — 24 operations in all — are
  there as `dlake admin crmpro_<name>`, each with `--help` for its arguments.
  `dlake admin list` names them.
- **CRMPro commands need the Admin role.** The API key has to belong to a user
  who holds Admin on the tenant. If it doesn't, the command says so plainly and
  exits 3; nothing is changed.
- The bundled CRMPro guide (`dlake skills show dlake-crmpro`) ships updated in
  this version — read it before the editing commands: a few of them protect you
  from behaviour that is not obvious from the arguments alone.

## 0.5.8 (2026-08-14)

- **Fixed: `dlake status` reported every host as blocked when the platform was
  fine.** The health check was acquiring an API token before probing, so if one
  service was unreachable the check failed for *all* of them — including hosts whose
  commands worked seconds later. Health checks now send no credential, and each host
  is judged on its own.
- Corrected in the bundled setup skill: creating a key is
  `create_api_key --keyName <name> --expirationDays <7|30|90|180>` (both required),
  and connecting an OAuth CRM needs `--redirectUri http://127.0.0.1:8801/callback/`
  — without it you get `missing_redirect_uri`, which is a step earlier than the
  provider-side `provider_app_not_configured`. The skill now lists all three codes
  and says which are yours to fix.
- Also clarified: you do **not** need to pick a different CRM to finish the CRM
  step. Select it, run `crm_finalize`, and complete the authorization later.

## 0.5.7 (2026-08-14)

**Please read the first item — it changes how you run every command.**

- **You now name the tenant on every command: `--profile <name>`.** There is no
  default profile any more, and `dlake profiles use` is gone. Set `DLAKE_PROFILE`
  once if you'd rather not repeat the flag in a shell session.
  Why: previously a command could run against a tenant you hadn't named. Register a
  new tenant on a machine that already had one, and the very next command — the one
  the CLI itself printed — went to the *old* tenant, with nothing on screen to say
  so. If you look after several customers, that is the wrong customer.
- **The skills ship inside the CLI.** `dlake skills list`, `dlake skills install`,
  `dlake skills show <name>`. Four guides: using the Data Lake, setting up a new
  integration, and operating each of the two sync agents. No download, no internet.
- **`--help` works when your key doesn't.** Tool help is served from a local cache
  when the live call fails, clearly marked as cached. Documentation shouldn't need
  the credential you're trying to fix.
- **After registering, the key check is trustworthy.** It retries before saying
  anything, and if it still can't confirm, it tells you the key is saved and the
  registration complete — the *check* is what's inconclusive.
- **Fewer misleading messages.** A blocked network no longer suggests you raise a
  firewall ticket when the cause may be an endpoint we pointed you at. Wizard output
  no longer shows a CRM you never chose, or a "not seeded" flag for a lake that is.
- Old profiles pointing at a download host that only answers on certain networks are
  moved to the public one automatically.
- `dlake profile` works as well as `dlake profiles`.
- `npm install -g` now shows progress instead of sitting silent for minutes.

Still needs an allowlisted network: `query`, `export`, `views`, `s3`, `keys`,
`projects`. `dlake` tells you when that's what you're hitting.

## 0.5.6 (2026-08-13)

- **Fixed: `dlake admin` and `dlake tool` now work from any network.** They used
  to fail with `'<' is an invalid start of a value` unless you were on an
  allowlisted network. Both now go through the public host, so a new customer can
  drive the setup wizard from anywhere. Just upgrade — nothing to reconfigure.
- **Better errors when something blocks you.** If a network filter answers instead
  of the API, `dlake` now says so and tells you what to ask support for, rather
  than printing a parser error. `dlake status` no longer reports a service as
  "down" when it is your network being challenged.
- **`dlake status` shows each host separately**, so you can see at a glance which
  commands will work.
- **After registering, the CLI now tests your new API key** instead of just
  claiming everything works.
- **Clearer guidance when you have no profile yet** — if your registration is
  still finishing, it points you at `dlake register status` instead of a login you
  cannot complete.
- `dlake profile` now works as well as `dlake profiles`.
- Help fixes: `--phone` is marked required; the scripted sign-up example keeps the
  password you will need later to resume; the two progress numbers now say what
  each one measures.
- **`SHA256SUMS` verifies correctly on Linux and macOS.** It shipped with Windows
  line endings, so `sha256sum -c` reported every file as failed even when the
  download was fine.

Still requires an allowlisted network: `query`, `export`, `views`, `s3`, `keys`,
`projects`. `dlake` now tells you when that is what you are hitting.

## 0.5.5 (2026-08-13)

- New: `dlake register login --email <you>` — resume a registration from any
  machine. Signs back in with the password you chose at `register start` and
  refreshes the saved token, so `register status` and `register resend` work
  again with no flags. Use `--password-stdin` for scripts; you'll be prompted
  otherwise.
- After your Data Lake is seeded, the registration wizard (CRM and ERP connector
  setup) can also be driven through `dlake admin` — run `dlake admin list` and
  look for the `registration_*` tools.

## 0.5.4 (2026-08-12)

- Registration and updates now use `datalake-ms-dab.commercient.com`. On some
  networks the old host could block `dlake register`, `npm install` and CLI
  updates; the new host works everywhere. No action needed — override with
  `--api-base` / `DLAKE_DOWNLOAD_BASE` if you use a custom host.
- No changes to commands, authentication, or workflows.
- macOS binaries are code signed as usual. If you downloaded a macOS build on
  2026-08-12 and it fails to start, simply re-download it.

## 0.5.3 (2026-08-01)

- Clearer error messages for tool calls.
- `dlake entities list` now shows whether each entity is currently being served.
- Docs and embedded links refreshed.

## 0.5.2 (2026-07-31)

- macOS binaries are now code signed. Fixes an intermittent startup failure on
  Apple Silicon. If an older download misbehaves, re-sign it once locally:
  `xattr -dr com.apple.quarantine ./dlake && codesign --force --sign - ./dlake`
  (see `dlake guide cli`). No command changes.

## 0.5.1 (2026-07-26)

- New `dlake register` command group — create a Data Lake account entirely from
  the terminal (`register start`, `register status [--watch]`, `register resend`).
- Array and object arguments now accept JSON, and any argument accepts `@file`
  (e.g. `--columns @columns.json`) — the recommended form on Windows.
- Two new platforms: macOS Intel (`osx-x64`) and Linux ARM (`linux-arm64`) —
  five supported in total.
- Per-tool `--help` now documents the argument forms.

## 0.4.1 (2026-07-25)

- Fix: `dlake tool` commands work reliably again.
- Cleaner MCP error output.

## 0.4.0 (2026-07-24)

- `dlake tool` — generic data-plane passthrough (records, query, aggregate,
  export, time travel, documents), completing MCP parity.
- `dlake guide` — the platform docs (`api`, `help`, `cli`) printed to stdout.
- `login --mcp-url` per-profile override.

## 0.3.0 (2026-07-19)

- `dlake s3 attach` / `detach` / `discover` — external tables on SQL Server
  2022+ tenants.
- New `dlake view` command group (`list`, `show`, `create`, `alter`, `drop`).
- `dlake --version`.
- Install and credential-handling improvements.

## 0.2.1 (2026-07-18)

- Runs on minimal Linux images (slim/alpine) with no extra packages beyond
  `ca-certificates`. Drop-in replacement for 0.2.0.

## 0.2.0 (2026-07-17)

- Object storage: the `dlake s3` command group — connections, browse (`ls`),
  streaming `put`/`get`, `rm`, and server-side table export into a bucket.

## 0.1.0 — initial release

- API-key login + per-tenant profiles, API keys & projects, `query`, `export`,
  `entities list`, `status`, and the `dlake admin` control-plane passthrough.
- Platforms: win-x64, linux-x64, osx-arm64. npm wrapper `@commercient/dlake`.
