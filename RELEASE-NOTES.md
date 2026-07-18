# dlake release notes

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
