# dlake release notes

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
