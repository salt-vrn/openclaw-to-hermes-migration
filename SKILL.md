---
name: openclaw-to-hermes-migration
description: "Migrate an agent from OpenClaw to Hermes. Use when: moving a client, reorganizing after migration, checking migration completeness."
version: 1.0.0
category: devops
triggers:
  - openclaw migration
  - migrate from openclaw
  - openclaw to hermes
  - agent migration
  - move agent
---

# OpenClaw → Hermes Migration

Step-by-step guide for migrating an agent from OpenClaw to Hermes.

> **Prerequisites:** Before starting, apply [hermes-advanced-memory](https://github.com/salt-vrn/hermes-advanced-memory) to organize the target workspace structure.

---

## ⚠️ Critical rules

1. **NEVER copy OpenClaw files as-is.** OpenClaw's MEMORY.md, TOOLS.md, IDENTITY.md, USER.md — these are OpenClaw concepts. In Hermes, data lives in AGENTS.md (index) and project README.md files.
2. **NEVER copy credentials.json into workspace.** Secrets go to `.env`. Read keys from credentials.json, write to `.env`, then verify.
3. **NEVER restart gateway yourself.** Ask the user to do it. Gateway restart from inside a session = potential deadlock.
4. **NEVER delegate to subagents.** Migration requires understanding context across files. One agent, one session.
5. **ALWAYS check paths after migration.** Every script, every cron job, every config reference must point to the new location.
6. **ALWAYS read the official docs.** Don't guess config format. Read: https://hermes-agent.nousresearch.com/docs/user-guide/configuration

---

## Phase 1: Analyze

**Goal:** understand what exists in the OpenClaw profile.

1. Find the OpenClaw profile directory (usually `/root/.openclaw/` or `/root/.openclaw/workspace/`)
2. List ALL files — `find /root/.openclaw -type f | sort`
3. Read key files:
   - `SOUL.md` / `IDENTITY.md` — agent personality
   - `MEMORY.md` — business knowledge (this will be DISTRIBUTED, not copied)
   - `TOOLS.md` — tool usage notes (this goes into project README.md files)
   - `USER.md` — user profile
   - `config.yaml` / `openclaw.json` — configuration
   - `credentials.json` — secrets (read, don't copy)
4. Check cron jobs: `cat /root/.openclaw/cron/jobs.json`
5. Check for scripts, skills, docs, media

**Output:** a complete inventory with categories:
- 🔴 Critical (must migrate)
- 🟡 Important (should migrate)
- 🟢 Nice-to-have (optional)
- ⚪ Skip (archives, temp files, __pycache__)

---

## Phase 2: Plan

**Goal:** present a migration plan to the user BEFORE doing anything.

Present the plan including:
1. What files go where (mapping old → new)
2. How MEMORY.md content will be distributed:
   - Short facts → AGENTS.md
   - Project-specific data → project-folder/README.md
3. How TOOLS.md content will be distributed:
   - Project-specific tools → project-folder/README.md
   - General notes → AGENTS.md
4. How credentials.json will be handled:
   - Read all keys
   - Write to `.env` with proper naming
   - Verify each API connection
5. How cron jobs will be recreated
6. What will NOT be migrated and why

**Wait for user approval before proceeding.**

---

## Phase 3: Create structure

**Goal:** build the Hermes workspace structure.

1. Determine the agent's profile. If unsure — **ask the user**. Do not guess.
2. Create workspace folder in the profile root
3. Create project folders based on the plan
4. Create `AGENTS.md` — empty index with header
5. For each project folder, create `README.md` with a placeholder

**Verify:** `ls -la` the new structure matches the plan.

---

## Phase 4: Migrate files

**Goal:** move files into the new structure.

For each file/group:

1. Copy files to their target locations
2. **DO NOT copy OpenClaw-specific files as-is:**
   - `MEMORY.md` → distribute content to AGENTS.md and project README.md files
   - `TOOLS.md` → distribute to project README.md files
   - `IDENTITY.md` → this is SOUL.md in Hermes (merge, don't duplicate)
   - `USER.md` → user profile info goes to Hermes config or AGENTS.md
   - `BOOT.md`, `HEARTBEAT.md` → skip (OpenClaw-specific)
3. Copy project files, scripts, docs, media to their folders
4. **docx/pdf files** — don't just dump them in a `docs/` folder. For each:
   - Read content: `python3 -c "from docx import Document; print('\n'.join(p.text for p in Document('file.docx').paragraphs))"` or ask the user
   - Determine which project it belongs to
   - Move to that project folder
   - Add a reference in the project's README.md: `## Documents: filename.docx — [brief description]`
   - If unsure — ask the user, don't guess

**Verify:** no orphan files in generic `docs/` folders. Every document is referenced in a project README.

**Verify:** count files in old vs new. No files should be lost.

---

## Phase 5: Migrate secrets

**Goal:** move credentials from JSON to .env.

1. Read `credentials.json` — parse all keys
2. Map each key to an env variable name:
   - `amoCRM.api_key` → `AMOCRM_API_KEY`
   - `megaplan.token` → `MEGAPLAN_TOKEN`
   - `1c.password` → `_1C_PASSWORD`
   - etc.
3. Write to `.env` in the profile root
4. Verify each connection:
   ```bash
   # Example: verify amoCRM
   curl -s "https://domain.amocrm.ru/api/v4/account" \
     -H "Authorization: Bearer $AMOCRM_ACCESS_TOKEN"
   ```
5. If any token is expired — note it in the report, don't try to refresh
6. **Token files** (avito_token.json, service_account.json, etc.) — also go to `.env`. Read the token value, write to `.env`, delete the file from workspace. NEVER leave token files in workspace.

**Verify:** `grep -c "credentials.json"` in workspace should be 0. `grep -c "token.json"` in workspace should be 0.

---

## Phase 6: Migrate scripts

**Goal:** update all script references to use new paths and .env.

1. Find all scripts that reference old paths:
   ```bash
   grep -rl "/root/.openclaw" /path/to/new/workspace/
   grep -rl "credentials.json" /path/to/new/workspace/
   ```
2. For each script, update:
   - `json.load(open("credentials.json"))` → `os.getenv("KEY_NAME")`
   - `/root/.openclaw/workspace/` → new workspace path
   - Any hardcoded paths
3. Test critical scripts (amoCRM, Megaplan, 1C)

**Verify:** `grep -c "/root/.openclaw"` in new workspace should be 0.

---

## Phase 7: Migrate cron jobs

**Goal:** recreate all cron jobs in Hermes.

Documentation: https://hermes-agent.nousresearch.com/docs/user-guide/configuration

1. Read OpenClaw cron: `cat /root/.openclaw/cron/jobs.json`
2. For each job, check:
   - **Schedule** — same cron expression works in Hermes?
   - **Prompt** — does it reference old paths? Update.
   - **Delivery** — group chat IDs, user IDs — verify they exist in Hermes config
   - **Scripts** — if the job calls a script, verify the script path and internal paths
   - **API keys** — if the job uses APIs, verify keys are in .env
3. Create each job in Hermes using `cronjob create`
4. **DO NOT** create all jobs at once. Create 2-3, test, then continue.
5. After creating each job — run it manually to verify it works: `cronjob run <job_id>`
6. Check that the job's script exists, paths are correct, API keys are accessible

**Verify:** list all jobs, compare count with OpenClaw.

---

## Phase 8: Configure Hermes

**Goal:** set up the Hermes config (groups, users, behavior).

Documentation: https://hermes-agent.nousresearch.com/docs/user-guide/configuration

1. Read the old OpenClaw config for:
   - Allowed chat IDs
   - Allowed user IDs
   - Group behavior (respond to all? only @mentions?)
2. Update Hermes `config.yaml`:
   - `telegram.allowed_chats` — all group IDs
   - `telegram.allowed_users` — all user IDs
   - `telegram.free_response_chats` — groups where agent responds to everyone
   - `telegram.require_mention` — groups where agent responds only to @mentions
3. Ask the user to restart the gateway (do NOT do it yourself)

---

## Phase 9: Update AGENTS.md

**Goal:** create the master index.

1. List all projects with brief descriptions (2-3 sentences each)
2. Link each project to its README.md
3. Include:
   - Team/user info (from old MEMORY.md/USER.md)
   - Key rules and conventions
   - Cron job table (name, schedule, delivery)
   - API connection status (working/expired/unknown)
4. Keep it concise — AGENTS.md loads every session, it must be readable

---

## Phase 10: Report

**Goal:** tell the user what was done, what's pending, what needs manual work.

Report format:

```
Migration complete: OpenClaw → Hermes

✅ Done:
- [N] files migrated
- [N] scripts updated (credentials.json → .env)
- [N] cron jobs created
- [N] groups configured
- AGENTS.md index created

⚠️ Needs attention:
- [expired token / broken script / missing config]

❌ Not migrated (with reason):
- [backup archives / temp files / OpenClaw-specific]

📁 Structure:
[tree of new workspace]
```

**Ask the user to restart the gateway** if config was changed.

---

## Common mistakes to avoid

| Mistake | Why it's wrong | What to do instead |
|---------|---------------|-------------------|
| Copy MEMORY.md as-is | OpenClaw concept, doesn't exist in Hermes | Distribute to AGENTS.md + project README.md |
| Copy TOOLS.md as-is | OpenClaw concept | Distribute to project README.md files |
| Copy credentials.json to workspace | Secrets in workspace = security risk | Parse keys → write to .env |
| Copy token files (avito_token.json, etc.) to workspace | Same as credentials — secrets belong in .env | Read token → write to .env → delete file |
| Restart gateway yourself | Deadlock — gateway can't restart from inside session | Ask user to restart |
| Create all cron jobs at once | If one is wrong, all waste tokens | Create 2-3, test, continue |
| Guess config format | Config format changes between versions | Read official docs |
| Delegate to subagents | Subagents lose context between files | One agent, one session |
| Skip path verification | Scripts will fail silently | grep for old paths after migration |

---

## Documentation links

- **Hermes configuration:** https://hermes-agent.nousresearch.com/docs/user-guide/configuration
- **Hermes advanced memory (workspace rules):** https://github.com/salt-vrn/hermes-advanced-memory
- **Cron jobs:** https://hermes-agent.nousresearch.com/docs/user-guide/configuration (cron section)
- **Skills:** https://hermes-agent.nousresearch.com/docs/skills
