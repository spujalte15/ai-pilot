# AI POC EU — Project Context & Session Memory

> This file is the persistent reference for all skill-building work in this workspace.
> Update it after every significant session. Always read this before starting new work.

---

## Project Goal

Build customer-facing skills for **Treasure AI Studio** (web agent at `agent.eu01.treasure.ai`)
mapped one-to-one with JIRA tickets in the `ARCH` project. Each ticket → one `SKILL.md` file.

**Repo target:** `treasure-data/ai-poc-framework — skills/<skill-name>/SKILL.md`
**Local working path:** `/Users/biswadip.paul/Documents/TreasureData/ai-pilot-eu/skills/`

---

## Treasure AI Studio — Constraints vs Treasure Work

| Capability | Treasure Work (desktop) | Treasure AI Studio (web) |
|---|---|---|
| Filesystem writes | ✅ Full | ⚠️ Limited (workflow pull, ps desc -o work; /home/agent hooks do NOT) |
| Pre/Post hooks | ✅ settings.json | ❌ NOT available |
| `skill-usage-tracker` auto-hook | ✅ | ❌ Must be inline Step 0 in every skill |
| Marketplace skills | ✅ All | ✅ Same marketplace |
| Context / memory | ✅ Long | ⚠️ Shorter |
| `mcp__tas__open_file` | ✅ | ❌ Use `mcp__work__render_react` instead |
| `information_schema.columns` | N/A | ❌ NOT available in TD Trino — use `tdx describe` |
| `tdx query INSERT INTO` | Convention per skill-usage-tracker | ❌ DOES NOT WORK — TD Trino is read-only. Use `/v3/table/import_with_id` REST endpoint instead |

---

## Skill Usage Tracking — Canonical Pattern for Treasure AI Studio

Every skill built for TAS must include this as **Step 0** (inline, not hooked):

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01

TD_STATUS=$(tdx status 2>/dev/null)
USER_ID=$(echo "$TD_STATUS" | grep "^User:" | awk '{print $2}')
ACCOUNT_ID=$(echo "$TD_STATUS" | grep "^Account ID:" | awk '{print $3}')
RECORD_ID=$(python3 -c "import uuid; print(uuid.uuid4())")

tdx api -X POST /v3/database/create/ai_usage 2>/dev/null || true
tdx api -X POST /v3/table/create/ai_usage/skills_usage_tracker/log 2>/dev/null || true
tdx api -X POST \
  --data '{"schema":"[[\"id\",\"string\"],[\"user_id\",\"string\"],[\"account_id\",\"string\"],[\"skill_name\",\"string\"]]"}' \
  /v3/table/update-schema/ai_usage/skills_usage_tracker 2>/dev/null || true

tdx api -X POST \
  --data "{\"columns\":[{\"name\":\"id\",\"value\":\"$RECORD_ID\"},{\"name\":\"user_id\",\"value\":\"$USER_ID\"},{\"name\":\"account_id\",\"value\":\"$ACCOUNT_ID\"},{\"name\":\"skill_name\",\"value\":\"<SKILL_NAME>\"}]}" \
  /v3/table/import_with_id/ai_usage/skills_usage_tracker/json 2>/dev/null || true
```

**Schema confirmed live on eu01:**
- Database: `ai_usage`
- Table: `skills_usage_tracker`
- Columns: `id (varchar)`, `user_id (varchar)`, `account_id (varchar)`, `skill_name (varchar)`, `time (bigint)` (auto)

**Pending clarification (Sébastien Pujalte, ARCH-1335 reporter):**
The JIRA ticket says `{customer_slug}_ai_poc_tracking.skill_usage`. It is unclear whether
this is a separate customer-scoped table in addition to `ai_usage.skills_usage_tracker`,
or a replacement for it. Until clarified, skills use `ai_usage.skills_usage_tracker`.

---

## Verified CLI Commands (Tested on eu01, 2026-07-02)

```bash
# Auth
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01

# User/account identity extraction
tdx status | grep "^User:"         # → "User: email@example.com"
tdx status | grep "^Account ID:"   # → "Account ID: 61"

# Current eu01 account
User: biswadip.paul+se-emea@treasure-data.com
Account ID: 61

# List parent segments (140 found on this account)
tdx ps list

# Describe parent segment — CRITICAL: -o flag MUST come before segment name
tdx ps desc -o ps_schema.json "<SEGMENT_NAME>"   # ✅ works
tdx ps desc "<SEGMENT_NAME>" -o ps_schema.json   # ❌ does NOT write file

# List tables in a database
tdx tables "<db_name>.*" --json                  # ✅ correct form
tdx tables <db_name> --json                      # ❌ ambiguous / may not work

# Schema inspection (NOT information_schema.columns which doesn't work in TD)
tdx describe "<db>.<table>" --json               # ✅

# Workflow commands
tdx wf projects                                  # list all projects (632 on this account)
tdx wf pull <project_name> --yes                 # ✅ correct flag
tdx wf pull <project_name> -y                    # ❌ -y not supported

# tdx wf list — does NOT exist
tdx wf list                                      # ❌ "unknown command 'list'"
# Use: tdx wf projects  OR  tdx wf sessions  OR  tdx wf attempts
```

---

## Skills Built

### uc-data-lineage (ARCH-1335)
- **File:** `skills/uc-data-lineage/SKILL.md`
- **Status:** v2 — reviewed, all 16 issues from expert review fixed
- **Team:** Data Ops
- **What it does:** Full pipeline lineage Raw → Staging → Golden → Unification → Parent Segment, React dashboard
- **Marketplace gaps documented:**
  - `id-unification-lineage` — NOT in marketplace; Step 5d replicates its patterns inline
  - `uc-data-quality-monitor` — NOT in marketplace; Step 6 queries its expected output table conditionally
- **Key design decisions:**
  - Usage tracking: inline Step 0, writes to `ai_usage.skills_usage_tracker`
  - `tdx ps desc -o` flag must precede segment name
  - `information_schema.columns` removed — replaced with `tdx describe`
  - Column-click interactivity: clicking column pill in Pipeline Overview navigates to Explorer tab with that column pre-selected
  - Each layer node shows key columns (ticket requirement)
  - `isDark` used as pre-loaded global (available in render_react sandbox)
  - `--yes` not `-y` for `tdx wf pull`

---

## Pending Work

- [ ] Await Sébastien Pujalte's response on `{customer_slug}_ai_poc_tracking.skill_usage` vs `ai_usage.skills_usage_tracker`
- [ ] Other ARCH tickets to be assigned — fetch from Jira board when available
- [ ] `id-unification-lineage` local skill — if/when added to marketplace, update Step 5d in uc-data-lineage
- [ ] `uc-data-quality-monitor` — companion skill, not yet built

---

## How to Resume Work in a New Session

1. Read this file first: `/Users/biswadip.paul/Documents/TreasureData/ai-pilot-eu/CONTEXT.md`
2. Read `skill-usage-tracker.md` for the canonical tracking pattern
3. Fetch the JIRA ticket via `mcp__atlassian__getJiraIssue` (cloudId: `treasure-data.atlassian.net`)
4. Check marketplace skill availability before writing — flag gaps explicitly
5. Test CLI commands on eu01 before putting them in skill steps
6. Run expert review agent (opus model) before finalising
7. Update this file after completing each skill

---

## Atlassian Authentication

Jira access requires browser OAuth via:
`mcp__atlassian__authenticate` → open URL in browser → confirm

Project: `ARCH` (Solution Architecture)
Cloud ID: `treasure-data.atlassian.net`
