---
name: skill-usage-tracker
description: |
  Sets up automated skill usage tracking for this session. Invoke this at the
  start of a new chat to log every skill call to ai_usage.skills_usage_tracker
  in Treasure Data (eu01). Creates the hook script and wires it into the
  session's settings.json automatically.
  TRIGGER: When user asks to "enable skill tracking", "set up skill tracker",
  or at the start of a session when this skill is available.
---

## ⚠️ Environment Compatibility

| Environment | Tracking Method | Works? |
|---|---|---|
| **Treasure Work** (desktop app) | PostToolUse hook — auto-fires on every Skill call | ✅ Full automation |
| **Treasure AI Studio** (`agent.eu01.treasure.ai`) | Hooks not available — filesystem write blocked | ❌ Steps 3–4 will fail |

**If you are in Treasure AI Studio:** Do NOT use this skill. Instead, every individual skill
must include an inline Step 0 tracking block. See the canonical pattern in
`skills/uc-data-lineage/SKILL.md` Step 0 — copy that block into each new TAS skill,
changing only the `skill_name` value.

The inline Step 0 pattern requires credentials to be requested explicitly BEFORE the bash
block runs (TAS does not pre-load credentials):
1. Call `mcp__tas__request_credential` with `td_api_production_eu01` first
2. Then run the bash tracking block

---

## Treasure Work — Steps (hook-based, fully automated)

When this skill is invoked in Treasure Work, perform the following steps in order:

### Step 1 — Resolve user identity

First request credentials explicitly, then resolve identity:

```
Request credential: td_api_production_eu01
```

Then run:

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
tdx status
```

Extract `User:` and `Account ID:` from the output. Use these for USER_ID and ACCOUNT_ID
in the hook script below.

### Step 2 — Ensure ai_usage database and table exist

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01

tdx api -X POST /v3/database/create/ai_usage 2>/dev/null || true
tdx api -X POST /v3/table/create/ai_usage/skills_usage_tracker/log 2>/dev/null || true
tdx api -X POST \
  --data '{"schema":"[[\"id\",\"string\"],[\"user_id\",\"string\"],[\"account_id\",\"string\"],[\"skill_name\",\"string\"]]"}' \
  /v3/table/update-schema/ai_usage/skills_usage_tracker 2>/dev/null || true
```

### Step 3 — Write the hook script

> **Treasure Work only** — this step writes to `/home/agent/.claude/hooks/` which is
> not available in Treasure AI Studio.

```bash
mkdir -p /home/agent/.claude/hooks
```

Write the file at `/home/agent/.claude/hooks/skill_usage_tracker.sh`
(replace `<USER_ID>` and `<ACCOUNT_ID>` with values from Step 1):

```bash
#!/bin/bash
set -u

INPUT=$(cat)

PARSED=$(python3 -c '
import json, sys, uuid
try:
    d = json.loads(sys.stdin.read())
    tool = d.get("tool_name", "")
    skill = (d.get("tool_input") or {}).get("skill", "")
    if tool == "Skill" and skill:
        print(f"{uuid.uuid4()}\t{skill}")
except Exception:
    pass
' <<<"$INPUT")

[ -n "$PARSED" ] || exit 0

UUID="${PARSED%%	*}"
SKILL_NAME="${PARSED##*	}"

TOKEN=$(curl -sf --max-time 5 http://172.30.0.1:18080/credentials/td_api_production_eu01 2>/dev/null)
[ -n "$TOKEN" ] || exit 0

SKILL_ESC=$(printf '%s' "$SKILL_NAME" | sed "s/'/''/g")

export TDX_ACCESS_TOKEN="$TOKEN"
export TDX_SITE="eu01"

tdx query --database ai_usage \
  "INSERT INTO skills_usage_tracker (id, user_id, account_id, skill_name) VALUES ('$UUID', '<USER_ID>', '<ACCOUNT_ID>', '$SKILL_ESC')" \
  >/dev/null 2>&1 || true

exit 0
```

Then make it executable:

```bash
chmod +x /home/agent/.claude/hooks/skill_usage_tracker.sh
```

### Step 4 — Wire the hook into session settings

> **Treasure Work only** — `settings.json` hook wiring is not available in TAS.

Create `.claude/settings.json` in the current working directory:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Skill",
        "hooks": [
          {
            "type": "command",
            "command": "/home/agent/.claude/hooks/skill_usage_tracker.sh",
            "async": true,
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

### Step 5 — Confirm

Tell the user:
- Which user identity was detected
- That the hook is set up for this session
- That skill calls from this point forward will be logged to `ai_usage.skills_usage_tracker` on eu01
- To open `/hooks` once to ensure the session picks up the new settings

---

## Tracking Table Reference

**Database:** `ai_usage`
**Table:** `skills_usage_tracker`
**Schema:** `id (varchar)`, `user_id (varchar)`, `account_id (varchar)`, `skill_name (varchar)`, `time (bigint, auto)`

Verify tracking is working:
```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
tdx query --database ai_usage \
  "SELECT id, user_id, account_id, skill_name, td_time_string(time, 's!', 'UTC') as invoked_at FROM skills_usage_tracker ORDER BY time DESC LIMIT 10"
```
