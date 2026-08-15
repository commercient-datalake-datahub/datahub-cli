# dlake release notes

## 0.5.9 (2026-08-15)

- **New: `dlake crmpro`** — manage your CRMPro sync from the command line: check status, turn sync on or off, list your sync processes, and view history and errors. Run `dlake crmpro --help` for the full list.
- Requires an Admin API key.
- The built-in CRMPro guide is updated: `dlake skills show dlake-crmpro`.

Upgrade: `npm install -g @commercient/dlake`

## 0.5.8 (2026-08-14)

- Fixed: `dlake status` could report services as unavailable when they were fine.
- Corrections and improvements to the bundled setup guide.

## 0.5.7 (2026-08-14)

- **You now name the tenant on every command: `--profile <name>`.** There is no default profile any more, and `dlake profiles use` is gone. Set `DLAKE_PROFILE` once if you'd rather not repeat the flag. This ensures a command always runs against the tenant you named.
- **The guides ship inside the CLI.** `dlake skills list`, `dlake skills install`, `dlake skills show <name>` — no download needed.
- `--help` for admin tools works even when your key doesn't, served from a local cache and clearly marked as cached.
- Clearer messages after registering and when a request can't get through.
- `dlake profile` works as well as `dlake profiles`.
- `npm install -g` now shows progress.

## 0.5.6 (2026-08-13)

- `dlake admin` and `dlake tool` now work reliably from any network. Just upgrade — nothing to reconfigure.
- Clearer error messages, and `dlake status` shows each service separately.
- After registering, the CLI tests your new API key instead of just claiming everything works.
- The checksum file now verifies correctly on Linux and macOS.

## 0.5.5 (2026-08-13)

- New: `dlake register login --email <you>` — resume a registration from any machine with the password you chose at sign-up.
- After your Data Lake is ready, setup can also be driven through `dlake admin` (see `dlake admin list`).

## 0.5.4 (2026-08-12)

- More reliable registration, installs and updates on some networks. No action needed.
- macOS binaries are code signed as usual; if a build downloaded on 2026-08-12 fails to start, re-download it.

## 0.5.3 (2026-08-01)

- Clearer error messages for tool calls.
- `dlake entities list` now shows whether each entity is currently being served.
- Docs refreshed.

## 0.5.2 (2026-07-31)

- macOS binaries are now code signed, fixing an intermittent startup failure on Apple Silicon.

## 0.5.1 (2026-07-26)

- New `dlake register` command group — create an account entirely from the terminal.
- Array and object arguments accept JSON, and any argument accepts `@file` (recommended on Windows).
- Two new platforms: macOS Intel and Linux ARM — five supported in total.

## 0.4.1 (2026-07-25)

- Fix: `dlake tool` commands work reliably again.

## 0.4.0 (2026-07-24)

- `dlake tool` — generic data passthrough (records, query, export, documents).
- `dlake guide` — the platform docs printed to stdout.

## 0.3.0 (2026-07-19)

- `dlake s3 attach` / `detach` / `discover` — external tables on supported tenants.
- New `dlake view` command group.
- `dlake --version`.

## 0.2.1 (2026-07-18)

- Runs on minimal Linux images with no extra packages. Drop-in replacement for 0.2.0.

## 0.2.0 (2026-07-17)

- Object storage: the `dlake s3` command group.

## 0.1.0 — initial release

- Login and per-tenant profiles, API keys and projects, `query`, `export`, `entities list`, `status`, and `dlake admin`.
