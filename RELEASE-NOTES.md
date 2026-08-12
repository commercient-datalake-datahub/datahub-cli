# dlake release notes

## 0.5.4 (2026-08-12)

- Registration and self-update now default to the public host
  `datalake-ms-dab.commercient.com`. The previous default
  (`downloads.datalake.commercient.com`) sits behind the estate's
  bot-verification layer, which redirects a bare request to a verification page:
  a browser clears it and never notices, but `dlake register` and the installer
  have no cookie jar and (correctly, fail-closed) refuse the unexpected host. So
  onboarding and CLI updates failed from any network that gets challenged. Same
  bytes, same origin, no such layer. Override with `--api-base` /
  `DLAKE_DOWNLOAD_BASE` as before.
- No change to how the CLI authenticates or reaches the data and control planes:
  register once, then the returned admin key drives the MCP-wrapped commands.
- macOS binaries are ad-hoc code signed, as in 0.5.2 and 0.5.3. (The first
  0.5.4 upload was briefly UNSIGNED because the signer was not on PATH during
  the build; the artifacts and the GitHub release assets were replaced with
  signed builds and the checksum manifests reissued. If you pulled an
  osx-arm64/osx-x64 binary in that window, re-download it.)

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
