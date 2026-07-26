# dlake release notes

## 0.5.1 — 2026-07-26

- **Self-service sign-up from the terminal** (previously staged as unreleased
  0.5.0): the new `dlake register` command group creates a Commercient Data Lake
  \ Data Hub account without an existing account, key, or browser session —
  `register start --email <you@company.com> --company "<name>" --phone <tel>
  [--instance-type <erp>]`, `register status [--watch] [--interval <secs>]`, and
  `register resend`. The password is taken from a hidden prompt or
  `--password-stdin`; it is deliberately **not** accepted as a flag (shell
  history / process table). `--instance-type` is optional and defaults to the
  Data Lake.
  You click **one** verification link in the e-mail; everything after that is
  automatic — `register status` reports the progress (email verified →
  provisioned → Data Lake seeded), and a second e-mail delivers your Data Lake
  welcome message with the tenant owner's temporary password. On the first
  status poll after seeding the CLI prints a **once-only, 7-day, full
  owner-admin API key** and saves it into your profile, so the same terminal
  immediately drives both the data plane (`dlake tool …`) and the control plane
  (`dlake admin …`, including `create_schema` / `create_table`). Copy the key
  when it is shown — it is never displayed again. Registration is CLI/REST only
  by design; there is no MCP equivalent, because an MCP connection is
  tenant-scoped and needs a key that cannot exist before you register.
- **Fix: array and object arguments now work.** Every argument declared as an
  array used to be comma-split and each fragment coerced as a scalar, so any
  tool taking an array of **objects** — `create_table --columns`,
  `create_index`, `set_entity_exposure`, `aggregate --aggregates` — could not be
  invoked at all: it failed locally before a request was ever sent. An
  array/object argument whose value starts with `[` or `{` is now parsed as JSON
  and sent verbatim, and malformed JSON raises a clear usage error naming the
  argument instead of silently falling back to comma-splitting. The simple comma
  form (`a,b,c`) still works for arrays of scalars.
- **New: `@file` for any argument.** Prefix a value with `@` to read it from a
  file — `--columns @columns.json`. One trailing newline is trimmed, a missing
  file is a clear usage error naming both the argument and the path, and `@@`
  escapes a literal leading `@`. This is the recommended form on Windows, where
  quoted JSON is mangled by the shell before the program ever sees it:

  ```json
  [
    { "name": "Id",        "type": "int",      "identity": true,  "nullable": false },
    { "name": "Number",    "type": "nvarchar", "size": 50,        "nullable": false, "unique": true },
    { "name": "Amount",    "type": "decimal",  "precision": 18,   "scale": 2, "nullable": false, "default": "0" },
    { "name": "CreatedUtc","type": "datetime2","nullable": false, "default": "SYSUTCDATETIME()", "isExpression": true }
  ]
  ```

  ```bash
  dlake admin create_table --table Invoice --columns @columns.json --primaryKey Id
  ```

  Column objects take `{ name, type, size?, precision?, scale?, nullable?,
  default?, isExpression?, identity?, unique?, primaryKey? }`, with the table's
  key given as a top-level `primaryKey` list.
- **Two new platforms — Intel macOS (`osx-x64`) and ARM Linux
  (`linux-arm64`)**, bringing the supported set to five: Windows x64, Linux x64,
  Linux arm64, macOS Apple Silicon, macOS Intel. Both `npm install` and
  self-update resolve them automatically.
- **`<tool> --help` documents the argument conventions** — whenever a tool takes
  an array or object argument, its help now spells out the JSON, comma and
  `@file` forms.

## 0.4.1 — 2026-07-25

- **Fix: `dlake tool` now works.** In 0.4.0 every `dlake tool …` command (`list`,
  `<tool>`, `<tool> --help`) failed with JSON-RPC `-32000` *"A new session can
  only be created by an initialize request…"* — the entire data-plane MCP surface
  was unreachable. The CLI now performs the MCP streamable-HTTP session handshake:
  it sends an `initialize` request first, captures the `Mcp-Session-Id` response
  header, and echoes it on every subsequent `tools/list` / `tools/call` in the
  invocation. `dlake admin` (a separate REST bridge) was never affected.
- **Cleaner MCP errors.** A JSON-RPC tool error now prints as `MCP error <code>:
  <message>` instead of a raw envelope, and a dropped/expired session surfaces as
  *"MCP session handshake failed"* rather than the raw `-32000` text.

## 0.4.0 — 2026-07-24

- **Full MCP parity.** The CLI is now a complete peer of the MCP connector —
  every tool callable, every guide readable.
- **`dlake tool` — generic DATA-plane MCP passthrough** (sibling of `dlake
  admin`): `tool list`, `tool <tool> --help`, `tool <tool> [--arg value ...]`.
  Reaches the per-tenant DAB `/mcp` surface through the auth-proxy (JSON-RPC 2.0,
  JSON **or** SSE), with the same arg-coercion, output and exit-code contract as
  `admin`. Exposes DAB entity CRUD (`read_records`, `create_record`,
  `update_record`, `delete_record`, `describe_entities`, `execute_entity`), the
  analytics tools (`query`, `export_query`, `export_table`, `aggregate`,
  `get_active_schema`, `ingest_table`), the read-only Time-Travel tools
  (`query_as_of`, `row_history`, `time_travel_diff`, `temporal_status`), and the
  document tools.
- **`dlake guide` — platform docs to stdout**: `guide api` (API Usage Guide) and
  `guide help [--section <name>]` (Help docs) fetch the server's live copy via
  the MCP doc tools; `guide cli` prints this CLI reference (bundled at build
  time).
- **`login --mcp-url`** overrides the data-plane MCP host per profile (default
  `https://datalake-ms-dab.commercient.com`); existing profiles pick up the
  default with no reconfiguration.
- Docs: `dlake tool`, `dlake guide` and the (previously undocumented) `dlake
  docs` document-store commands are now in the CLI reference.

## 0.3.0 — 2026-07-19

- **External tables (PolyBase, SQL Server 2022+ tenants)**: new `dlake s3`
  subcommands — `attached` (list what's linked), `attach` (map a parquet/CSV
  object as a queryable external table; columns auto-discovered server-side,
  or pinned with `--columns-file` for a fully deterministic attach), `detach`,
  and `discover` (preview the inferred schema before attaching). CSV options:
  `--no-header`, `--delimiter`; `--format` is inferred from a `.csv` location.
- **SQL views**: new `dlake view` command group — `list`, `show <name>`,
  `create`/`alter <name>` (`--select "<sql>"` or `--select-file <path>`; you
  supply the view body only, the server wraps and validates the DDL), and
  `drop <name>`. Together with `s3 attach --columns-file`, this makes typed
  repair of a broken external table a pure-CLI operation (see the docs'
  worked example).
- **`dlake --version`** now prints the version (previously it showed the
  usage banner).
- **Hardened npm installs** (previously staged as unreleased 0.2.2): the npm
  wrapper verifies each downloaded binary against the release's `SHA256SUMS`
  before installing, enforces an https-only download base, and allowlists
  redirect hosts.
- **Secrets off argv**: credential-taking commands (`login`, `s3 connections
  add`) read secrets from env vars, `--...-stdin`, or an interactive masked
  prompt; bare `--api-key`/`--secret-access-key` values on the command line
  still work but are discouraged (shell history / process table).

## 0.2.1 — 2026-07-18

- **Fix: runs on minimal Linux with no ICU.** The self-contained binary now
  builds with invariant globalization, so it no longer hard-depends on `libicu`
  — it runs on slim/alpine images and stripped servers where earlier builds
  aborted at startup with *"Couldn't find a valid ICU package installed"*.
  (`ca-certificates` is still required for HTTPS on minimal images — install it
  if `login`/`status` report a TLS error.)
- No command or flag changes; a drop-in replacement for 0.2.0.

## 0.2.0 — 2026-07-17

- **Object storage (S3 outlet)**: new `dlake s3` command group — manage bucket
  connections (`connections list|add|test|remove`), browse (`ls`), streaming
  upload/download (`put` / `get`), delete (`rm`), and server-side table export
  straight into a bucket (`s3 export <conn> <entity> --format csv|parquet`).
- Uploads/downloads stream end-to-end (no whole-file buffering); progress on
  stderr, suppressed under `--json`.
- Connection secrets are write-only — never returned by any command.

## 0.1.0 — initial release

- API-key login + per-tenant profiles (`dlake login`, profile switching).
- API keys & projects (`keys list|create|revoke`, `projects list`).
- `query` (raw SQL for keys with AllowRawSql), `export` (csv/parquet),
  `entities list`, `status`.
- Generic MCP admin control-plane passthrough (`admin tools`, `admin call`).
- Platforms: win-x64, linux-x64, osx-arm64. npm wrapper `@commercient/dlake`.
