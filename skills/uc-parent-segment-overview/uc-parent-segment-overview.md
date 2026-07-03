---
name: uc-parent-segment-overview
description: |
  Customer-facing dashboard for Marketing Ops and Data Ops pilot users (ARCH-1337).
  Provides a single-screen overview of all parent segments in the account:
  health status, activation summary, child segments, data freshness, schedule
  compliance, and anomaly flags. Answers "what is actually running in our CDP right now?"
  Triggers on: "parent segment overview", "what segments do we have", "CDP overview",
  "segment dashboard", "show all parent segments", "what is running", "segment health",
  "uc-parent-segment-overview", "activation summary", "CDP inventory".
  Team: Marketing Ops + Data Ops. JIRA: ARCH-1337. POC Framework Phase: Weeks 4–6.
---

# uc-parent-segment-overview — All Parent Segments Dashboard

Single-screen overview of every parent segment in the account: health status,
activation summary, child segments, data freshness, schedule compliance, and anomalies.
Generates a self-contained HTML file.

**Adapts from:** `tdx-skills:parent-segment-analysis` (single-segment deep analysis).
This skill adds multi-segment overview framing. Do NOT modify `parent-segment-analysis`.

---

## ⚠️ Marketplace Skill Availability Notice

| Skill | Status | Usage in this skill |
|---|---|---|
| `tdx-skills:parent-segment-analysis` | ✅ Available | Single-segment drill-down |
| `tdx-skills:parent-segment` | ✅ Available | YAML/schedule patterns |
| `tdx-skills:activation` | ✅ Available | Step 2b: `tdx sg pull --yes` to get batch activation config from YAML |
| `tdx-skills:connector-config` | ✅ Available | Step 2b: `tdx connection list` to verify connections |
| `realtime-skills:activations` | ✅ Available | Step 2c: query `cdp_audience_<id>_rt.activations` for RT run status |
| `realtime-skills:rt-journey-monitor` | ✅ Available | Step 2c: RT activation failure detection patterns |
| `tdx-skills:tdx-basic` | ✅ Available | Used throughout |
| `sql-skills:trino` | ✅ Available | Freshness and activation queries |
| `treasure-work-skills:react-dashboard` | ❌ NOT in TAS | Step 4 generates self-contained HTML (Chart.js CDN) |
| `uc-rfm-segmentation` | ❌ NOT in repo or marketplace | Drill-down uses `tdx-skills:parent-segment-analysis` instead |
| `uc-data-quality-monitor` | ✅ Available (this repo) | Step 3 queries `ai_usage.data_quality_flags` if present |

> **Tracking table:** Ticket specifies `{customer_slug}_ai_poc_tracking.skill_usage`.
> This skill standardises to `ai_usage.skills_usage_tracker` per ARCH-1335 platform pattern.
> Reason: `customer_slug` is not reliably available as a variable in Treasure AI Studio at
> runtime, making per-customer-slug table creation impractical in TAS.

> **CDN note:** Generated HTML uses Chart.js CDN. Open in external browser — Treasure Work's
> built-in file viewer is sandboxed and will show blank charts.

> **Activation data availability — two tiers:**
>
> | Activation Type | Config Available | Run Status Available | How |
> |---|---|---|---|
> | Batch (segment/journey) | ✅ via `tdx sg pull --yes` YAML | ❌ No run history in CLI | YAML shows connection, schedule, columns |
> | RT 2.0 (journeys) | ✅ via RT journey config | ✅ `cdp_audience_<id>_rt.activations` table | `delivered`, `status`, `error`, `activation_type`, `last run time` |

> **Remaining gaps (not fixable via CLI):**
>
> | Data Point | Status | Note |
> |---|---|---|
> | Batch activation run history | ❌ Not available | Show config only from YAML |
> | Consecutive failures (batch) | ❌ Not available | Only detectable for RT via SQL |
> | Child segment author (createdBy) | ❌ Not in tdx sg list | Show time + size only |
> | Last PS workflow run status | ❌ Not in tdx ps | Use `tdx wf sessions` if workflow name known |

---

## Step 0 — Log Invocation

**THIS STEP IS MANDATORY. Execute it completely and confirm before moving to Step 1. Do NOT merge with any other step.**

### 0a — Request TD credentials (separate tool call)

Call `mcp__tas__request_credential` with `td_api_production_eu01`.
Wait for green "Authenticated" confirmation. **Do not run any bash until this succeeds.**

### 0b — Log invocation (separate bash call — do NOT combine with Step 1 or any other step)

> **Treasure AI Studio note:** skill-usage-tracker hooks do NOT work in TAS. Run the inline
> bash below directly. Do NOT call skill-usage-tracker as a Skill in TAS.

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01

TD_STATUS=$(tdx status 2>/dev/null)
USER_ID=$(echo "$TD_STATUS" | grep "^User:" | awk '{print $2}')
ACCOUNT_ID=$(echo "$TD_STATUS" | grep "^Account ID:" | awk '{print $3}')
RECORD_ID=$(python3 -c "import uuid; print(uuid.uuid4())" 2>/dev/null || echo "$(date +%s)-$$")

tdx api -X POST /v3/database/create/ai_usage 2>/dev/null || true
tdx api -X POST /v3/table/create/ai_usage/skills_usage_tracker/log 2>/dev/null || true
tdx api -X POST \
  --data '{"schema":"[[\"id\",\"string\"],[\"user_id\",\"string\"],[\"account_id\",\"string\"],[\"skill_name\",\"string\"]]"}' \
  /v3/table/update-schema/ai_usage/skills_usage_tracker 2>/dev/null || true

tdx query --database ai_usage \
  "INSERT INTO skills_usage_tracker (id, user_id, account_id, skill_name) VALUES ('$RECORD_ID', '$USER_ID', '$ACCOUNT_ID', 'uc-parent-segment-overview')" \
  2>&1

echo "RECORD_ID=$RECORD_ID"
echo "✅ Step 0 complete — uc-parent-segment-overview invocation logged for $USER_ID (record: $RECORD_ID)"
```

### 0c — Confirm before continuing

Output as your next assistant message:
```
✅ Step 0 complete — uc-parent-segment-overview invocation logged for <USER_ID> (record: <RECORD_ID>)
```
**Do NOT proceed to Step 1 until you have output that confirmation line.**

---

## Step 1 — Discover All Parent Segments

**Performance warning:** This account has 140 parent segments. Steps 1–2 run one CLI call per segment for child segment enrichment. For all 140 segments this will take 15–30 minutes. Ask the user first:

> "There are 140 parent segments. Would you like to see all of them, or filter by name pattern (e.g. `Automotive*`, `ANWB*`)? For best performance, start with a pattern filter — you can run again with 'all' for a full inventory."

If user says "all" — proceed and warn: "This will take approximately 15–30 minutes. I'll run all 140 segments."
If user provides a pattern — use: `tdx ps list "<pattern>" --json` (supports `*` and `?`)

**Step 1a — List segments:**

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
tdx ps list --json 2>&1
```

From the JSON, for each segment extract:
- `id`, `attributes.name`, `attributes.population`
- `attributes.scheduleType` — `none` | `daily` | `hourly` | `cron`
- `attributes.matrixUpdatedAt` — last data refresh (use for freshness)
- `attributes.updatedAt`, `attributes.createdAt`
- `url` — console link

**Step 1b — Get output database name (required by ticket):**

For each segment, call `tdx ps view` to get the `master` field:

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
tdx ps view "<SEGMENT_NAME>" --json 2>&1
```

Extract `master.parentDatabaseName` → this is the output database name.

> **Note:** `last run status` (success/fail) is NOT available in `tdx ps list` or `tdx ps view`.
> To get workflow run history for a segment, use `tdx wf sessions <workflow_project>` if the
> associated workflow project name is known. For this overview skill, data freshness from
> `matrixUpdatedAt` is used as a proxy for health — not workflow run status.

Store enriched list as `ALL_SEGMENTS`.

---

## Step 2 — Enrich Each Segment

For each segment in `ALL_SEGMENTS` run sub-steps 2a, 2b, 2c.

### 2a — Child Segments (`tdx-skills:segment` patterns)

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
tdx ps use "<SEGMENT_NAME>" 2>/dev/null
tdx sg list --json 2>&1
```

Extract only `type: "segment"` entries: `id`, `name`, `population`, `createdAt`, `updatedAt`.

> `createdBy` is not in `tdx sg list` output — show time + size only in Activity Feed.

> After running `tdx ps use` N times, the session context will be set to the last segment.
> Run `tdx ps use ""` or start a new session after this skill completes if needed.

### 2b — Activation Config (`tdx-skills:activation` patterns)

Pull the segment YAML to get configured activations:

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
tdx ps use "<SEGMENT_NAME>" 2>/dev/null
tdx sg pull --yes 2>&1
```

The pulled YAML files in `segments/<ps_name>/` contain `activations:` blocks with:
- `name` — activation name
- `connection` — connector name
- `schedule.type` — none | daily | hourly | cron
- `connector_config` — connector-specific settings

Parse the activation YAML for each segment now, then clean up:

```bash
# Clean up pulled YAML files — not needed after activation data is extracted
rm -rf segments/
```

> This prevents thousands of YAML files from accumulating in the CWD across multiple segment runs.

Also run to confirm connections exist:
```bash
tdx connection list 2>&1
```

For each activation in the segment YAML, look up `activations[].connection` (the connection name) in the `tdx connection list` output. The `type` column in that output is the connector type (e.g. `sfmc_out`, `google_ads_out`, `salesforce_crm_out`). Set `activation.type` = that connector type. If the connection name is not found in the list, set `type = "unknown"`.

This gives you **batch activation config** (connector type inferred from connection name + `tdx connection list` output).

> Batch activation **run history** (last run time, success/fail) is NOT available via `tdx sg` —
> only config is in the YAML. Show config only for batch activations.

### 2c — RT Activation Status (`realtime-skills:activations` patterns)

> **`<SEGMENT_ID>` = parent segment ID from `tdx ps list` (Step 1) — not a child segment ID.**

Check if an RT database exists for this segment:

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
tdx tables "cdp_audience_<SEGMENT_ID>_rt.*" --json 2>&1 | grep "activations"
```

If `cdp_audience_<SEGMENT_ID>_rt.activations` exists, query it for run status:

```bash
tdx query --database cdp_audience_<SEGMENT_ID>_rt \
  "SELECT
     activation_name,
     activation_type,
     COUNT(*) as total_attempts,
     SUM(CASE WHEN delivered='true' THEN 1 ELSE 0 END) as successes,
     SUM(CASE WHEN delivered='false' THEN 1 ELSE 0 END) as failures,
     td_time_string(MAX(time),'s!','UTC') as last_attempt,
     MAX(CASE WHEN delivered='false' THEN error ELSE NULL END) as last_error
   FROM activations
   WHERE td_interval(time, '-7d')
   GROUP BY activation_name, activation_type
   ORDER BY last_attempt DESC" 2>&1
```

For consecutive failures, use `realtime-skills:rt-journey-monitor` pattern:
```bash
tdx query --database cdp_audience_<SEGMENT_ID>_rt \
  "SELECT activation_name, COUNT(*) as consecutive_failures
   FROM (
     SELECT activation_name,
       SUM(CASE WHEN delivered='true' THEN 1 ELSE 0 END)
         OVER (PARTITION BY activation_name ORDER BY time DESC) as cumulative_success
     FROM activations
     WHERE td_interval(time, '-30d')
   ) t
   WHERE t.cumulative_success = 0
   GROUP BY activation_name" 2>&1
```

Store per segment: `activations: [{name, type, isRT, schedule, consecutiveFailures, lastAttempt, lastStatus, lastError}]`

> **Derive `lastStatus`:** If `consecutiveFailures = 0` AND `successes > 0`, set `lastStatus = "true"`. If `consecutiveFailures > 0`, set `lastStatus = "false"`. Otherwise `null`.

**Freshness calculation** (compute per segment):

```python
from datetime import datetime, timezone

matrix_updated = "<matrixUpdatedAt>"
schedule_type  = "<scheduleType>"
now            = datetime.now(timezone.utc)
last_updated   = datetime.fromisoformat(matrix_updated.replace("Z", "+00:00"))
hours_since    = (now - last_updated).total_seconds() / 3600

# Thresholds by schedule type
thresholds = {
    "hourly": {"amber": 2,   "red": 4},
    "daily":  {"amber": 26,  "red": 50},
    "cron":   {"amber": 26,  "red": 50},
    "none":   {"amber": 168, "red": 720},
}
t = thresholds.get(schedule_type, thresholds["none"])

if hours_since <= t["amber"]:
    health = "GREEN"
elif hours_since <= t["red"]:
    health = "AMBER"
else:
    health = "RED"
```

> **scheduleType "none" note:** Segments with `scheduleType: "none"` are manually managed.
> They will show AMBER after 7 days and RED after 30 days. Many intentionally-static segments
> may legitimately not refresh for months. In the dashboard, RED status on `none`-schedule
> segments carries a note: "Manual segment — verify if refresh is expected before treating as issue."

**Quality flags** (if `uc-data-quality-monitor` has run):

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
tdx tables "ai_usage.*" --json 2>&1 | grep "data_quality_flags"
```

If found:
```bash
tdx query --database ai_usage \
  "SELECT table_name, issue_type, severity FROM data_quality_flags
   WHERE run_id = (SELECT MAX(run_id) FROM data_quality_flags)
     AND severity IN ('HIGH','CRITICAL')
     AND table_name LIKE 'cdp_audience_<SEGMENT_ID>%'" 2>&1
```

---

## Step 3 — Surface Anomalies

Build `ANOMALIES` list automatically — no user prompting needed:

| Anomaly Type | Condition | Severity | Note |
|---|---|---|---|
| `not_refreshed` | health = RED | 🔴 RED | — |
| `delayed` | health = AMBER | 🟡 AMBER | — |
| `zero_population` | population = 0 AND scheduleType ≠ "none" | 🔴 RED | — |
| `no_child_segments` | childSegmentCount = 0 AND scheduleType ≠ "none" | 🟡 AMBER | — |
| `quality_flags` | Any HIGH/CRITICAL in data_quality_flags | 🔴 RED | — |
| `manual_segment_stale` | scheduleType = "none" AND health = RED | 🟡 AMBER | Note: may be intentional |
| `rt_activation_failures` | consecutiveFailures > 0 on RT activation | 🔴 RED | RT segments only |
| `rt_activation_all_failing` | All attempts in last 7d failed | 🔴 RED | RT segments only |
| `batch_activation_no_schedule` | Batch activation schedule.type = 'none' AND run_after_journey_refresh ≠ true AND parent scheduleType ≠ 'none' | 🟡 AMBER | Config check via tdx sg pull. Activations with `run_after_journey_refresh: true` have `schedule.type: none` by design — do not flag these. |
| `child_size_outlier` | N/A — no historical baseline | ⚪ SKIP | Requires multiple runs |

For each non-SKIP anomaly record: `{segmentName, segmentId, anomalyType, detail, severity}`

---

## Step 4 — Generate the HTML Dashboard

Build the complete `overviewData` object, then write the HTML file.
Output filename: `ps_overview_<YYYYMMDD>.html`

**Before running the Python block, build the `activations` array for each segment by merging Steps 2b and 2c:**
1. From Step 2b YAML: for each entry in `activations:` block, create `{isRT: False, name, connection, type (from connection list lookup), scheduleType (from schedule.type), lastAttempt: None, lastStatus: None, consecutiveFailures: 0, lastError: None}`
2. From Step 2c RT query: if `_rt.activations` table existed, merge RT data by `activation_name`: set `isRT: True`, update `lastAttempt`, `lastStatus`, `consecutiveFailures`, `lastError`. If an RT activation has no YAML match, add it as a new item with `isRT: True`.
3. If neither step found activations, set `activations: []`.

```bash
python3 << 'PYEOF'
import json, datetime, re

RUN_DATE = datetime.datetime.utcnow().strftime("%Y-%m-%d %H:%M UTC")

# Populate from real data in Steps 1–3
overview_data = {
  "runDate":       RUN_DATE,
  "accountId":     "<ACCOUNT_ID>",   # substitute from Step 0
  "totalSegments": 0,
  "summary": {
    "green": 0, "amber": 0, "red": 0,
    "totalChildSegments": 0,
    "totalPopulation": 0,
    "anomalyCount": 0,
  },
  "segments": [
    # One entry per parent segment:
    # {
    #   "id": "1087307",
    #   "name": "Automotive Demo",
    #   "database": "gld_stellantis",        # from tdx ps view master.parentDatabaseName
    #   "population": 253941,
    #   "scheduleType": "none",
    #   "matrixUpdatedAt": "2026-01-27T11:26:47.448Z",
    #   "hoursSinceRefresh": 1234.5,
    #   "health": "AMBER",
    #   "childSegments": [{"id":"...","name":"...","population":0,"createdAt":"...","updatedAt":"..."}],
    #   "childSegmentCount": 0,  # auto-derived from len(childSegments) — do not set manually
    #   "activations": [
    #     {
    #       "name": "export sfmc test",       # from YAML activations[].name
    #       "connection": "spujalte_sfmc",    # from YAML activations[].connection
    #       "type": "sfmc_out",               # inferred from tdx connection list type
    #       "scheduleType": "none",           # from YAML activations[].schedule.type
    #       "isRT": False,                    # True if from _rt.activations table
    #       "lastAttempt": None,              # RT only: td_time_string MAX(time)
    #       "lastStatus": None,               # RT only: delivered true/false
    #       "consecutiveFailures": 0,         # RT only: computed via window SQL
    #       "lastError": None                 # RT only: last error text
    #     }
    #   ],
    #   "hasQualityFlags": False,
    #   "qualityFlags": [],
    #   "url": "https://console-next.eu01.treasuredata.com/app/dw/parentSegments/1087307"
    # }
  ],
  "anomalies": [
    # One entry per anomaly (not SKIP):
    # {"segmentName":"...", "segmentId":"...", "anomalyType":"not_refreshed",
    #  "detail":"Last refresh 52h ago, daily schedule (threshold: 26h amber, 50h red)", "severity":"RED"}
  ]
}

# Derive summary — do NOT leave as zeros
segs = overview_data["segments"]
# Derive childSegmentCount from array length
for s in segs:
    if "childSegmentCount" not in s:
        s["childSegmentCount"] = len(s.get("childSegments", []))
overview_data["totalSegments"]                   = len(segs)
overview_data["summary"]["green"]                = sum(1 for s in segs if s["health"] == "GREEN")
overview_data["summary"]["amber"]                = sum(1 for s in segs if s["health"] == "AMBER")
overview_data["summary"]["red"]                  = sum(1 for s in segs if s["health"] == "RED")
overview_data["summary"]["totalChildSegments"]   = sum(len(s.get("childSegments", [])) for s in segs)
overview_data["summary"]["totalPopulation"]      = sum(s.get("population", 0) for s in segs)
overview_data["summary"]["anomalyCount"]         = len(overview_data["anomalies"])

# Safe JSON injection — escape </script> to prevent XSS breakout
json_str = json.dumps(overview_data, ensure_ascii=True)
json_str = json_str.replace("</", "<\\/")

html_template = r"""<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Parent Segment Overview</title>
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
<style>
*,*::before,*::after{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif;background:#f6f7ff;color:#1e1e2e;font-size:14px}
:root{--primary:#494FFF;--sec1:#8753FF;--sec2:#C466D4;--lime:#9DCC4C;--yellow:#D9E47C;--red:#EE2328;--salmon:#F69068}
.header{background:linear-gradient(135deg,rgba(73,79,255,.07),rgba(135,83,255,.04));
        border-bottom:1px solid rgba(73,79,255,.15);padding:20px 28px}
.header h1{font-size:20px;font-weight:800;color:var(--primary)}
.header .sub{font-size:12px;color:#6b7280;margin-top:4px}
.tabs{display:flex;border-bottom:1px solid #e5e7eb;padding:0 28px;background:#fff;overflow-x:auto}
.tab-btn{padding:12px 18px;font-size:13px;font-weight:500;border:none;background:none;
         cursor:pointer;color:#6b7280;border-bottom:2px solid transparent;white-space:nowrap;transition:.15s}
.tab-btn:hover{color:#374151}.tab-btn.active{color:var(--primary);border-bottom-color:var(--primary)}
.content{padding:24px 28px}.tab-panel{display:none}.tab-panel.active{display:block}
.kpi-row{display:grid;grid-template-columns:repeat(6,1fr);gap:12px;margin-bottom:24px}
@media(max-width:1100px){.kpi-row{grid-template-columns:repeat(3,1fr)}}
.kpi{background:#fff;border:1px solid #e5e7eb;border-radius:12px;padding:14px 16px}
.kpi-label{font-size:10px;font-weight:700;text-transform:uppercase;letter-spacing:.08em;color:#9ca3af;margin-bottom:4px}
.kpi-value{font-size:22px;font-weight:800}
.badge{display:inline-block;font-size:10px;font-weight:700;padding:2px 8px;border-radius:20px;color:#fff}
.badge-green{background:#16a34a}.badge-amber{background:#d97706}
.badge-red{background:var(--red)}.badge-skip{background:#9ca3af}
.section-label{font-size:10px;font-weight:700;text-transform:uppercase;letter-spacing:.08em;color:#9ca3af;margin-bottom:12px}
.seg-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(280px,1fr));gap:12px}
.seg-card{background:#fff;border:1px solid #e5e7eb;border-radius:12px;padding:16px;
          cursor:pointer;transition:box-shadow .15s;border-left:4px solid #e5e7eb}
.seg-card:hover{box-shadow:0 4px 12px rgba(0,0,0,.08)}
.seg-card.green{border-left-color:#16a34a}.seg-card.amber{border-left-color:#d97706}
.seg-card.red{border-left-color:var(--red)}
.seg-name{font-size:13px;font-weight:700;margin-bottom:4px;
          white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
.seg-meta{font-size:11px;color:#6b7280;margin-bottom:8px}
.seg-stats{display:flex;gap:12px;font-size:11px;flex-wrap:wrap}
.seg-stat{display:flex;flex-direction:column;gap:1px}
.seg-stat-label{color:#9ca3af;font-size:10px;font-weight:600;text-transform:uppercase}
.seg-stat-value{font-weight:700;color:#1e1e2e}
.pagination{display:flex;align-items:center;gap:8px;margin-top:16px;justify-content:center}
.page-btn{padding:6px 12px;border:1px solid #e5e7eb;border-radius:6px;background:#fff;
          cursor:pointer;font-size:12px;color:#374151}
.page-btn:hover{background:#f3f4f6}.page-btn.active{background:var(--primary);color:#fff;border-color:var(--primary)}
.search-bar{display:flex;gap:8px;margin-bottom:16px;flex-wrap:wrap}
.search-input{flex:1;min-width:180px;border:1px solid #e5e7eb;border-radius:8px;padding:8px 12px;
              font-size:13px;outline:none;background:#fff}
.search-input:focus{border-color:var(--primary)}
.filter-btn{padding:6px 12px;border:1px solid #e5e7eb;border-radius:8px;background:#fff;
            cursor:pointer;font-size:12px;font-weight:500;color:#6b7280;transition:.15s}
.filter-btn.active{background:var(--primary);color:#fff;border-color:var(--primary)}
.drill-panel{display:none;position:fixed;right:0;top:0;height:100vh;width:420px;
             background:#fff;border-left:1px solid #e5e7eb;z-index:100;
             overflow-y:auto;padding:20px;box-shadow:-4px 0 20px rgba(0,0,0,.08)}
.drill-panel.open{display:block}
.drill-close{position:absolute;top:12px;right:16px;font-size:20px;cursor:pointer;
             background:none;border:none;color:#6b7280}
.drill-title{font-size:15px;font-weight:800;margin-bottom:4px;padding-right:32px}
.drill-section{margin-top:16px}
.drill-section h3{font-size:10px;font-weight:700;text-transform:uppercase;
                  letter-spacing:.06em;color:#9ca3af;margin-bottom:6px;
                  border-bottom:1px solid #f3f4f6;padding-bottom:4px}
.child-seg-row{display:flex;align-items:center;padding:5px 0;
               border-bottom:1px solid #f9fafb;font-size:12px;gap:8px}
.child-seg-name{font-weight:500;flex:1;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
.activity-item{display:flex;gap:10px;padding:8px 0;border-bottom:1px solid #f3f4f6}
.activity-dot{width:8px;height:8px;border-radius:50%;flex-shrink:0;margin-top:4px}
.activity-title{font-size:12px;font-weight:500}
.activity-meta{font-size:11px;color:#6b7280;margin-top:1px}
.anomaly-card{border-radius:8px;padding:12px;margin-bottom:8px}
.anomaly-red{border:1px solid #fca5a5;background:#fff5f5}
.anomaly-amber{border:1px solid #fcd34d;background:#fffbeb}
.all-good{text-align:center;padding:48px;color:#16a34a;font-size:15px;font-weight:600}
.chart-row{display:grid;grid-template-columns:1fr 1fr;gap:16px;margin-top:20px}
@media(max-width:700px){.chart-row{grid-template-columns:1fr}}
.chart-card{background:#fff;border:1px solid #e5e7eb;border-radius:12px;padding:16px}
.chart-card h3{font-size:11px;font-weight:700;color:#374151;margin-bottom:12px;
               text-transform:uppercase;letter-spacing:.06em}
.chart-wrap{position:relative;height:220px}
.gap-banner{background:#fefce8;border:1px solid #fde047;border-radius:8px;
            padding:12px 16px;font-size:12px;color:#854d0e;margin-bottom:16px}
.info-banner{background:#eff6ff;border:1px solid #93c5fd;border-radius:8px;
             padding:12px 16px;font-size:12px;color:#1d4ed8;margin-bottom:16px}
.card{background:#fff;border:1px solid #e5e7eb;border-radius:10px;padding:12px 16px;margin-bottom:8px}
.card-header{display:flex;align-items:center;gap:8px;margin-bottom:8px;flex-wrap:wrap}
.card-title{font-size:13px;font-weight:700;color:#1e1e2e}
.card-db{font-size:11px;color:#6b7280;margin-left:auto}
</style>
</head>
<body>
<div class="header">
  <h1>Parent Segment Overview</h1>
  <div class="sub" id="h-sub"></div>
</div>
<div class="tabs">
  <button class="tab-btn active" onclick="showTab('health')">Segment Health</button>
  <button class="tab-btn"        onclick="showTab('activation')">Activation Summary</button>
  <button class="tab-btn"        onclick="showTab('activity')">Activity Feed</button>
  <button class="tab-btn"        onclick="showTab('anomalies')">Anomalies</button>
</div>
<div class="content">
  <div id="tab-health"     class="tab-panel active"></div>
  <div id="tab-activation" class="tab-panel"></div>
  <div id="tab-activity"   class="tab-panel"></div>
  <div id="tab-anomalies"  class="tab-panel"></div>
</div>
<div id="drill-panel" class="drill-panel">
  <button class="drill-close" onclick="closeDrill()">✕</button>
  <div class="drill-title" id="drill-title"></div>
  <div id="drill-content"></div>
</div>
<script>
const D = __OVERVIEW_DATA_JSON__;
const C={primary:"#494FFF",sec1:"#8753FF",sec2:"#C466D4",lime:"#9DCC4C",
         yellow:"#D9E47C",red:"#EE2328",salmon:"#F69068"};
const HC={GREEN:"#16a34a",AMBER:"#d97706",RED:"#EE2328"};
const PAGE_SIZE=20;
let activeFilter="all", searchTerm="", currentPage=0;

function badge(h){
  const m={GREEN:["badge-green","✅ ON SCHEDULE"],AMBER:["badge-amber","⚠️ DELAYED"],
           RED:["badge-red","🔴 OVERDUE"]};
  const [cls,lbl]=m[h]||["badge-skip",h];
  return `<span class="badge ${cls}">${lbl}</span>`;
}
function esc(s){return String(s||"").replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;")}
function fmt(n){return n!=null?Number(n).toLocaleString():"—"}
function relTime(iso){
  if(!iso) return "never";
  const h=(new Date()-new Date(iso))/3600000;
  if(h<1) return `${Math.round(h*60)}m ago`;
  if(h<24) return `${Math.round(h)}h ago`;
  return `${Math.round(h/24)}d ago`;
}
function showTab(id){
  document.querySelectorAll(".tab-panel").forEach(p=>p.classList.remove("active"));
  document.querySelectorAll(".tab-btn").forEach(b=>b.classList.remove("active"));
  document.getElementById("tab-"+id).classList.add("active");
  ["health","activation","activity","anomalies"].forEach((l,i)=>{
    if(l===id) document.querySelectorAll(".tab-btn")[i].classList.add("active");
  });
}

// ── Drill-down ──────────────────────────────────────────────────────────────
function openDrill(segId){
  const seg=D.segments.find(s=>s.id===segId); if(!seg) return;
  document.getElementById("drill-title").textContent=seg.name;
  const childRows=(seg.childSegments||[]).map(c=>
    `<div class="child-seg-row">
      <span class="child-seg-name" title="${esc(c.name)}">${esc(c.name)}</span>
      <span style="color:#6b7280;font-size:11px;white-space:nowrap">${fmt(c.population)} profiles</span>
      <span style="color:#9ca3af;font-size:10px;white-space:nowrap">${relTime(c.createdAt)}</span>
    </div>`
  ).join("")||'<div style="color:#9ca3af;font-size:12px">No child segments</div>';
  const actRows=(seg.activations||[]).map(a=>
    `<div class="child-seg-row">
      <span class="child-seg-name">${esc(a.name)}</span>
      <span style="color:#6b7280;font-size:11px">${esc(a.type||a.connection||'—')}</span>
      <span style="font-size:10px">${a.isRT
        ?(a.consecutiveFailures>0?`<span style="color:#EE2328">🔴 ${a.consecutiveFailures} failures</span>`
          :(a.lastStatus==='true'?'<span style="color:#16a34a">✅ OK</span>':'—'))
        :'<span style="color:#9ca3af">Config only</span>'}</span>
    </div>`
  ).join('')||'<div style="color:#9ca3af;font-size:12px">No activations configured</div>';
  const flags=(seg.qualityFlags||[]).map(f=>
    `<div style="font-size:12px;color:#dc2626;padding:3px 0">${esc(f.issue_type)}</div>`
  ).join("")||'<div style="font-size:12px;color:#16a34a">No quality flags</div>';
  document.getElementById("drill-content").innerHTML=`
    <div style="margin-bottom:10px">${badge(seg.health)}</div>
    <div style="display:grid;grid-template-columns:1fr 1fr;gap:8px;margin-bottom:14px">
      <div><div style="font-size:10px;color:#9ca3af;font-weight:700;text-transform:uppercase">Population</div>
           <div style="font-size:18px;font-weight:800">${fmt(seg.population)}</div></div>
      <div><div style="font-size:10px;color:#9ca3af;font-weight:700;text-transform:uppercase">Schedule</div>
           <div style="font-size:14px;font-weight:600">${seg.scheduleType||"—"}</div></div>
      <div><div style="font-size:10px;color:#9ca3af;font-weight:700;text-transform:uppercase">Database</div>
           <div style="font-size:12px;font-weight:600;font-family:monospace">${esc(seg.database||"—")}</div></div>
      <div><div style="font-size:10px;color:#9ca3af;font-weight:700;text-transform:uppercase">Last Refresh</div>
           <div style="font-size:13px;font-weight:600">${relTime(seg.matrixUpdatedAt)}</div></div>
    </div>
    <div class="drill-section">
      <h3>Child Segments (${seg.childSegmentCount||0})</h3>${childRows}
    </div>
    <div class="drill-section">
      <h3>Activations (${(seg.activations||[]).length})</h3>${actRows}
    </div>
    <div class="drill-section">
      <h3>Quality Flags</h3>${flags}
    </div>
    <div class="drill-section">
      <h3>Deep Analysis</h3>
      <div style="font-size:12px;color:#374151">
        Use <code style="background:#f3f4f6;padding:2px 6px;border-radius:4px;font-size:11px">tdx-skills:parent-segment-analysis</code>
        for deep analysis of this segment.<br>
        <span style="font-size:10px;color:#9ca3af;display:block;margin-top:4px">
          uc-rfm-segmentation not available in this account.
        </span>
        <a href="${esc(seg.url||"#")}" target="_blank"
           style="font-size:12px;color:var(--primary);display:block;margin-top:6px">
          Open in Treasure Data Console ↗
        </a>
      </div>
    </div>`;
  document.getElementById("drill-panel").classList.add("open");
}
function closeDrill(){document.getElementById("drill-panel").classList.remove("open");}

// ── Tab: Segment Health ─────────────────────────────────────────────────────
function buildHealth(){
  const panel=document.getElementById("tab-health");
  const s=D.summary;
  panel.innerHTML=`
    <div class="kpi-row">
      <div class="kpi"><div class="kpi-label">Total</div><div class="kpi-value" style="color:${C.primary}">${fmt(D.totalSegments)}</div></div>
      <div class="kpi"><div class="kpi-label">On Schedule</div><div class="kpi-value" style="color:#16a34a">${fmt(s.green)}</div></div>
      <div class="kpi"><div class="kpi-label">Delayed</div><div class="kpi-value" style="color:#d97706">${fmt(s.amber)}</div></div>
      <div class="kpi"><div class="kpi-label">Overdue</div><div class="kpi-value" style="color:${C.red}">${fmt(s.red)}</div></div>
      <div class="kpi"><div class="kpi-label">Total Profiles</div><div class="kpi-value" style="color:${C.sec1}">${fmt(s.totalPopulation)}</div></div>
      <div class="kpi"><div class="kpi-label">Child Segments</div><div class="kpi-value" style="color:${C.sec2}">${fmt(s.totalChildSegments)}</div></div>
    </div>
    <div class="search-bar">
      <input class="search-input" id="seg-search" placeholder="Search segments by name…" oninput="applyFilter()"/>
      <button class="filter-btn active" onclick="setFilter('all',this)">All</button>
      <button class="filter-btn" onclick="setFilter('red',this)">🔴 Overdue</button>
      <button class="filter-btn" onclick="setFilter('amber',this)">⚠️ Delayed</button>
      <button class="filter-btn" onclick="setFilter('green',this)">✅ Healthy</button>
    </div>
    <div class="seg-grid" id="seg-grid"></div>
    <div class="pagination" id="pagination"></div>
    <div class="chart-row">
      <div class="chart-card"><h3>Health Distribution</h3>
        <div class="chart-wrap"><canvas id="chart-health"></canvas></div></div>
      <div class="chart-card"><h3>Top 10 by Population</h3>
        <div class="chart-wrap"><canvas id="chart-pop"></canvas></div></div>
    </div>`;
  renderSegGrid();
  new Chart(document.getElementById("chart-health"),{
    type:"doughnut",
    data:{labels:["On Schedule","Delayed","Overdue"],
          datasets:[{data:[s.green,s.amber,s.red],
                     backgroundColor:["#16a34a","#d97706",C.red],borderWidth:0}]},
    options:{responsive:true,maintainAspectRatio:false,
             plugins:{legend:{position:"right",labels:{font:{size:11}}}}}
  });
  const top10=[...D.segments].sort((a,b)=>b.population-a.population).slice(0,10);
  new Chart(document.getElementById("chart-pop"),{
    type:"bar",
    data:{labels:top10.map(s=>s.name.length>18?s.name.slice(0,16)+"…":s.name),
          datasets:[{data:top10.map(s=>s.population),
                     backgroundColor:top10.map(s=>HC[s.health]||"#6b7280"),borderWidth:0}]},
    options:{responsive:true,maintainAspectRatio:false,plugins:{legend:{display:false}},
             scales:{y:{ticks:{font:{size:10}}},x:{ticks:{font:{size:9},maxRotation:45}}}}
  });
}
function filteredSegs(){
  const term=(document.getElementById("seg-search")?.value||"").toLowerCase();
  return D.segments.filter(s=>{
    if(activeFilter!=="all"&&s.health.toLowerCase()!==activeFilter) return false;
    if(term&&!s.name.toLowerCase().includes(term)) return false;
    return true;
  });
}
function renderSegGrid(){
  const grid=document.getElementById("seg-grid"); if(!grid) return;
  const segs=filteredSegs();
  const page=segs.slice(currentPage*PAGE_SIZE,(currentPage+1)*PAGE_SIZE);
  grid.innerHTML=page.map(seg=>`
    <div class="seg-card ${seg.health.toLowerCase()}" onclick="openDrill('${seg.id}')">
      <div class="seg-name" title="${esc(seg.name)}">${esc(seg.name)}</div>
      <div class="seg-meta">${badge(seg.health)}&nbsp;
        <span style="font-size:10px;color:#9ca3af">${seg.scheduleType||"manual"}</span>
        ${seg.database?`<span style="font-size:10px;color:#9ca3af"> · ${esc(seg.database)}</span>`:""}
      </div>
      <div class="seg-stats">
        <div class="seg-stat"><span class="seg-stat-label">Profiles</span>
          <span class="seg-stat-value">${fmt(seg.population)}</span></div>
        <div class="seg-stat"><span class="seg-stat-label">Children</span>
          <span class="seg-stat-value">${seg.childSegmentCount||0}</span></div>
        <div class="seg-stat"><span class="seg-stat-label">Last Refresh</span>
          <span class="seg-stat-value">${relTime(seg.matrixUpdatedAt)}</span></div>
      </div>
      ${seg.hasQualityFlags?'<div style="font-size:10px;color:#d97706;margin-top:6px">⚠️ Quality flags</div>':""}
      ${seg.health==="RED"&&seg.scheduleType==="none"?'<div style="font-size:10px;color:#9ca3af;margin-top:4px">Manual segment — verify if refresh expected</div>':""}
    </div>`
  ).join("")||'<div style="color:#9ca3af;padding:20px">No segments match filter.</div>';
  renderPagination(segs.length);
}
function renderPagination(total){
  const pages=Math.ceil(total/PAGE_SIZE);
  const pg=document.getElementById("pagination"); if(!pg) return;
  if(pages<=1){pg.innerHTML="";return;}
  const showing=`<span style="font-size:12px;color:#6b7280">Showing ${currentPage*PAGE_SIZE+1}–${Math.min((currentPage+1)*PAGE_SIZE,total)} of ${total}</span>`;
  const prev=currentPage>0?`<button class="page-btn" onclick="goPage(${currentPage-1})">← Prev</button>`:"";
  const next=currentPage<pages-1?`<button class="page-btn" onclick="goPage(${currentPage+1})">Next →</button>`:"";
  pg.innerHTML=prev+showing+next;
}
function goPage(p){currentPage=p;renderSegGrid();}
function setFilter(f,btn){
  activeFilter=f;currentPage=0;
  document.querySelectorAll(".filter-btn").forEach(b=>b.classList.remove("active"));
  btn.classList.add("active");
  renderSegGrid();
}
function applyFilter(){currentPage=0;renderSegGrid();}

// ── Tab: Activation Summary ─────────────────────────────────────────────────
function buildActivation(){
  const panel=document.getElementById("tab-activation");

  // Collect all segments that have activation data
  const segsWithActs = D.segments.filter(s=>(s.activations||[]).length>0);
  const rtSegs       = segsWithActs.filter(s=>s.activations.some(a=>a.isRT));
  const batchSegs    = segsWithActs.filter(s=>s.activations.some(a=>!a.isRT));
  const noActSegs    = D.segments.filter(s=>!(s.activations||[]).length);

  // Activation type breakdown across all segments
  const typeCount = {};
  D.segments.forEach(s=>(s.activations||[]).forEach(a=>{
    const t = a.type||"unknown";
    typeCount[t]=(typeCount[t]||0)+1;
  }));

  const rtFailures = D.segments.flatMap(s=>
    (s.activations||[]).filter(a=>a.isRT && a.consecutiveFailures>0)
      .map(a=>({segName:s.name, ...a}))
  );

  panel.innerHTML=`
    <div class="kpi-row" style="grid-template-columns:repeat(4,1fr)">
      <div class="kpi"><div class="kpi-label">Segments with Activations</div>
        <div class="kpi-value" style="color:${C.primary}">${fmt(segsWithActs.length)}</div></div>
      <div class="kpi"><div class="kpi-label">RT Activations</div>
        <div class="kpi-value" style="color:${C.lime}">${fmt(rtSegs.length)}</div></div>
      <div class="kpi"><div class="kpi-label">Batch Activations</div>
        <div class="kpi-value" style="color:${C.sec1}">${fmt(batchSegs.length)}</div></div>
      <div class="kpi"><div class="kpi-label">RT Consecutive Failures</div>
        <div class="kpi-value" style="color:${rtFailures.length>0?C.red:C.lime}">${fmt(rtFailures.length)}</div></div>
    </div>

    ${Object.keys(typeCount).length ? `
    <div style="margin-bottom:16px">
      <div class="section-label">Connector Types in Use</div>
      <div style="display:flex;flex-wrap:wrap;gap:8px">
        ${Object.entries(typeCount).map(([t,n])=>
          `<span class="badge badge-skip" style="background:${C.sec1};font-size:12px;padding:4px 10px">
            ${esc(t)} (${n})
          </span>`
        ).join("")}
      </div>
    </div>` : ""}

    ${rtFailures.length ? `
    <div style="margin-bottom:16px">
      <div class="section-label">⚠️ RT Activations with Consecutive Failures</div>
      ${rtFailures.map(a=>`
        <div class="anomaly-card anomaly-red">
          <div style="font-weight:700;font-size:13px;color:#dc2626">${esc(a.segName)} — ${esc(a.name)}</div>
          <div style="font-size:11px;color:#6b7280">${esc(a.type)} · ${a.consecutiveFailures} consecutive failures · Last: ${esc(a.lastAttempt||"—")}</div>
          ${a.lastError?`<div style="font-size:11px;color:#dc2626;margin-top:2px;font-family:monospace">${esc(a.lastError)}</div>`:""}
        </div>`).join("")}
    </div>` : ""}

    <div style="margin-bottom:16px">
      <div class="section-label">All Activations by Segment</div>
      ${segsWithActs.length===0
        ? '<div style="color:#9ca3af;font-size:13px;padding:12px">No activations configured on any segment.</div>'
        : segsWithActs.map(seg=>`
          <div class="card" style="margin-bottom:8px">
            <div class="card-header">
              <span class="card-title">${esc(seg.name)}</span>
              <span style="font-size:11px;color:#9ca3af;margin-left:auto">${seg.activations.length} activation${seg.activations.length!==1?"s":""}</span>
            </div>
            <table style="width:100%;border-collapse:collapse;font-size:12px">
              <thead><tr style="color:#9ca3af">
                <th style="text-align:left;padding:3px 8px 3px 0;font-size:10px;font-weight:700;text-transform:uppercase">Name</th>
                <th style="text-align:left;padding:3px 8px 3px 0;font-size:10px;font-weight:700;text-transform:uppercase">Type</th>
                <th style="text-align:left;padding:3px 8px 3px 0;font-size:10px;font-weight:700;text-transform:uppercase">Schedule</th>
                <th style="text-align:left;padding:3px 0;font-size:10px;font-weight:700;text-transform:uppercase">Status</th>
              </tr></thead>
              <tbody>
                ${seg.activations.map(a=>`
                  <tr style="border-top:1px solid #f3f4f6">
                    <td style="padding:5px 8px 5px 0">${esc(a.name)}</td>
                    <td style="padding:5px 8px 5px 0;color:#6b7280">${esc(a.type||a.connection||"—")}</td>
                    <td style="padding:5px 8px 5px 0;color:#6b7280">${esc(a.scheduleType||"—")}</td>
                    <td style="padding:5px 0">
                      ${a.isRT
                        ? (a.consecutiveFailures>0
                            ? `<span class="badge badge-red">🔴 ${a.consecutiveFailures} failures</span>`
                            : (a.lastStatus==="true"
                                ? `<span class="badge badge-green">✅ OK</span>`
                                : `<span class="badge badge-skip">—</span>`))
                        : `<span class="badge badge-skip" title="Batch run history not available via CLI">Config only</span>`}
                    </td>
                  </tr>`).join("")}
              </tbody>
            </table>
          </div>`).join("")}
    </div>

    ${noActSegs.length ? `<div style="color:#9ca3af;font-size:12px;padding:8px 0;border-top:1px solid #f3f4f6;margin-top:8px">${noActSegs.length} segment${noActSegs.length!==1?"s":""} have no activations configured.</div>` : ""}

    <div class="info-banner" style="margin-top:8px">
      ℹ️ <strong>Batch activation run history</strong> is not available via the tdx CLI —
      only configuration is shown. RT activation status comes from
      <code>cdp_audience_&lt;id&gt;_rt.activations</code> table (last 7 days).
      For full activation history visit the
      <a href="https://console-next.eu01.treasuredata.com" target="_blank" style="color:#1d4ed8">
        Treasure Data Console ↗
      </a>
    </div>`;
}

// ── Tab: Activity Feed ──────────────────────────────────────────────────────
function buildActivity(){
  const panel=document.getElementById("tab-activity");
  const allChildren=D.segments.flatMap(ps=>
    (ps.childSegments||[]).map(c=>({...c,parentName:ps.name,parentId:ps.id}))
  );
  allChildren.sort((a,b)=>new Date(b.createdAt)-new Date(a.createdAt));
  const recent=allChildren.slice(0,50);
  const items=recent.map(c=>`
    <div class="activity-item">
      <div class="activity-dot" style="background:${C.primary}"></div>
      <div class="activity-content">
        <div class="activity-title">${esc(c.name)}</div>
        <div class="activity-meta">
          In <strong>${esc(c.parentName)}</strong> ·
          ${fmt(c.population)} profiles ·
          Created ${relTime(c.createdAt)}
          <span style="color:#9ca3af;font-size:10px"> · Author not available via CLI</span>
        </div>
      </div>
    </div>`
  ).join("")||'<div style="color:#9ca3af;padding:20px">No child segments found.</div>';
  panel.innerHTML=`
    <div class="section-label">Recent Child Segments (last 50 by creation date)</div>
    <div style="font-size:11px;color:#9ca3af;margin-bottom:12px">
      ℹ️ "Created by" (author) is not available via tdx sg list — showing time and size only.
    </div>
    ${items}`;
}

// ── Tab: Anomalies ──────────────────────────────────────────────────────────
function buildAnomalies(){
  const panel=document.getElementById("tab-anomalies");
  const anomalies=D.anomalies||[];
  if(!anomalies.length){
    panel.innerHTML='<div class="all-good">✅ No anomalies detected — all segments within expected parameters.</div>';
  } else {
    const red=anomalies.filter(a=>a.severity==="RED");
    const amber=anomalies.filter(a=>a.severity==="AMBER");
    const render=(items,cls,tc)=>items.map(a=>
      `<div class="anomaly-card ${cls}">
        <div style="font-weight:700;font-size:13px;color:${tc}">${esc(a.segmentName)}</div>
        <div style="font-size:11px;color:#6b7280;margin-top:2px">${esc(a.anomalyType.replace(/_/g," "))}</div>
        <div style="font-size:12px;color:#374151;margin-top:4px">${esc(a.detail)}</div>
      </div>`
    ).join("");
    panel.innerHTML=`
      <div class="section-label">Critical — ${red.length} issue${red.length!==1?"s":""}</div>
      ${red.length?render(red,"anomaly-red","#dc2626"):'<div style="color:#16a34a;font-size:13px;margin-bottom:12px">No critical issues.</div>'}
      <div class="section-label" style="margin-top:20px">Warnings — ${amber.length} issue${amber.length!==1?"s":""}</div>
      ${amber.length?render(amber,"anomaly-amber","#d97706"):'<div style="color:#16a34a;font-size:13px">No warnings.</div>'}`;
  }
  // Always append documented gaps section
  panel.innerHTML+=`
    <div style="margin-top:24px">
      <div class="section-label">Anomalies Not Detectable via CLI</div>
      <div style="font-size:12px;color:#6b7280;background:#f9fafb;border:1px solid #e5e7eb;
                  border-radius:8px;padding:12px">
        <div style="margin-bottom:6px">⚪ <strong>Batch activation run history</strong> — only config available via tdx sg pull YAML (no run status)</div>
        <div>⚪ <strong>Child segment size outliers</strong> — requires historical baseline (not available on first run)</div>
      </div>
    </div>`;
}

(function init(){
  document.getElementById("h-sub").textContent=
    `Account: ${D.accountId} · ${D.totalSegments} segments · Run: ${D.runDate}`;
  buildHealth();
  buildActivation();
  buildActivity();
  buildAnomalies();
})();
</script>
</body>
</html>"""

html_out = html_template.replace("__OVERVIEW_DATA_JSON__", json_str, 1)
date_str = datetime.datetime.utcnow().strftime("%Y%m%d")
filename = f"ps_overview_{date_str}.html"
with open(filename, "w") as f:
    f.write(html_out)
print(f"✅ Dashboard written: {filename} ({len(html_out):,} bytes)")
print(f"   Open in browser: open {filename}")
PYEOF
```

---

## Step 5 — Show Your Work

Output after the dashboard:

```
=== PARENT SEGMENT OVERVIEW ===
Account: <ACCOUNT_ID> | Run: <timestamp>
Segments in scope: <N> (filter: <pattern or "all">)

HEALTH SUMMARY:
  ✅ On schedule: <N>
  ⚠️ Delayed:     <N>
  🔴 Overdue:     <N>

ACTIVATION SUMMARY:
  Segments with activations: <N>
  RT activations (with status): <N>
  Batch activations (config only): <N>
  RT consecutive failures: <N>

Commands run:
  tdx ps list --json                     → <N> segments retrieved
  tdx ps view × <N>                      → database names retrieved
  tdx ps use + tdx sg list × <N>         → <N> total child segments
  tdx ps use + tdx sg pull --yes × <N>   → <N> activation configs retrieved
  tdx connection list                    → connections verified
  cdp_audience_<id>_rt.activations × <N> → RT activation status (last 7d)
  ai_usage.data_quality_flags            → <N> quality flags (or "not found")

ANOMALIES: <N> total
  🔴 <segment>: rt_activation_failures — <N> consecutive failures on <activation>
  🟡 <segment>: delayed — last refresh <X>h ago (threshold: <Y>h)
```

---

## Acceptance Criteria Checklist

- [ ] Dashboard renders for accounts with 1–20+ parent segments (pagination handles >20)
- [ ] Health status calculated from `matrixUpdatedAt` vs `scheduleType` — NOT hardcoded
- [ ] Anomaly flags surface automatically without user prompting
- [ ] Activation Summary tab shows real data: batch config from `tdx sg pull` YAML + RT status from `_rt.activations`
- [ ] RT consecutive failures detected and shown as RED anomalies
- [ ] "No activations configured" state shown when no activations found (ticket requirement)
- [ ] Batch activation config shown (name, connector, schedule) — run history documented as unavailable
- [ ] Activity Feed shows child segments with time and size (author documented as unavailable)
- [ ] Drill-down opens on card click — shows database, activations, child segments, quality flags
- [ ] `uc-rfm-segmentation` gap documented — drill-down uses `parent-segment-analysis`
- [ ] `scheduleType: "none"` RED cards carry "verify if refresh expected" note
- [ ] `</script>` XSS injection prevented via `ensure_ascii=True` + replace
- [ ] Performance: user warned about time estimate before running 140 segments
- [ ] Usage logged to `ai_usage.skills_usage_tracker`
- [ ] HTML written with date-stamped filename, user told to open in external browser

---

## Verified CLI Syntax (tested on eu01)

```bash
tdx ps list --json                    # all segments — scheduleType, matrixUpdatedAt, url
tdx ps list "<pattern>" --json        # filtered — supports * and ?
tdx ps view "<name>" --json           # single segment — master.parentDatabaseName
tdx ps use "<name>"                   # set context (required before tdx sg commands)
tdx sg list --json                    # child segments (requires ps context)
tdx sg pull --yes                     # pull segment YAMLs incl. activations: block
tdx connection list                   # list all configured connections (name + type)
# Note: verify this command is available in your TAS environment before relying on it
```

**Activation data sources:**
- Batch activations: `tdx sg pull --yes` → `activations:` block in segment YAML (config only, no run history)
- RT activations: `cdp_audience_<id>_rt.activations` table → `activation_name`, `activation_type`, `delivered`, `status`, `error`, `time`
- RT consecutive failures: computed via SQL window function over `delivered` column

**Fields available in `tdx ps list --json`:** `name`, `population`, `scheduleType`,
`matrixUpdatedAt`, `updatedAt`, `createdAt`, `url`

**Fields NOT available via tdx CLI:**
- Batch activation run history (last run time, success/fail)
- Last PS workflow run status (use `tdx wf sessions <project>` if workflow name known)
- Child segment author (`createdBy`)
