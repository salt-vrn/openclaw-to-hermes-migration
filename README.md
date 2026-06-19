# OpenClaw → Hermes Migration

🌐 [Русский](README.ru.md)

Step-by-step skill for migrating AI agents from OpenClaw to [Hermes Agent](https://hermes-agent.nousresearch.com/docs).

## What it does

A structured migration guide that an LLM agent follows to move a client from OpenClaw to Hermes:

- **Analyzes** the existing OpenClaw profile (files, scripts, cron, secrets)
- **Plans** the migration before doing anything
- **Creates** workspace structure using [hermes-advanced-memory](https://github.com/salt-vrn/hermes-advanced-memory)
- **Migrates** files, secrets (.env), scripts, cron jobs, config
- **Reports** what was done, what's pending, what needs manual work

## Why a skill, not a script?

Every agent's OpenClaw setup is different. Some have 5 files, some have 200. Some have custom folder names like "какашки". A script can't understand context — an LLM agent can.

The skill provides:
- **Rules** (what NEVER to do)
- **Phases** (what to do in what order)
- **Verification** (how to check each step)
- **Documentation links** (where to read the latest docs)

The agent adapts to each specific case.

## Installation

```bash
hermes skill install salt-vrn/openclaw-to-hermes-migration
```

Or manually:

```bash
cd ~/.hermes/skills/
git clone https://github.com/salt-vrn/openclaw-to-hermes-migration.git
```

## Prerequisites

Before migration, apply [hermes-advanced-memory](https://github.com/salt-vrn/hermes-advanced-memory) to organize the target workspace.

## The 10 phases

1. **Analyze** — inventory all files in OpenClaw profile
2. **Plan** — present migration plan, wait for approval
3. **Create structure** — build workspace with project folders
4. **Migrate files** — distribute (NOT copy) MEMORY.md, TOOLS.md
5. **Migrate secrets** — credentials.json → .env
6. **Migrate scripts** — update paths, replace credentials.json with os.getenv()
7. **Migrate cron jobs** — recreate in Hermes, test incrementally
8. **Configure Hermes** — groups, users, behavior
9. **Update AGENTS.md** — create master index
10. **Report** — summary of what's done and what's pending

## License

MIT

---

> A project by [NeiroHost.ru](https://neirohost.ru) — AI-agent hosting powered by Hermes and OpenClaw
