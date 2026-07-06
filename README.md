# ai-pilot-eu

Customer-facing POC skills for **Treasure AI Studio** (`agent.eu01.treasure.ai`).
Built for the AI POC framework — Data Ops skill build phase (Weeks 4–6).

## Skills

| Skill | JIRA | Description |
|---|---|---|
| `uc-data-lineage` | [ARCH-1335](https://treasure-data.atlassian.net/browse/ARCH-1335) | Full pipeline lineage from raw CDP landing to parent segment — interactive HTML dashboard |
| `uc-data-quality-monitor` | [ARCH-1325](https://treasure-data.atlassian.net/browse/ARCH-1325) | CDP data health monitoring — fill rate, row count trend, ID stitching coverage, dedup rate |
| `uc-parent-segment-overview` | [ARCH-1337](https://treasure-data.atlassian.net/browse/ARCH-1337) | Overview of all parent segments — health, activation summary, child segments, freshness dashboard |
| `skill-usage-tracker` | — | Skill usage tracking for Treasure Work (hooks) and TAS (inline Step 0 reference) |

## Adding to Marketplace

This repo is structured as a Claude Code marketplace plugin.
To load these skills in Treasure Work or Treasure AI Studio, add the following to your plugin config:

```json
{
  "source": "github:treasure-data/ai-pilot-eu",
  "plugin": "ai-pilot-eu"
}
```

Or clone and load locally:
```json
{
  "source": "./path/to/ai-pilot-eu",
  "plugin": "ai-pilot-eu"
}
```

## Skill Design Principles

All skills in this repo are built for **Treasure AI Studio** constraints:
- **No hooks** — usage tracking is inline Step 0, not PostToolUse hooks
- **No filesystem** — no writes to `/home/agent/.claude/hooks/`
- **No `mcp__work__render_react`** — dashboards are self-contained HTML files (Chart.js + D3.js CDN)
- **Explicit credential request** — `mcp__tas__request_credential` called before any bash block
- **`tdx query INSERT`** — confirmed working on eu01 for tracking writes

## Tracking

Every skill invocation logs to `ai_usage.skills_usage_tracker` on eu01:
```sql
SELECT skill_name, user_id, td_time_string(time, 's!', 'UTC') as invoked_at
FROM ai_usage.skills_usage_tracker
ORDER BY time DESC
LIMIT 20
```
