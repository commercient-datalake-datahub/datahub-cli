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
| [`dlake-txdownloaderpro-setup/`](dlake-txdownloaderpro-setup/SKILL.md) | Set up the **TxDownloaderPro writeback leg** — the CRM→source direction of an integration. Expose the gateway sync-state objects (`TxDownloaderPro`, `TxDownloaderProTrans`, `TxDownloaderProBlockDuplicateId`, `TimeStampRepository`, `TimeStampRepositoryHistory`) to the Data API, scope an API key to them, and apply it with one deliberate DAB restart. Covers exposure-vs-key-scope, `keyFields`, and the `SFUpdated` state-machine warning. |

Use `dlake/` for a tenant you already have; use `dlake-integration-setup/` for one you are
creating; use `dlake-txdownloaderpro-setup/` when that integration needs writeback.

## Install

A skill is just a folder. Copy it into the directory your harness scans for skills:

| Harness | Skills directory |
|---------|------------------|
| **Claude Code / Cowork** | `.claude/skills/<skill>/` (per-project) or `~/.claude/skills/<skill>/` (global) |
| **OpenAI Codex** | `~/.codex/skills/<skill>/` or your project's skills directory |
| **OpenCode** | `.opencode/skill/<skill>/` |
| **Other harnesses** | wherever the harness discovers skills — keep the `<skill>/SKILL.md` layout intact |

```bash
# from a clone of this repo — install either or both
cp -r skills/dlake             ~/.claude/skills/   # operate an existing tenant
cp -r skills/dlake-integration-setup  ~/.claude/skills/   # set up a new integration
cp -r skills/dlake-txdownloaderpro-setup ~/.claude/skills/   # add the writeback leg
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
