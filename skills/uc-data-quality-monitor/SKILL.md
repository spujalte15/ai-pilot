---
name: uc-data-quality-monitor
description: |
  Customer-facing skill for Data Operations pilot users (ARCH-1325).
  Monitors fill rate, completeness, and ID stitching health across the
  customer's CDP data. Flags issues proactively with a green/amber/red
  health scorecard and trend over last 7 runs. Governance & Operations
  use case — relevant for Data/Engineering-led POCs (SC1 + SC4 + SC5).
  Triggers on: "data quality", "health check", "data monitor", "fill rate",
  "completeness check", "id stitching health", "data health", "quality monitor",
  "uc-data-quality-monitor", "CDP health", "check my data quality",
  "are there data issues", "monitor my tables".
  Team: Governance & Ops. JIRA: ARCH-1325. POC Framework Phase: Weeks 4–6.
  NOTE: Lighter customer-facing monitoring view only — no remediation guidance.
  For full SE-facing deep diagnostics use cdp-assessment instead.
---

# uc-data-quality-monitor — CDP Data Health Monitor

Monitor fill rate, completeness, ID stitching coverage, and deduplication health
across a customer's CDP tables. Produces a green/amber/red health scorecard with
trend over last 7 runs as a self-contained HTML file.

**Scope:** Always-on health checks only. No remediation guidance — that stays in
`cdp-assessment`. Do NOT replicate `cdp-assessment` logic here.

---

## ⚠️ Marketplace Skill Availability Notice

| Skill | Status | Resolution |
|---|---|---|
| `cdp-assessment` | ✅ Available (local SE skill) | Do NOT call it — this is a lighter customer-facing version |
| `tdx-skills:tdx-basic` | ✅ Available | Used throughout |
| `sql-skills:trino` | ✅ Available | Used for all queries |
| `treasure-work-skills:react-dashboard` | ❌ NOT available in TAS | Step 5 generates self-contained HTML (Chart.js CDN) |
| `uc-data-lineage` | ✅ Available (this repo) | Step 4 writes `ai_usage.data_quality_flags` consumed by uc-data-lineage Step 6 |

> **Tracking table note:** JIRA ticket specifies `{customer_slug}_ai_poc_tracking.skill_usage`.
> This skill standardises to `ai_usage.skills_usage_tracker` per ARCH-1335 platform pattern,
> pending clarification from Sébastien Pujalte on whether a customer-scoped table is intended.

> **CDN note:** The generated HTML uses Chart.js CDN. Charts render correctly when the file is
> opened in an external browser (Chrome/Safari/Firefox). Treasure Work's built-in file viewer
> is sandboxed and blocks CDN — charts will appear blank there. Always open via browser.

---

## Step 0 — Log Invocation

**THIS STEP IS MANDATORY. Execute it completely and confirm before moving to Step 1. Do NOT merge this step with any other step.**

### 0a — Request TD credentials (separate tool call)

Call `mcp__tas__request_credential` with `td_api_production_eu01`.
Wait for the green "Authenticated" confirmation before proceeding.
**Do not run any bash until this succeeds.**

### 0b — Log to tracking table (separate bash call — do NOT combine with Step 1 or any other step)

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
  "INSERT INTO skills_usage_tracker (id, user_id, account_id, skill_name) VALUES ('$RECORD_ID', '$USER_ID', '$ACCOUNT_ID', 'uc-data-quality-monitor')" \
  2>&1

echo "✅ Tracking complete — user: $USER_ID | record: $RECORD_ID"
```

**Store `RECORD_ID` value** — you will need it in Step 3 for snapshot tracking.

### 0c — Confirm before continuing

Output:
```
✅ Step 0 complete — uc-data-quality-monitor invocation logged for <USER_ID>
```
**Only proceed to Step 1 after outputting this confirmation.**

---

## Step 1 — Scope Monitoring Targets

Ask the user these specific questions:

1. **Which parent segment to monitor?** (provide name — the skill will automatically discover its master table and behaviour tables)
2. **Are any of these behaviour table types present?** Orders / Purchases, Email events (send/open/click), Pageviews / Web events — provide table names if known, or the skill will discover them automatically.
3. **Staging and golden databases accessible?** (provide names e.g. `stg_acme`, `gld_acme` — or skip)
4. **Custom thresholds?** (optional — if omitted, defaults below apply)

**Default thresholds:**
| Check | RED threshold | AMBER threshold |
|---|---|---|
| Fill rate — key column | < 80% | 80–95% |
| Row count change | > 10% drop OR > 50% spike | 5–10% drop OR 20–50% spike |
| ID stitching coverage | < 70% | 70–90% |
| Duplicate rate | > 1% | 0.5–1% |

Wait for user answers. Store as `TARGET_SEGMENT`, `BEHAVIOUR_TABLES`, `STAGING_DB`, `GOLDEN_DB`, `CUSTOM_THRESHOLDS`.

**After Step 2 discovers tables, present the full list to the user and ask: "These are the tables I will monitor — confirm to proceed or adjust the scope."** Wait for confirmation before running health checks.

---

## Step 2 — Discover Tables in Scope

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01

# Discover parent segment output database
tdx ps list 2>&1 | grep -i "<TARGET_SEGMENT>"

# Describe parent segment to get output DB and table names
tdx ps desc -o ps_schema.json "<TARGET_SEGMENT>"
cat ps_schema.json
```

From `ps_schema.json` extract:
- `OUTPUT_DB` = `cdp_audience_<id>`
- `MASTER_TABLE` = typically `customers` — **confirm from ps_schema.json, do not hardcode**
- `CANONICAL_ID_COL` = look for the first column containing `id` in the master table schema that is NOT `time` — candidates: `cdp_customer_id`, `canonical_id`, `contact_id`, `customer_id`. Use `tdx describe <OUTPUT_DB>.<MASTER_TABLE> --json` to inspect columns and identify the canonical ID column.
- `BEHAVIOUR_TABLES` = all `behavior_*` tables from ps_schema.json — prioritise: orders/purchases, email send/open/click, pageview/web tables. If user named specific tables, use those.

For staging and golden databases (if provided):
```bash
tdx tables "<STAGING_DB>.*" --json 2>&1
tdx tables "<GOLDEN_DB>.*" --json 2>&1
```

Also list tables in the parent segment output DB:
```bash
tdx tables "<OUTPUT_DB>.*" --json 2>&1
```

For each table to be monitored, get the column list:
```bash
tdx describe "<DB>.<TABLE>" 2>&1
```

Key column heuristics: columns named `*_id`, `*_key`, `email*`, `phone*`, `<CANONICAL_ID_COL>`, `time`.

Build `TABLES_IN_SCOPE` list: `[{db, table, layer, keyColumns[], primaryKeyCol}]`

**Present this list to the user for confirmation before proceeding to Step 3.**

---

## Step 3 — Run Health Checks

Run all four checks for every table in `TABLES_IN_SCOPE`.

**Important:** Each check is a separate bash call. Do NOT combine multiple queries into one block.
Re-export credentials at the start of each bash call:
```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
```

Also re-declare `RECORD_ID` at the top of each bash call (set it to the value confirmed in Step 0):
```bash
RECORD_ID="<value from Step 0>"
```

---

### Check A — Fill Rate (per key column)

For each table, for each key column:

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
tdx query --database <DB> \
  "SELECT COUNT(*) AS total_rows, COUNT(<COLUMN>) AS non_null_count,
   ROUND(COUNT(<COLUMN>) * 100.0 / COUNT(*), 2) AS fill_rate_pct
   FROM <TABLE>" 2>&1
```

Classify against fill rate thresholds. Record each result in `FILL_RATE_RESULTS`.

---

### Check B — Row Count + Trend (last 7 runs)

**Step B1 — Get current count:**
```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
tdx query --database <DB> "SELECT COUNT(*) AS row_count FROM <TABLE>" 2>&1
```

Capture the integer result as `CURRENT_COUNT`.

**Step B2 — Ensure metric history table exists:**
```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
tdx api -X POST /v3/database/create/ai_usage 2>/dev/null || true
tdx api -X POST /v3/table/create/ai_usage/dq_metric_history/log 2>/dev/null || true
tdx api -X POST \
  --data '{"schema":"[[\"run_id\",\"string\"],[\"table_fqn\",\"string\"],[\"check_type\",\"string\"],[\"column_name\",\"string\"],[\"metric_value\",\"double\"],[\"status\",\"string\"]]"}' \
  /v3/table/update-schema/ai_usage/dq_metric_history 2>/dev/null || true
```

**Step B3 — Query last 7 runs for trend:**
```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
tdx query --database ai_usage \
  "SELECT run_id, metric_value, status, td_time_string(time,'s!','UTC') as run_date
   FROM dq_metric_history
   WHERE table_fqn = '<DB>.<TABLE>' AND check_type = 'row_count'
   ORDER BY time DESC LIMIT 7" 2>&1
```

If the query returns rows, store as `ROW_COUNT_HISTORY` (list of `{run_id, metric_value, status, run_date}`).
If no rows: `ROW_COUNT_HISTORY = []` — first run, baseline being established.

Compute change vs previous run:
- If `ROW_COUNT_HISTORY` has ≥1 entry: `prev = ROW_COUNT_HISTORY[0].metric_value`; `change_pct = (CURRENT_COUNT - prev) / prev * 100`
- Flag: drop >10% → RED, drop 5–10% → AMBER, spike >50% → RED, spike 20–50% → AMBER, else → GREEN

**Step B4 — Write current snapshot to history:**
```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
RECORD_ID="<value from Step 0>"
STATUS="<GREEN/AMBER/RED from B3 classification>"
tdx query --database ai_usage \
  "INSERT INTO dq_metric_history (run_id, table_fqn, check_type, column_name, metric_value, status) VALUES ('$RECORD_ID', '<DB>.<TABLE>', 'row_count', 'row_count', <CURRENT_COUNT>, '$STATUS')" \
  2>&1
```

> **Note:** `<CURRENT_COUNT>` must be the actual integer from Step B1 — substitute it before running.

---

### Check C — ID Stitching Coverage

Only run if a `cdp_unification_*` database exists AND the parent segment output DB is in scope.

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01

# Use CANONICAL_ID_COL discovered in Step 2 — do NOT hardcode 'cdp_customer_id'
tdx query --database <OUTPUT_DB> \
  "SELECT COUNT(*) AS total,
   COUNT(<CANONICAL_ID_COL>) AS with_canonical_id,
   ROUND(COUNT(<CANONICAL_ID_COL>) * 100.0 / COUNT(*), 2) AS coverage_pct
   FROM <MASTER_TABLE>" 2>&1
```

Also query the ID graph (if `cdp_unification_*` DB found):
```bash
tdx query --database <cdp_unification_db> \
  "SELECT COUNT(DISTINCT leader_id) AS canonical_groups,
   COUNT(*) AS total_ids,
   ROUND((COUNT(*) - COUNT(DISTINCT leader_id)) * 100.0 / COUNT(*), 2) AS merge_rate_pct,
   MAX(cnt) AS largest_cluster
   FROM (SELECT leader_id, COUNT(*) AS cnt FROM <canonical_id>_id_graph GROUP BY leader_id)" 2>&1
```

Classify against stitching thresholds. Write to history:
```bash
tdx query --database ai_usage \
  "INSERT INTO dq_metric_history (run_id, table_fqn, check_type, column_name, metric_value, status) VALUES ('$RECORD_ID', '<OUTPUT_DB>.<MASTER_TABLE>', 'stitching_coverage', '<CANONICAL_ID_COL>', <coverage_pct>, '<STATUS>')" \
  2>&1
```

---

### Check D — Duplicate Rate

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01

# Use primaryKeyCol discovered in Step 2
tdx query --database <DB> \
  "SELECT COUNT(*) AS total_rows,
   COUNT(*) - COUNT(DISTINCT <PRIMARY_KEY_COL>) AS duplicate_count,
   ROUND((COUNT(*) - COUNT(DISTINCT <PRIMARY_KEY_COL>)) * 100.0 / COUNT(*), 2) AS duplicate_rate_pct
   FROM <TABLE>" 2>&1
```

If no primary key was identified in Step 2: skip this check and record status = SKIP.

Write to history:
```bash
tdx query --database ai_usage \
  "INSERT INTO dq_metric_history (run_id, table_fqn, check_type, column_name, metric_value, status) VALUES ('$RECORD_ID', '<DB>.<TABLE>', 'duplicate_rate', '<PRIMARY_KEY_COL>', <duplicate_rate_pct>, '<STATUS>')" \
  2>&1
```

---

### Classify and Build ANOMALIES List

For each check result assign GREEN/AMBER/RED/SKIP. Build `ANOMALIES` list — every RED or AMBER:

```js
{
  table:       "<db>.<table>",
  checkType:   "fill_rate | row_count | stitching_coverage | duplicate_rate",
  column:      "<column name or 'row_count'>",
  actualValue: "72.3%",
  threshold:   "80%",
  severity:    "RED",
  lastHealthy: "<result from history query below>"
}
```

**For `lastHealthy`** — query `dq_metric_history` for each anomalous check:
```bash
tdx query --database ai_usage \
  "SELECT td_time_string(time,'s!','UTC') as run_date, metric_value
   FROM dq_metric_history
   WHERE table_fqn = '<db>.<table>'
     AND check_type = '<check_type>'
     AND column_name = '<column>'
     AND status = 'GREEN'
   ORDER BY time DESC LIMIT 1" 2>&1
```

If a row is returned: `lastHealthy = "Last GREEN: <run_date> (value: <metric_value>)"`.
If no rows: `lastHealthy = "Unknown — no prior GREEN run recorded"`.

---

## Step 4 — Write Quality Flags for uc-data-lineage

Write RED anomalies to `ai_usage.data_quality_flags` so `uc-data-lineage` can surface ⚠️ badges.
Include `run_id` so stale flags from previous runs can be distinguished.

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
RECORD_ID="<value from Step 0>"

tdx api -X POST /v3/table/create/ai_usage/data_quality_flags/log 2>/dev/null || true
tdx api -X POST \
  --data '{"schema":"[[\"run_id\",\"string\"],[\"table_name\",\"string\"],[\"issue_type\",\"string\"],[\"severity\",\"string\"],[\"actual_value\",\"string\"],[\"threshold\",\"string\"]]"}' \
  /v3/table/update-schema/ai_usage/data_quality_flags 2>/dev/null || true
```

For each RED anomaly:
```bash
tdx query --database ai_usage \
  "INSERT INTO data_quality_flags (run_id, table_name, issue_type, severity, actual_value, threshold) VALUES ('$RECORD_ID', '<table>', '<check_type>', 'HIGH', '<actual>', '<threshold>')" \
  2>&1
```

> **uc-data-lineage integration note:** `uc-data-lineage` Step 6 queries `ai_usage.data_quality_flags`
> filtered by `severity IN ('HIGH','CRITICAL')` and the latest `run_id`. The `run_id` column
> ensures only current-run flags surface as ⚠️ badges — not stale flags from previous runs.

---

## Step 5 — Generate the HTML Health Dashboard

Build the complete `monitorData` object from Steps 3–4, then write the HTML file.

Output filename: `dq_monitor_<segment_name_sanitised>_<YYYYMMDD>.html`

**Step 5a — Build the data object in Python and inject into the HTML template:**

```bash
python3 << 'PYEOF'
import json, datetime, re

# ── Populate ALL values from real query results above ──────────────────────
# Do NOT leave placeholder strings — substitute every <value> with real data
SEGMENT_NAME = "<TARGET_SEGMENT>"   # substitute
RUN_DATE     = datetime.datetime.utcnow().strftime("%Y-%m-%d %H:%M UTC")
RECORD_ID    = "<value from Step 0>"  # substitute

monitor_data = {
  "segmentName": SEGMENT_NAME,
  "runDate":     RUN_DATE,
  "runId":       RECORD_ID,
  "databases":   ["<list of monitored databases>"],  # substitute
  "thresholds": {
    "fillRate":    {"red": 80,  "amber": 95},
    "rowCountDrop": {"red": 10, "amber": 5},
    "rowCountSpike": {"red": 50, "amber": 20},
    "stitching":   {"red": 70,  "amber": 90},
    "duplicates":  {"red": 1.0, "amber": 0.5}
  },
  "tables": [
    # One entry per table — example structure, populate from real results:
    {
      "db": "<db>", "table": "<table>", "layer": "PARENT_SEGMENT",
      "rowCount": 0,            # from Check B Step B1 — substitute integer
      "rowCountHistory": [      # from Check B Step B3 — up to 7 entries
        # {"run_date": "2026-07-03 10:00 UTC", "metric_value": 253941, "status": "GREEN"}
      ],
      "rowCountChangePct": 0,   # computed from history
      "rowCountStatus": "GREEN",
      "fillRates": [
        # {"column": "email", "fillRatePct": 98.2, "status": "GREEN"}
      ],
      "stitchingCoveragePct": None,  # None if not applicable
      "stitchingStatus": "SKIP",
      "mergeRatePct": None,
      "largestCluster": None,
      "duplicateRatePct": 0,
      "duplicateStatus": "GREEN",
      "overallStatus": "GREEN",  # worst of all checks for this table
    }
  ],
  "anomalies": [
    # One entry per RED or AMBER result — example:
    # {
    #   "table": "cdp_audience_1087307.customers",
    #   "checkType": "fill_rate",
    #   "column": "phone",
    #   "actualValue": "72.3%",
    #   "threshold": "80%",
    #   "severity": "RED",
    #   "lastHealthy": "Unknown — no prior GREEN run recorded"
    # }
  ],
  "summary": {
    "totalTables":   0,   # derive: len(tables)
    "green":         0,   # derive: count tables where overallStatus == GREEN
    "amber":         0,
    "red":           0,
    "totalAnomalies": 0,  # derive: len(anomalies)
    "isFirstRun":    True # True if rowCountHistory was empty for all tables
  }
}

# Derive summary values — do NOT leave as zeros
tables = monitor_data["tables"]
monitor_data["summary"]["totalTables"]    = len(tables)
monitor_data["summary"]["green"]          = sum(1 for t in tables if t["overallStatus"] == "GREEN")
monitor_data["summary"]["amber"]          = sum(1 for t in tables if t["overallStatus"] == "AMBER")
monitor_data["summary"]["red"]            = sum(1 for t in tables if t["overallStatus"] == "RED")
monitor_data["summary"]["totalAnomalies"] = len(monitor_data["anomalies"])
monitor_data["summary"]["isFirstRun"]     = all(len(t.get("rowCountHistory", [])) == 0 for t in tables)

json_str = json.dumps(monitor_data, ensure_ascii=False)

# ── HTML template — __MONITOR_DATA_JSON__ is the injection point ──────────
html_template = r"""<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>CDP Data Quality Monitor</title>
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
<style>
*,*::before,*::after{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif;background:#f6f7ff;color:#1e1e2e;font-size:14px}
:root{--primary:#494FFF;--sec1:#8753FF;--lime:#9DCC4C;--yellow:#D9E47C;--red:#EE2328;--salmon:#F69068;
      --green:#16a34a;--amber:#d97706}
.header{background:linear-gradient(135deg,rgba(73,79,255,.07),rgba(135,83,255,.04));
        border-bottom:1px solid rgba(73,79,255,.15);padding:20px 28px}
.header h1{font-size:20px;font-weight:800;color:var(--primary)}
.header .sub{font-size:12px;color:#6b7280;margin-top:4px}
.tabs{display:flex;border-bottom:1px solid #e5e7eb;padding:0 28px;background:#fff;overflow-x:auto}
.tab-btn{padding:12px 18px;font-size:13px;font-weight:500;border:none;background:none;
         cursor:pointer;color:#6b7280;border-bottom:2px solid transparent;white-space:nowrap;transition:.15s}
.tab-btn:hover{color:#374151}.tab-btn.active{color:var(--primary);border-bottom-color:var(--primary)}
.content{padding:24px 28px}.tab-panel{display:none}.tab-panel.active{display:block}
.kpi-row{display:grid;grid-template-columns:repeat(5,1fr);gap:12px;margin-bottom:24px}
@media(max-width:900px){.kpi-row{grid-template-columns:repeat(3,1fr)}}
.kpi{background:#fff;border:1px solid #e5e7eb;border-radius:12px;padding:14px 16px}
.kpi-label{font-size:10px;font-weight:700;text-transform:uppercase;letter-spacing:.08em;color:#9ca3af;margin-bottom:4px}
.kpi-value{font-size:24px;font-weight:800}
.badge{display:inline-block;font-size:10px;font-weight:700;padding:2px 8px;border-radius:20px;color:#fff}
.badge-green{background:var(--green)}.badge-amber{background:var(--amber)}
.badge-red{background:var(--red)}.badge-skip{background:#9ca3af}
.section-label{font-size:10px;font-weight:700;text-transform:uppercase;letter-spacing:.08em;color:#9ca3af;margin-bottom:10px}
.card{background:#fff;border:1px solid #e5e7eb;border-radius:12px;padding:16px;margin-bottom:12px}
.card-header{display:flex;align-items:center;gap:8px;margin-bottom:12px;flex-wrap:wrap}
.card-title{font-size:13px;font-weight:700}.card-db{font-size:11px;color:#6b7280;margin-left:auto}
.metric-table{width:100%;border-collapse:collapse;font-size:12px}
.metric-table th{text-align:left;color:#9ca3af;padding:4px 8px 4px 0;font-weight:600;font-size:11px}
.metric-table td{padding:6px 8px 6px 0;border-top:1px solid #f3f4f6}
.fill-bar-bg{background:#f3f4f6;border-radius:4px;height:8px;width:80px;display:inline-block;vertical-align:middle;margin-right:6px}
.fill-bar-fg{height:8px;border-radius:4px;display:inline-block}
.anomaly-card{border-radius:8px;padding:12px;margin-bottom:8px}
.anomaly-red{border:1px solid #fca5a5;background:#fff5f5}
.anomaly-amber{border:1px solid #fcd34d;background:#fffbeb}
.all-green{text-align:center;padding:48px;color:var(--green);font-size:15px;font-weight:600}
.first-run-banner{background:#eff6ff;border:1px solid #93c5fd;border-radius:8px;padding:12px 16px;
                  font-size:13px;color:#1d4ed8;margin-bottom:20px}
.chart-row{display:grid;grid-template-columns:1fr 1fr;gap:16px;margin-top:20px}
@media(max-width:700px){.chart-row{grid-template-columns:1fr}}
.chart-card{background:#fff;border:1px solid #e5e7eb;border-radius:12px;padding:16px}
.chart-card h3{font-size:11px;font-weight:700;color:#374151;margin-bottom:12px;
               text-transform:uppercase;letter-spacing:.06em}
.chart-wrap{position:relative;height:200px}
.layer-dot{width:8px;height:8px;border-radius:50%;display:inline-block;margin-right:4px;vertical-align:middle}
.trend-mini{width:80px;height:28px;display:inline-block;vertical-align:middle}
</style>
</head>
<body>
<div class="header">
  <h1 id="h-title">CDP Data Quality Monitor</h1>
  <div class="sub" id="h-sub"></div>
</div>
<div class="tabs">
  <button class="tab-btn active" onclick="showTab('overview')">Overview</button>
  <button class="tab-btn"        onclick="showTab('tables')">Table Health</button>
  <button class="tab-btn"        onclick="showTab('anomalies')">Anomalies</button>
  <button class="tab-btn"        onclick="showTab('stitching')">ID Stitching</button>
</div>
<div class="content">
  <div id="tab-overview"  class="tab-panel active"></div>
  <div id="tab-tables"    class="tab-panel"></div>
  <div id="tab-anomalies" class="tab-panel"></div>
  <div id="tab-stitching" class="tab-panel"></div>
</div>
<script>
const D = __MONITOR_DATA_JSON__;
const STATUS_COLOR={GREEN:"#16a34a",AMBER:"#d97706",RED:"#EE2328",SKIP:"#9ca3af"};
const C={primary:"#494FFF",sec1:"#8753FF",lime:"#9DCC4C",yellow:"#D9E47C",red:"#EE2328",salmon:"#F69068"};
const LAYER_COLOR={RAW:"#f97316",STAGING:"#6366f1",GOLDEN:"#a855f7",PARENT_SEGMENT:"#f59e0b"};

function badge(s){
  const cls={GREEN:"badge-green",AMBER:"badge-amber",RED:"badge-red",SKIP:"badge-skip"}[s]||"badge-skip";
  const lbl={GREEN:"✅ GREEN",AMBER:"⚠️ AMBER",RED:"🔴 RED",SKIP:"⚪ N/A"};
  return `<span class="badge ${cls}">${lbl[s]||s}</span>`;
}
function esc(s){return String(s||"").replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;")}
function fmt(n){return n!=null?Number(n).toLocaleString():"—"}
function fillBar(pct,status){
  const c=STATUS_COLOR[status]||"#6b7280";
  return `<div class="fill-bar-bg"><div class="fill-bar-fg" style="width:${Math.min(pct,100)}%;background:${c}"></div></div>`;
}
function showTab(id){
  document.querySelectorAll(".tab-panel").forEach(p=>p.classList.remove("active"));
  document.querySelectorAll(".tab-btn").forEach(b=>b.classList.remove("active"));
  document.getElementById("tab-"+id).classList.add("active");
  ["overview","tables","anomalies","stitching"].forEach((l,i)=>{
    if(l===id) document.querySelectorAll(".tab-btn")[i].classList.add("active");
  });
}

// Mini sparkline for trend (last 7 runs) using canvas
function sparkline(history, canvasId){
  const canvas = document.getElementById(canvasId);
  if(!canvas || !history || history.length < 2) return;
  const vals = history.slice().reverse().map(h=>h.metric_value);
  new Chart(canvas,{
    type:"line",
    data:{
      labels: history.slice().reverse().map(h=>h.run_date.slice(0,10)),
      datasets:[{data:vals,borderColor:C.primary,borderWidth:2,pointRadius:2,
                 backgroundColor:"rgba(73,79,255,0.08)",fill:true,tension:0.3}]
    },
    options:{responsive:true,maintainAspectRatio:false,plugins:{legend:{display:false},tooltip:{enabled:false}},
             scales:{x:{display:false},y:{display:false}}}
  });
}

function buildOverview(){
  const s = D.summary;
  document.getElementById("h-title").textContent = `CDP Data Quality Monitor — ${D.segmentName}`;
  document.getElementById("h-sub").textContent =
    `${(D.databases||[]).join(", ")} · Run: ${D.runDate} · ${s.totalTables} tables monitored`;

  const panel = document.getElementById("tab-overview");
  const firstRunBanner = s.isFirstRun
    ? `<div class="first-run-banner">ℹ️ First run — row count and metric baselines established. Trend data (last 7 runs) will be available from the next run.</div>` : "";

  panel.innerHTML = `
    ${firstRunBanner}
    <div class="kpi-row">
      <div class="kpi"><div class="kpi-label">Tables</div><div class="kpi-value" style="color:${C.primary}">${fmt(s.totalTables)}</div></div>
      <div class="kpi"><div class="kpi-label">Healthy</div><div class="kpi-value" style="color:#16a34a">${fmt(s.green)}</div></div>
      <div class="kpi"><div class="kpi-label">Warning</div><div class="kpi-value" style="color:#d97706">${fmt(s.amber)}</div></div>
      <div class="kpi"><div class="kpi-label">Critical</div><div class="kpi-value" style="color:#EE2328">${fmt(s.red)}</div></div>
      <div class="kpi"><div class="kpi-label">Anomalies</div><div class="kpi-value" style="color:${s.totalAnomalies>0?"#EE2328":C.primary}">${fmt(s.totalAnomalies)}</div></div>
    </div>
    ${s.totalAnomalies===0?'<div class="all-green">✅ All metrics are healthy — no issues found.</div>':""}
    <div class="chart-row">
      <div class="chart-card"><h3>Health Distribution</h3><div class="chart-wrap"><canvas id="chart-health"></canvas></div></div>
      <div class="chart-card"><h3>Anomalies by Check Type</h3><div class="chart-wrap"><canvas id="chart-checks"></canvas></div></div>
    </div>`;

  new Chart(document.getElementById("chart-health"),{
    type:"doughnut",
    data:{labels:["Healthy","Warning","Critical"],
          datasets:[{data:[s.green,s.amber,s.red],backgroundColor:["#16a34a","#d97706","#EE2328"],borderWidth:0}]},
    options:{responsive:true,maintainAspectRatio:false,plugins:{legend:{position:"right",labels:{font:{size:11}}}}}
  });
  const cc={};
  (D.anomalies||[]).forEach(a=>{cc[a.checkType]=(cc[a.checkType]||0)+1;});
  new Chart(document.getElementById("chart-checks"),{
    type:"bar",
    data:{labels:Object.keys(cc),datasets:[{data:Object.values(cc),
          backgroundColor:[C.primary,C.sec1,"#a855f7","#10b981"],borderWidth:0}]},
    options:{responsive:true,maintainAspectRatio:false,plugins:{legend:{display:false}},
             scales:{y:{ticks:{font:{size:10}}},x:{ticks:{font:{size:10}}}}}
  });
}

function buildTables(){
  const panel = document.getElementById("tab-tables");
  if(!D.tables||!D.tables.length){panel.innerHTML='<div style="color:#9ca3af">No tables in scope.</div>';return;}
  panel.innerHTML = D.tables.map((t,ti) => {
    const lc = LAYER_COLOR[t.layer]||"#6b7280";
    const hist = t.rowCountHistory||[];
    const prev = hist.length ? hist[0].metric_value : null;
    const chgPct = prev ? ((t.rowCount-prev)/prev*100).toFixed(1) : null;
    const chgColor = chgPct===null?"#9ca3af":Math.abs(chgPct)>10?"#EE2328":Math.abs(chgPct)>5?"#d97706":"#16a34a";
    const chgHtml = chgPct!==null
      ? `<span style="color:${chgColor}">${chgPct>0?"+":""}${chgPct}% vs last run</span>`
      : `<span style="color:#9ca3af">First run — baseline set</span>`;
    const sparkId = `spark-${ti}`;
    const fillRows = (t.fillRates||[]).map(fr=>
      `<tr>
        <td style="font-family:monospace">${esc(fr.column)}</td>
        <td>${fr.fillRatePct!=null?fr.fillRatePct+"%":"—"}</td>
        <td>${fr.fillRatePct!=null?fillBar(fr.fillRatePct,fr.status):""}</td>
        <td>${badge(fr.status)}</td>
       </tr>`
    ).join("");
    return `<div class="card" style="border-left:4px solid ${lc}">
      <div class="card-header">
        <span class="layer-dot" style="background:${lc}"></span>
        <span class="card-title">${esc(t.table)}</span>
        ${badge(t.overallStatus)}
        <span class="card-db">${esc(t.db)}</span>
      </div>
      <div style="display:grid;grid-template-columns:1fr 1fr 1fr 1fr;gap:12px;margin-bottom:12px;align-items:start">
        <div>
          <div style="font-size:10px;color:#9ca3af;font-weight:700;text-transform:uppercase">Row Count</div>
          <div style="font-size:18px;font-weight:800">${fmt(t.rowCount)}</div>
          <div style="font-size:11px">${chgHtml}</div>
        </div>
        <div>
          <div style="font-size:10px;color:#9ca3af;font-weight:700;text-transform:uppercase">Trend (7 runs)</div>
          <canvas id="${sparkId}" class="trend-mini" style="width:80px;height:28px"></canvas>
          ${hist.length<2?'<div style="font-size:10px;color:#9ca3af">Insufficient history</div>':""}
        </div>
        <div>
          <div style="font-size:10px;color:#9ca3af;font-weight:700;text-transform:uppercase">ID Stitching</div>
          <div style="font-size:18px;font-weight:800">${t.stitchingCoveragePct!=null?t.stitchingCoveragePct+"%":"N/A"}</div>
          ${t.stitchingCoveragePct!=null?badge(t.stitchingStatus):""}
        </div>
        <div>
          <div style="font-size:10px;color:#9ca3af;font-weight:700;text-transform:uppercase">Dup Rate</div>
          <div style="font-size:18px;font-weight:800">${t.duplicateRatePct!=null?t.duplicateRatePct+"%":"N/A"}</div>
          ${t.duplicateStatus!=="SKIP"?badge(t.duplicateStatus):""}
        </div>
      </div>
      ${fillRows.length?`<table class="metric-table">
        <thead><tr><th>Column</th><th>Fill Rate</th><th>Bar</th><th>Status</th></tr></thead>
        <tbody>${fillRows}</tbody>
      </table>`:'<div style="font-size:12px;color:#9ca3af">No key columns identified.</div>'}
    </div>`;
  }).join("");

  // Render sparklines after DOM is built
  D.tables.forEach((t,ti)=>sparkline(t.rowCountHistory||[], `spark-${ti}`));
}

function buildAnomalies(){
  const panel = document.getElementById("tab-anomalies");
  const anomalies = D.anomalies||[];
  if(!anomalies.length){
    panel.innerHTML='<div class="all-green">✅ No anomalies detected — all metrics within thresholds.</div>';
    return;
  }
  const red   = anomalies.filter(a=>a.severity==="RED");
  const amber = anomalies.filter(a=>a.severity==="AMBER");
  const renderGroup = (items, cls, titleColor) => items.map(a=>
    `<div class="anomaly-card ${cls}">
      <div style="font-weight:700;font-size:13px;color:${titleColor}">${esc(a.table)} — ${esc(a.column||a.checkType)}</div>
      <div style="font-size:11px;color:#6b7280;margin-top:2px">
        Check: <strong>${esc(a.checkType)}</strong> · Actual: <strong>${esc(a.actualValue)}</strong> · Threshold: ${esc(a.threshold)}
      </div>
      <div style="font-size:11px;color:#2563eb;margin-top:4px">Last healthy: ${esc(a.lastHealthy)}</div>
    </div>`
  ).join("");

  panel.innerHTML = `
    <div class="section-label">Critical — ${red.length} issue${red.length!==1?"s":""}</div>
    ${red.length?renderGroup(red,"anomaly-red","#dc2626"):'<div style="color:#16a34a;font-size:13px;margin-bottom:16px">No critical issues.</div>'}
    <div class="section-label" style="margin-top:20px">Warnings — ${amber.length} issue${amber.length!==1?"s":""}</div>
    ${amber.length?renderGroup(amber,"anomaly-amber","#d97706"):'<div style="color:#16a34a;font-size:13px">No warnings.</div>'}`;
}

function buildStitching(){
  const panel = document.getElementById("tab-stitching");
  const stitchTables = (D.tables||[]).filter(t=>t.stitchingCoveragePct!=null);
  if(!stitchTables.length){
    panel.innerHTML='<div style="color:#9ca3af;padding:20px">No ID stitching data — no cdp_unification_* database found.</div>';
    return;
  }
  panel.innerHTML = `
    <div class="chart-card" style="margin-bottom:16px">
      <h3>ID Stitching Coverage by Table</h3>
      <div class="chart-wrap" style="height:250px"><canvas id="chart-stitch"></canvas></div>
    </div>
    ${stitchTables.map(t=>`<div class="card">
      <div class="card-header">
        <span class="card-title">${esc(t.table)}</span>
        ${badge(t.stitchingStatus)}
        <span class="card-db">${esc(t.db)}</span>
      </div>
      <div style="font-size:13px">
        Coverage: <strong>${t.stitchingCoveragePct}%</strong> of records have a canonical ID
        ${t.mergeRatePct!=null?` · Merge rate: <strong>${t.mergeRatePct}%</strong>`:""}
        ${t.largestCluster!=null?` · Largest cluster: <strong style="color:${t.largestCluster>100?"#EE2328":"inherit"}">${fmt(t.largestCluster)}</strong>`:""}
      </div>
    </div>`).join("")}`;

  new Chart(document.getElementById("chart-stitch"),{
    type:"bar",
    data:{labels:stitchTables.map(t=>t.table),
          datasets:[{label:"Coverage %",data:stitchTables.map(t=>t.stitchingCoveragePct),
                     backgroundColor:stitchTables.map(t=>
                       t.stitchingCoveragePct>=90?"#16a34a":t.stitchingCoveragePct>=70?"#d97706":"#EE2328"),
                     borderWidth:0}]},
    options:{responsive:true,maintainAspectRatio:false,plugins:{legend:{display:false}},
             scales:{y:{min:0,max:100,ticks:{callback:v=>v+"%",font:{size:10}}},x:{ticks:{font:{size:10}}}}}
  });
}

(function init(){
  buildOverview();
  buildTables();
  buildAnomalies();
  buildStitching();
})();
</script>
</body>
</html>"""

# Inject data into template
html_out = html_template.replace("__MONITOR_DATA_JSON__", json_str, 1)

# Build filename
slug = re.sub(r"[^a-z0-9_]", "_", SEGMENT_NAME.lower())[:32]
date = datetime.datetime.utcnow().strftime("%Y%m%d")
filename = f"dq_monitor_{slug}_{date}.html"

with open(filename, "w") as f:
    f.write(html_out)
print(f"✅ Dashboard written: {filename} ({len(html_out):,} bytes)")
print(f"   Open in browser: open {filename}")
PYEOF
```

---

## Step 6 — Show Your Work

Output a plain-text summary after the dashboard:

```
=== CDP DATA QUALITY MONITOR ===
Segment: <TARGET_SEGMENT>
Run:     <timestamp> | ID: <RECORD_ID>
Databases: <list>
Tables checked: <N>

HEALTH SCORECARD:
  <db>.<table>  [GREEN|AMBER|RED]
    Fill rate <col>:     98.2%  ✅ GREEN
    Fill rate <col>:     71.3%  🔴 RED — below 80% threshold
    Row count:           9,726  ✅ GREEN (+0.3% vs last run)
    Row count trend:     ↗ stable (7-run history available)
    Duplicate rate:      0.2%   ✅ GREEN
    ID stitching:        89.7%  🟡 AMBER — below 90% threshold

ANOMALIES: <N> total (<N> critical, <N> warnings)
  🔴 <table>.<col>: fill_rate 71.3% < 80% — last healthy: <date or Unknown>
  🟡 <table>:       stitching 89.7% < 90% — last healthy: <date or Unknown>

Metric history written to: ai_usage.dq_metric_history
Quality flags written to:  ai_usage.data_quality_flags (run_id: <RECORD_ID>)
```

---

## Acceptance Criteria Checklist

Before saying "done", verify every item:

- [ ] Works against any TD database — no hardcoded table names or column names
- [ ] Canonical ID column discovered dynamically in Step 2, not hardcoded as `cdp_customer_id`
- [ ] User confirmed scope before checks ran
- [ ] Thresholds are the user's values (or documented defaults)
- [ ] Each anomaly shows: specific value, threshold, and last healthy timestamp
- [ ] Row count trend over last 7 runs displayed (sparkline in Table Health tab)
- [ ] "First run — baseline established" shown if no history exists
- [ ] Row count **spikes** flagged (>50% RED, >20% AMBER) — not just drops
- [ ] Dashboard renders in "all green" state with "No issues found" message
- [ ] `ai_usage.dq_metric_history` written with per-check per-run results
- [ ] `ai_usage.data_quality_flags` written with `run_id` for stale-flag prevention
- [ ] Usage logged to `ai_usage.skills_usage_tracker`
- [ ] HTML file written with `<segment>_<YYYYMMDD>` filename, path reported to user
- [ ] User told to open HTML in external browser (not Treasure Work file viewer)

---

## Verified CLI Syntax (from ARCH-1335 — tested on eu01)

```bash
tdx tables "<db_name>.*" --json        # list tables — quoted glob, with --json
tdx describe "<db>.<table>"            # schema — NOT information_schema.columns
tdx query --database <db> "<SQL>"      # run query
tdx ps desc -o <file> "<name>"         # -o flag MUST come before segment name
tdx ps list                            # list parent segments
tdx wf projects                        # list workflows
```
