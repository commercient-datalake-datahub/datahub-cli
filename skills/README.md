# Agent skills for the `dlake` CLI

Drop-in **skills** that brief an AI coding agent on how to drive the Commercient Data Lake / Data Hub
CLI correctly — the right command *ordering*, the non-obvious gotchas, and the HTTP contract for the
Data API. A skill is a small folder your agent loads on demand, turning "figure `dlake` out by trial
and error" into "already knows the happy path."

## Available skills

| Skill | What it covers |
|-------|----------------|
| [`dlake/`](dlake/SKILL.md) | Build and operate a tenant end-to-end: schema → expose → restart → scoped key, the sharp edges (the scoped-key restart, exposed-vs-raw entities, IDENTITY-key limits), and the REST/GraphQL + events contract — all in one [`SKILL.md`](dlake/SKILL.md). |
| [`dlake-integration-setup/`](dlake-integration-setup/SKILL.md) | Stand up a NEW integration: register a tenant, seed it, then drive the setup wizard — server IP, CRM (connect now or later, OAuth included), and the ERP connector that declares Syspro/QuickBooks/SQL Server/ODBC. Covers the wizard's ordering and guards, credential lifetimes, and how to resume days later. |
| [`dlake-txdownloaderpro/`](dlake-txdownloaderpro/SKILL.md) | Set up **and operate TxDownloaderPro** — the CRM→source (writeback) sync agent. Expose its gateway objects to the Data API and scope a key to them, then CRUD the configuration and in-flight transaction rows, edit the field mapping in their JSON/XML columns, and use the filter-operator vocabulary that decides which retrieved CRM records reach the source. Covers exposure-vs-key-scope, `keyFields`, and the `SFUpdated` state machine. |
| [`dlake-normalsync/`](dlake-normalsync/SKILL.md) | Choose which ERP tables **Normal Sync** — the on-prem change-tracking agent for SQL Server 2008 R2+ — clones into the gateway database's `dbo` clone tables. The available-tables dropdown, the two-call add, per-table sync toggles and row filters, the shared-catalogue semantics, and the prerequisites this surface cannot set (so finishing it does not mean the customer syncs). |
| [`dlake-crmpro/`](dlake-crmpro/SKILL.md) | Set up **and operate CRMPro** — the source→CRM (forward) sync agent that pushes ERP data into the supported CRM and e-commerce platforms. Flag-driven: CRUD the configuration and run-history/error tables, read and edit the field mapping, and know which of its tables are reachable as lake views and which deliberately are not. |

Use `dlake/` for a tenant you already have and `dlake-integration-setup/` for one you are
creating. The last three cover the **sync agents**, in pipeline order: `dlake-normalsync/` extracts
the customer's ERP tables into the gateway database's clone tables, `dlake-crmpro/` pushes those
clone tables source→CRM, and `dlake-txdownloaderpro/` moves changes back CRM→source. Most
integrations run Normal Sync plus CRMPro; add TxDownloaderPro when changes made in the CRM must
travel back. Normal Sync, CRM Pro and TxDownloaderPro are three distinct products — if a table is
missing from the CRM, `dlake-normalsync/` tells you whether it is an extract problem and
`dlake-crmpro/` whether it is a push problem.

## Install

A skill is just a folder. Copy it into the directory your harness scans for skills:

| Harness | Skills directory |
|---------|------------------|
| **Claude Code / Cowork** | `.claude/skills/<skill>/` (per-project) or `~/.claude/skills/<skill>/` (global) |
| **OpenAI Codex** | `~/.codex/skills/<skill>/` or your project's skills directory |
| **OpenCode** | `.opencode/skill/<skill>/` |
| **Other harnesses** | wherever the harness discovers skills — keep the `<skill>/SKILL.md` layout intact |

```bash
# from a clone of this repo — install any or all
cp -r skills/dlake                   ~/.claude/skills/   # operate an existing tenant
cp -r skills/dlake-integration-setup ~/.claude/skills/   # set up a new integration
cp -r skills/dlake-normalsync        ~/.claude/skills/   # which ERP tables get cloned
cp -r skills/dlake-crmpro            ~/.claude/skills/   # the source→CRM sync agent
cp -r skills/dlake-txdownloaderpro   ~/.claude/skills/   # the CRM→source sync agent
```

Confirm your harness's exact path against its own docs — skill-loading conventions still vary between
tools and change over time.

## How it works

Each skill is a `SKILL.md` with YAML frontmatter (`name` + `description`) followed by instructions.
The **`description` is the trigger**: your agent reads it and decides whether the skill is relevant to
the task at hand — so it fires on "build a backend on the data lake" or "why is my `dlk_` key 403-ing?"
without you naming the skill. The whole skill is one file here; if it grows, split rarely-needed detail
into a `references/` subfolder so it loads only when relevant (progressive disclosure).

```
dlake/
└── SKILL.md                 # trigger + build sequence + rules + Data API HTTP contract
```

## See also

- **CLI reference:** `dlake guide cli` · **live API guide:** `dlake guide api`
- Install the CLI: `npm install -g @commercient/dlake`
- Questions / issues: [support@commercient.com](mailto:support@commercient.com)
