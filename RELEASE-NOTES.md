# dlake release notes

## 0.5.9 (2026-08-15)

- **New: `dlake crmpro`** — manage your CRMPro sync from the command line: check status, turn sync on or off, list your sync processes, and view history and errors. Run `dlake crmpro --help` for the full list.
- Requires an Admin API key.
- The built-in CRMPro guide is updated: `dlake skills show dlake-crmpro`.

Upgrade: `npm install -g @commercient/dlake`


## 0.5.8 (2026-0 8-14)

- **Fixed: `dlake status` reported eve ry host as blocked when the platform was
  fi ne.** The health check was acquiring an API t oken before probing, so if one
  service was  unreachable the check failed for *all* of the m — including hosts whose
  commands worked  seconds later. Health checks now send no cre dential, and each host
  is judged on its own .
- Corrected in the bundled setup skill: cre ating a key is
  `create_api_key --keyName <n ame> --expirationDays <7|30|90|180>` (both re quired),
  and connecting an OAuth CRM needs  `--redirectUri http://127.0.0.1:8801/callback /`
  — without it you get `missing_redirect _uri`, which is a step earlier than the
  pro vider-side `provider_app_not_configured`. The  skill now lists all three codes
  and says w hich are yours to fix.
- Also clarified: you  do **not** need to pick a different CRM to fi nish the CRM
  step. Select it, run `crm_fina lize`, and complete the authorization later.
 
## 0.5.7 (2026-08-14)

**Please read the fir st item — it changes how you run every comm and.**

- **You now name the tenant on every  command: `--profile <name>`.** There is no
   default profile any more, and `dlake profiles  use` is gone. Set `DLAKE_PROFILE`
  once if  you'd rather not repeat the flag in a shell s ession.
  Why: previously a command could run  against a tenant you hadn't named. Register  a
  new tenant on a machine that already had  one, and the very next command — the one
   the CLI itself printed — went to the *old*  tenant, with nothing on screen to say
  so. I f you look after several customers, that is t he wrong customer.
- **The skills ship inside  the CLI.** `dlake skills list`, `dlake skill s install`,
  `dlake skills show <name>`. Fou r guides: using the Data Lake, setting up a n ew
  integration, and operating each of the t wo sync agents. No download, no internet.
- * *`--help` works when your key doesn't.** Tool  help is served from a local cache
  when the  live call fails, clearly marked as cached. D ocumentation shouldn't need
  the credential  you're trying to fix.
- **After registering,  the key check is trustworthy.** It retries be fore saying
  anything, and if it still can't  confirm, it tells you the key is saved and t he
  registration complete — the *check* is  what's inconclusive.
- **Fewer misleading me ssages.** A blocked network no longer suggest s you raise a
  firewall ticket when the caus e may be an endpoint we pointed you at. Wizar d output
  no longer shows a CRM you never ch ose, or a "not seeded" flag for a lake that i s.
- Old profiles pointing at a download host  that only answers on certain networks are
   moved to the public one automatically.
- `dla ke profile` works as well as `dlake profiles` .
- `npm install -g` now shows progress inste ad of sitting silent for minutes.

Still need s an allowlisted network: `query`, `export`,  `views`, `s3`, `keys`,
`projects`. `dlake` te lls you when that's what you're hitting.

##  0.5.6 (2026-08-13)

- **Fixed: `dlake admin`  and `dlake tool` now work from any network.**  They used
  to fail with `'<' is an invalid  start of a value` unless you were on an
  all owlisted network. Both now go through the pub lic host, so a new customer can
  drive the s etup wizard from anywhere. Just upgrade — n othing to reconfigure.
- **Better errors when  something blocks you.** If a network filter  answers instead
  of the API, `dlake` now say s so and tells you what to ask support for, r ather
  than printing a parser error. `dlake  status` no longer reports a service as
  "dow n" when it is your network being challenged.
 - **`dlake status` shows each host separately **, so you can see at a glance which
  comman ds will work.
- **After registering, the CLI  now tests your new API key** instead of just
   claiming everything works.
- **Clearer guid ance when you have no profile yet** — if yo ur registration is
  still finishing, it poin ts you at `dlake register status` instead of  a login you
  cannot complete.
- `dlake profi le` now works as well as `dlake profiles`.
-  Help fixes: `--phone` is marked required; the  scripted sign-up example keeps the
  passwor d you will need later to resume; the two prog ress numbers now say what
  each one measures .
- **`SHA256SUMS` verifies correctly on Linu x and macOS.** It shipped with Windows
  line  endings, so `sha256sum -c` reported every fi le as failed even when the
  download was fin e.

Still requires an allowlisted network: `q uery`, `export`, `views`, `s3`, `keys`,
`proj ects`. `dlake` now tells you when that is wha t you are hitting.

## 0.5.5 (2026-08-13)

-  New: `dlake register login --email <you>` —  resume a registration from any
  machine. Si gns back in with the password you chose at `r egister start` and
  refreshes the saved toke n, so `register status` and `register resend`  work
  again with no flags. Use `--password- stdin` for scripts; you'll be prompted
  othe rwise.
- After your Data Lake is seeded, the  registration wizard (CRM and ERP connector
   setup) can also be driven through `dlake admi n` — run `dlake admin list` and
  look for  the `registration_*` tools.

## 0.5.4 (2026-0 8-12)

- Registration and updates now use `da talake-ms-dab.commercient.com`. On some
  net works the old host could block `dlake registe r`, `npm install` and CLI
  updates; the new  host works everywhere. No action needed — o verride with
  `--api-base` / `DLAKE_DOWNLOAD _BASE` if you use a custom host.
- No changes  to commands, authentication, or workflows.
-  macOS binaries are code signed as usual. If  you downloaded a macOS build on
  2026-08-12  and it fails to start, simply re-download it. 

## 0.5.3 (2026-08-01)

- Clearer error mess ages for tool calls.
- `dlake entities list`  now shows whether each entity is currently be ing served.
- Docs and embedded links refresh ed.

## 0.5.2 (2026-07-31)

- macOS binaries  are now code signed. Fixes an intermittent st artup failure on
  Apple Silicon. If an older  download misbehaves, re-sign it once locally :
  `xattr -dr com.apple.quarantine ./dlake & & codesign --force --sign - ./dlake`
  (see ` dlake guide cli`). No command changes.

## 0. 5.1 (2026-07-26)

- New `dlake register` comm and group — create a Data Lake account enti rely from
  the terminal (`register start`, ` register status [--watch]`, `register resend` ).
- Array and object arguments now accept JS ON, and any argument accepts `@file`
  (e.g.  `--columns @columns.json`) — the recommende d form on Windows.
- Two new platforms: macOS  Intel (`osx-x64`) and Linux ARM (`linux-arm6 4`) —
  five supported in total.
- Per-tool  `--help` now documents the argument forms.

 ## 0.4.1 (2026-07-25)

- Fix: `dlake tool` co mmands work reliably again.
- Cleaner MCP err or output.

## 0.4.0 (2026-07-24)

- `dlake t ool` — generic data-plane passthrough (reco rds, query, aggregate,
  export, time travel,  documents), completing MCP parity.
- `dlake  guide` — the platform docs (`api`, `help`,  `cli`) printed to stdout.
- `login --mcp-url`  per-profile override.

## 0.3.0 (2026-07-19) 

- `dlake s3 attach` / `detach` / `discover`  — external tables on SQL Server
  2022+ te nants.
- New `dlake view` command group (`lis t`, `show`, `create`, `alter`, `drop`).
- `dl ake --version`.
- Install and credential-hand ling improvements.

## 0.2.1 (2026-07-18)

-  Runs on minimal Linux images (slim/alpine) wi th no extra packages beyond
  `ca-certificate s`. Drop-in replacement for 0.2.0.

## 0.2.0  (2026-07-17)

- Object storage: the `dlake s3 ` command group — connections, browse (`ls` ),
  streaming `put`/`get`, `rm`, and server- side table export into a bucket.

## 0.1.0 � � initial release

- API-key login + per-tena nt profiles, API keys & projects, `query`, `e xport`,
  `entities list`, `status`, and the  `dlake admin` control-plane passthrough.
- Pl atforms: win-x64, linux-x64, osx-arm64. npm w rapper `@commercient/dlake`.
 