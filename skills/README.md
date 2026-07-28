# Agent skills for the `dlake` CLI

Drop-in **skills** that brief an AI coding agent on how to drive the Commercient Data Lake / Data Hub
CLI correctly — the right command *ordering*, the non-obvious gotchas, and the HTTP contract for the
Data API. A skill is a small folder your agent loads on demand, turning "figure `dlake` out by trial
and error" into "already knows the happy path."

## Available skills

| Skill | What it covers |
|-------|----------------|
| [`dlake/`](dlake/SKILL.md) | Build and operate a tenant end-to-end: schema → expose → restart → scoped key, the sharp edges (the scoped-key restart, exposed-vs-raw entities, IDENTITY-key limits), and the REST/GraphQL + events contract — all in one [`SKILL.md`](dlake/SKILL.md). |

## Install

A skill is just a folder. Copy it into the directory your harness scans for skills:

| Harness | Skills directory |
|---------|------------------|
| **Claude Code / Cowork** | `.claude/skills/dlake/` (per-project) or `~/.claude/skills/dlake/` (global) |
| **OpenAI Codex** | `~/.codex/skills/dlake/` or your project's skills directory |
| **OpenCode** | `.opencode/skill/dlake/` |
| **Other harnesses** | wherever the harness discovers skills — keep the `dlake/SKILL.md` layout intact |

```bash
# from a clone of this repo
cp -r skills/dlake ~/.claude/skills/       # e.g. install globally for Claude Code
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
