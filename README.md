# dlake — Commercient Data Lake \ Data Hub CLI

`dlake` is the official cross-platform command-line client for the
**Commercient Data Lake \ Data Hub** — in the spirit of the Stripe and HubSpot
CLIs. It authenticates with a tenant API key, manages profiles, and wraps the
platform's REST and admin surfaces with a scripting-friendly UX.

> **Binary distribution.** This repository publishes the official `dlake`
> binaries and release notes. The source code is not published here.

- Human-readable tables by default; `--json` everywhere for scripting.
- Exit codes: **0** ok, **1** error, **2** usage, **3** permission/auth denied.
- Self-contained single-file binaries — no .NET runtime install required.

## Install

### npm (recommended)

```bash
npm install -g @commercient/dlake
dlake login --domain mycompany --api-key dlk_...
```

The package downloads the platform-matched binary on install and exposes it as
the `dlake` command.

### Direct download

Grab the binary for your platform from the latest
[Release](../../releases/latest) (or from
`https://downloads.datalake.commercient.com/downloads/dlake/<version>/<rid>/dlake[.exe]`),
verify it against `SHA256SUMS`, and put it on your `PATH`.

| Platform | Asset |
|---|---|
| Windows x64 | `dlake-win-x64.exe` |
| Linux x64 | `dlake-linux-x64` |
| macOS Apple Silicon | `dlake-osx-arm64` |

## Quickstart

```bash
# Authenticate once per tenant; profiles switch between tenants.
dlake login --domain mycompany --api-key dlk_...
dlake status                        # tenant, key, agent + service health

# API keys & projects
dlake keys list
dlake keys create --name ci-reader  # prints the raw key ONCE

# Query & export
dlake query "SELECT TOP 10 * FROM account" --json
dlake export account --format parquet --out ./account.parquet

# Object storage (S3 outlet)
dlake s3 connections list
dlake s3 ls sales
dlake s3 put sales ./q1.csv reports/
dlake s3 get sales reports/q1.csv ./local.csv
dlake s3 export sales account --format parquet   # server-side table → bucket

# Generic admin control plane (MCP passthrough)
dlake admin tools                   # list every admin tool your key can use
dlake admin call list_tables
```

Run `dlake --help` or `dlake <command> --help` for the full surface.

## Documentation

- **API usage guide** — endpoints, auth, scopes, rate limits: see the Data Lake
  \ Data Hub API guide served from your tenant's Help page.
- **Permissions** — object writes and connection management need
  `data.ingest.manage`; reads accept any `data.ingest.*` tier; scoped API keys
  are enforced server-side (fail-closed) down to entity and field level.

## Configuration

| Env var | Default | Purpose |
|---|---|---|
| `DLAKE_DOWNLOAD_BASE` | `https://downloads.datalake.commercient.com/downloads/dlake` | Binary mirror base (npm installs) |
| `DLAKE_VERSION` | npm package version | Pin a specific binary version |

## Verifying downloads

Every release ships a `SHA256SUMS` file:

```bash
sha256sum -c SHA256SUMS --ignore-missing
```

## Support

Issues and feature requests: open a GitHub issue here, or contact
support@commercient.com.

## License

The `dlake` binaries are proprietary software, free to use with a Commercient
Data Lake \ Data Hub subscription — see [LICENSE](LICENSE.txt).
