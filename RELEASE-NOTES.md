# dlake release notes

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
