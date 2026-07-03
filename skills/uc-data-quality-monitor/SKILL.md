---
name: uc-data-quality-monitor
description: |
  Customer-facing skill for Data Operations pilot users (ARCH-1325).
  Monitors fill rate, completeness, and ID stitching health across the
  customer's CDP data. Flags issues proactively with a green/amber/red
  health scorecard. Governance & Operations use case — relevant for
  Data/Engineering-led POCs (SC1 + SC4 + SC5).
  Triggers on: "data quality", "health check", "data monitor", "fill rate",
  "completeness check", "id stitching health", "data health", "quality monitor",
  "uc-data-quality-monitor", "CDP health", "check my data quality",
  "are there data issues", "monitor my tables".
  Team: Governance & Ops. JIRA: ARCH-1325. POC Framework Phase: Weeks 4–6.
  NOTE: This is a lighter customer-facing monitoring view.
  For full SE-facing deep diagnostics use cdp-assessment instead.
---

# uc-data-quality-monitor — CDP Data Health Monitor

Monitor fill rate, completeness, ID stitching coverage, and deduplication health
across a customer's CDP tables. Produces a green/amber/red health scorecard as a
self-contained HTML file.

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
| `uc-data-lineage` | ✅ Available (this repo) | Step 5 writes `data_quality_flags` table consumed by `uc-data-lineage` |

---

## Step 0 — Log Invocation

**THIS STEP IS MANDATORY. Execute it completely and confirm before moving to Step 1. Do NOT merge with any other step.**

### 0a — Request TD credentials (separate tool call)

Call `mcp__tas__request_credential` with `td_api_production_eu01`.
Wait for the green "Authenticated" confirmation before proceeding.

### 0b — Log to tracking table (separate bash call)

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

### 0c — Confirm before continuing

Output:
```
✅ Step 0 complete — uc-data-quality-monitor invocation logged for <USER_ID>
```

---

## Step 1 — Scope Monitoring Targets

Ask the user:
1. **Which database(s) to monitor?** (e.g. parent segment output `cdp_audience_*`, staging `stg_*`, golden `gld_*` — or all)
2. **Custom thresholds?** (optional — if omitted use defaults below)

**Default thresholds:**
| Check | Threshold | Severity |
|---|---|---|
| Fill rate on key column | < 80% | 🔴 RED |
| Fill rate on key column | 80–95% | 🟡 AMBER |
| Row count drop vs last snapshot | > 10% drop | 🔴 RED |
| Row count drop vs last snapshot | 5–10% drop | 🟡 AMBER |
| ID stitching coverage | < 70% | 🔴 RED |
| ID stitching coverage | 70–90% | 🟡 AMBER |
| Duplicate rate by primary key | > 1% | 🔴 RED |
| Duplicate rate by primary key | 0.5–1% | 🟡 AMBER |

Wait for user answers. Store as `TARGET_DBS` and `CUSTOM_THRESHOLDS`.
If user says "all" — discover all relevant databases automatically in Step 2.

---

## Step 2 — Discover Tables in Scope

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01

# List all databases
tdx databases 2>&1
```

For each database in `TARGET_DBS`:
```bash
tdx tables "<db_name>.*" 2>&1
```

For each table discovered, collect:
- Table name, database name, layer classification (RAW/STAGING/GOLDEN/PARENT_SEGMENT)
- Identify **key columns** per table: email, phone, id columns, primary keys
  - Use `tdx describe "<db>.<table>"` to get column list
  - Key column heuristics: columns named `*_id`, `*_key`, `email*`, `phone*`, `cdp_customer_id`, `canonical_id`

Store as `TABLES_IN_SCOPE` — a list of `{db, table, layer, keyColumns[]}`.

---

## Step 3 — Run Health Checks

For each table in `TABLES_IN_SCOPE`, run all four checks. Run queries in parallel where possible.

### Check A — Fill Rate (per key column)

```sql
SELECT
  '<table>' AS table_name,
  '<column>' AS column_name,
  COUNT(*) AS total_rows,
  COUNT(<column>) AS non_null_count,
  ROUND(COUNT(<column>) * 100.0 / COUNT(*), 2) AS fill_rate_pct
FROM <db>.<table>
```

Run once per key column per table. Flag against fill rate thresholds.

### Check B — Row Count Snapshot

```sql
SELECT COUNT(*) AS row_count FROM <db>.<table>
```

Compare against previous snapshot stored in `ai_usage.dq_snapshots` (if it exists).
If no previous snapshot exists, record current count as baseline — flag as "First run — baseline established, no trend available yet."

Store current snapshot:
```sql
-- Check if dq_snapshots table exists first
-- If not, create it:
tdx api -X POST /v3/database/create/ai_usage 2>/dev/null || true
tdx api -X POST /v3/table/create/ai_usage/dq_snapshots/log 2>/dev/null || true
tdx api -X POST \
  --data '{"schema":"[[\"table_fqn\",\"string\"],[\"row_count\",\"long\"],[\"run_id\",\"string\"]]"}' \
  /v3/table/update-schema/ai_usage/dq_snapshots 2>/dev/null || true

-- Insert current snapshot
tdx query --database ai_usage \
  "INSERT INTO dq_snapshots (table_fqn, row_count, run_id) VALUES ('<db>.<table>', <count>, '<RECORD_ID>')"
```

### Check C — ID Stitching Coverage

Only run if a `cdp_unification_*` database exists for this customer.

```sql
-- What % of parent segment customers have a canonical ID
SELECT
  COUNT(*) AS total_customers,
  COUNT(cdp_customer_id) AS with_canonical_id,
  ROUND(COUNT(cdp_customer_id) * 100.0 / COUNT(*), 2) AS stitching_coverage_pct
FROM <cdp_audience_db>.customers
```

Also query the ID graph for merge rate:
```sql
SELECT
  COUNT(DISTINCT leader_id) AS canonical_groups,
  COUNT(*) AS total_ids,
  ROUND((COUNT(*) - COUNT(DISTINCT leader_id)) * 100.0 / COUNT(*), 2) AS merge_rate_pct
FROM <cdp_unification_db>.<canonical_id>_id_graph
```

### Check D — Duplicate Rate

```sql
SELECT
  COUNT(*) AS total_rows,
  COUNT(*) - COUNT(DISTINCT <primary_key>) AS duplicate_count,
  ROUND((COUNT(*) - COUNT(DISTINCT <primary_key>)) * 100.0 / COUNT(*), 2) AS duplicate_rate_pct
FROM <db>.<table>
```

Use the first identified key column as `primary_key`. If no clear primary key, skip this check and note "No primary key identified — dedup check skipped."

### Classify Each Result

For each check result, assign:
- 🟢 **GREEN** — within thresholds
- 🟡 **AMBER** — approaching threshold
- 🔴 **RED** — threshold breached
- ⚪ **SKIP** — check not applicable or insufficient data

Build `ANOMALIES` list: every RED or AMBER result with `{table, column, check_type, actual_value, threshold, severity, last_healthy}`.

For `last_healthy`: query `ai_usage.dq_snapshots` for the last run where the value was GREEN. If no history, set to "Unknown — first run."

---

## Step 4 — Write Quality Flags Table

Write a summary of RED/CRITICAL anomalies to `ai_usage.data_quality_flags` so that
`uc-data-lineage` can surface them as ⚠️ badges on affected layer nodes.

```bash
tdx api -X POST /v3/table/create/ai_usage/data_quality_flags/log 2>/dev/null || true
tdx api -X POST \
  --data '{"schema":"[[\"table_name\",\"string\"],[\"issue_type\",\"string\"],[\"severity\",\"string\"],[\"actual_value\",\"string\"],[\"threshold\",\"string\"]]"}' \
  /v3/table/update-schema/ai_usage/data_quality_flags 2>/dev/null || true
```

For each RED anomaly:
```bash
tdx query --database ai_usage \
  "INSERT INTO data_quality_flags (table_name, issue_type, severity, actual_value, threshold) VALUES ('<table>', '<check_type>', 'HIGH', '<actual>', '<threshold>')"
```

---

## Step 5 — Generate the HTML Health Dashboard

Build the complete results object from Steps 3–4, then write a self-contained HTML file.

Output filename: `dq_monitor_<db_name_sanitised>_<YYYYMMDD>.html`

```bash
python3 << 'PYEOF'
import json, datetime

# Populate from real query results above
monitor_data = {
  "runDate": datetime.datetime.utcnow().strftime("%Y-%m-%d %H:%M UTC"),
  "runId": "<RECORD_ID>",
  "databases": ["<list of monitored databases>"],
  "thresholds": {
    "fillRate":    {"red": 80,  "amber": 95},
    "rowCountDrop": {"red": 10,  "amber": 5},
    "stitching":   {"red": 70,  "amber": 90},
    "duplicates":  {"red": 1.0, "amber": 0.5}
  },
  "tables": [
    # One entry per table checked:
    {
      "db": "<db>", "table": "<table>", "layer": "STAGING",
      "rowCount": 0, "previousRowCount": 0, "rowCountChangePct": 0,
      "fillRates": [
        {"column": "email", "fillRatePct": 0, "status": "GREEN"},
        # ... one per key column
      ],
      "stitchingCoveragePct": None,  # null if not applicable
      "duplicateRatePct": 0,
      "overallStatus": "GREEN",  # worst of all checks
      "anomalies": []  # list of {checkType, actualValue, threshold, severity}
    }
  ],
  "anomalies": [
    # All RED/AMBER anomalies across all tables
    {
      "table": "<db>.<table>",
      "checkType": "fill_rate",
      "column": "email",
      "actualValue": "72.3%",
      "threshold": "80%",
      "severity": "RED",
      "lastHealthy": "Unknown — first run"
    }
  ],
  "summary": {
    "totalTables": 0,
    "green": 0, "amber": 0, "red": 0,
    "totalAnomalies": 0,
    "isFirstRun": True
  }
}

html = """<!DOCTYPE html>
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
      --green:#16a34a;--amber:#d97706;--grey:#6b7280}
.header{background:linear-gradient(135deg,rgba(73,79,255,.07),rgba(135,83,255,.04));
        border-bottom:1px solid rgba(73,79,255,.15);padding:20px 28px}
.header h1{font-size:20px;font-weight:800;color:var(--primary)}
.header .sub{font-size:12px;color:#6b7280;margin-top:4px}
.tabs{display:flex;border-bottom:1px solid #e5e7eb;padding:0 28px;background:#fff;overflow-x:auto}
.tab-btn{padding:12px 18px;font-size:13px;font-weight:500;border:none;background:none;
         cursor:pointer;color:#6b7280;border-bottom:2px solid transparent;white-space:nowrap;transition:.15s}
.tab-btn:hover{color:#374151}
.tab-btn.active{color:var(--primary);border-bottom-color:var(--primary)}
.content{padding:24px 28px}
.tab-panel{display:none}
.tab-panel.active{display:block}
.kpi-row{display:grid;grid-template-columns:repeat(5,1fr);gap:12px;margin-bottom:24px}
@media(max-width:900px){.kpi-row{grid-template-columns:repeat(3,1fr)}}
.kpi{background:#fff;border:1px solid #e5e7eb;border-radius:12px;padding:14px 16px}
.kpi-label{font-size:10px;font-weight:700;text-transform:uppercase;letter-spacing:.08em;color:#9ca3af;margin-bottom:4px}
.kpi-value{font-size:24px;font-weight:800}
.badge{display:inline-block;font-size:10px;font-weight:700;padding:2px 8px;border-radius:20px;color:#fff}
.badge-green{background:var(--green)}
.badge-amber{background:var(--amber)}
.badge-red{background:var(--red)}
.badge-skip{background:#9ca3af}
.badge-grey{background:#6b7280}
.section-label{font-size:10px;font-weight:700;text-transform:uppercase;letter-spacing:.08em;color:#9ca3af;margin-bottom:10px}
.card{background:#fff;border:1px solid #e5e7eb;border-radius:12px;padding:16px;margin-bottom:12px}
.card-header{display:flex;align-items:center;gap:8px;margin-bottom:12px}
.card-title{font-size:13px;font-weight:700}
.card-db{font-size:11px;color:#6b7280;margin-left:auto}
.metric-table{width:100%;border-collapse:collapse;font-size:12px}
.metric-table th{text-align:left;color:#9ca3af;padding:4px 8px 4px 0;font-weight:600;font-size:11px}
.metric-table td{padding:6px 8px 6px 0;border-top:1px solid #f3f4f6}
.fill-bar-bg{background:#f3f4f6;border-radius:4px;height:8px;width:100px;display:inline-block;vertical-align:middle}
.fill-bar-fg{height:8px;border-radius:4px;display:inline-block}
.anomaly-card{border-radius:8px;padding:12px;margin-bottom:8px}
.anomaly-red{border:1px solid #fca5a5;background:#fff5f5}
.anomaly-amber{border:1px solid #fcd34d;background:#fffbeb}
.anomaly-col{font-weight:700;font-size:13px}
.anomaly-meta{font-size:11px;color:#6b7280;margin-top:2px}
.anomaly-reason{font-size:12px;color:#374151;margin-top:4px}
.anomaly-tip{font-size:11px;color:#2563eb;margin-top:4px}
.all-green{text-align:center;padding:48px;color:var(--green);font-size:15px;font-weight:600}
.first-run-banner{background:#eff6ff;border:1px solid #93c5fd;border-radius:8px;padding:12px 16px;
                  font-size:13px;color:#1d4ed8;margin-bottom:20px}
.chart-row{display:grid;grid-template-columns:1fr 1fr;gap:16px;margin-top:20px}
@media(max-width:700px){.chart-row{grid-template-columns:1fr}}
.chart-card{background:#fff;border:1px solid #e5e7eb;border-radius:12px;padding:16px}
.chart-card h3{font-size:11px;font-weight:700;color:#374151;margin-bottom:12px;text-transform:uppercase;letter-spacing:.06em}
.chart-wrap{position:relative;height:200px}
.layer-dot{width:8px;height:8px;border-radius:50%;display:inline-block;margin-right:4px;vertical-align:middle}
</style>
</head>
<body>
<div class="header">
  <h1>CDP Data Quality Monitor</h1>
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

const STATUS_COLOR = {GREEN:"#16a34a",AMBER:"#d97706",RED:"#EE2328",SKIP:"#9ca3af"};
const C = {primary:"#494FFF",sec1:"#8753FF",lime:"#9DCC4C",yellow:"#D9E47C",red:"#EE2328",salmon:"#F69068"};

function badge(status){
  const cls={GREEN:"badge-green",AMBER:"badge-amber",RED:"badge-red",SKIP:"badge-skip"}[status]||"badge-grey";
  const labels={GREEN:"✅ GREEN",AMBER:"⚠️ AMBER",RED:"🔴 RED",SKIP:"⚪ N/A"};
  return `<span class="badge ${cls}">${labels[status]||status}</span>`;
}
function esc(s){return String(s||"").replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;")}
function fmt(n){return n!=null?Number(n).toLocaleString():"—"}
function fillBar(pct,status){
  const color=STATUS_COLOR[status]||"#6b7280";
  return `<div class="fill-bar-bg"><div class="fill-bar-fg" style="width:${Math.min(pct,100)}%;background:${color}"></div></div>`;
}
function showTab(id){
  document.querySelectorAll(".tab-panel").forEach(p=>p.classList.remove("active"));
  document.querySelectorAll(".tab-btn").forEach(b=>b.classList.remove("active"));
  document.getElementById("tab-"+id).classList.add("active");
  const labels=["overview","tables","anomalies","stitching"];
  document.querySelectorAll(".tab-btn")[labels.indexOf(id)].classList.add("active");
}

function buildOverview(){
  const s = D.summary;
  document.getElementById("h-sub").textContent =
    `${D.databases.join(", ")} · Run: ${D.runDate} · ${s.totalTables} tables monitored`;

  const panel = document.getElementById("tab-overview");
  const isFirstRun = s.isFirstRun ? `<div class="first-run-banner">ℹ️ First run — row count baselines established. Trend data will be available from the next run.</div>` : "";

  panel.innerHTML = `
    ${isFirstRun}
    <div class="kpi-row">
      <div class="kpi"><div class="kpi-label">Tables Monitored</div><div class="kpi-value" style="color:${C.primary}">${fmt(s.totalTables)}</div></div>
      <div class="kpi"><div class="kpi-label">Healthy</div><div class="kpi-value" style="color:#16a34a">${fmt(s.green)}</div></div>
      <div class="kpi"><div class="kpi-label">Warning</div><div class="kpi-value" style="color:#d97706">${fmt(s.amber)}</div></div>
      <div class="kpi"><div class="kpi-label">Critical</div><div class="kpi-value" style="color:#EE2328">${fmt(s.red)}</div></div>
      <div class="kpi"><div class="kpi-label">Anomalies</div><div class="kpi-value" style="color:${s.totalAnomalies>0?"#EE2328":C.primary}">${fmt(s.totalAnomalies)}</div></div>
    </div>
    ${s.totalAnomalies===0 ? '<div class="all-green">✅ All metrics are healthy — no issues found.</div>' : ""}
    <div class="chart-row">
      <div class="chart-card"><h3>Health Distribution</h3><div class="chart-wrap"><canvas id="chart-health"></canvas></div></div>
      <div class="chart-card"><h3>Anomalies by Check Type</h3><div class="chart-wrap"><canvas id="chart-checks"></canvas></div></div>
    </div>`;

  // Health doughnut
  new Chart(document.getElementById("chart-health"),{
    type:"doughnut",
    data:{labels:["Healthy","Warning","Critical"],
          datasets:[{data:[s.green,s.amber,s.red],
                     backgroundColor:["#16a34a","#d97706","#EE2328"],borderWidth:0}]},
    options:{responsive:true,maintainAspectRatio:false,plugins:{legend:{position:"right",labels:{font:{size:11}}}}}
  });

  // Anomalies by check type bar
  const checkCounts = {};
  (D.anomalies||[]).forEach(a=>{ checkCounts[a.checkType]=(checkCounts[a.checkType]||0)+1; });
  new Chart(document.getElementById("chart-checks"),{
    type:"bar",
    data:{labels:Object.keys(checkCounts),
          datasets:[{data:Object.values(checkCounts),
                     backgroundColor:[C.primary,C.sec1,"#a855f7","#10b981"],borderWidth:0}]},
    options:{responsive:true,maintainAspectRatio:false,plugins:{legend:{display:false}},
             scales:{y:{ticks:{font:{size:10}}},x:{ticks:{font:{size:10}}}}}
  });
}

function buildTables(){
  const panel = document.getElementById("tab-tables");
  if(!D.tables||!D.tables.length){panel.innerHTML='<div style="color:#9ca3af">No tables in scope.</div>';return;}
  panel.innerHTML = D.tables.map(t => {
    const layerColors={RAW:"#f97316",STAGING:"#6366f1",GOLDEN:"#a855f7",PARENT_SEGMENT:"#f59e0b"};
    const lc = layerColors[t.layer]||"#6b7280";
    const rowChangePct = t.previousRowCount
      ? ((t.rowCount - t.previousRowCount) / t.previousRowCount * 100).toFixed(1)
      : null;
    const rowChangeHtml = rowChangePct !== null
      ? `<span style="color:${Math.abs(rowChangePct)>10?"#EE2328":Math.abs(rowChangePct)>5?"#d97706":"#16a34a"}">${rowChangePct>0?"+":""}${rowChangePct}% vs last run</span>`
      : `<span style="color:#9ca3af">First run — baseline set</span>`;

    const fillRows = (t.fillRates||[]).map(fr =>
      `<tr>
        <td style="font-family:monospace">${esc(fr.column)}</td>
        <td>${fmt(fr.fillRatePct)}%</td>
        <td>${fillBar(fr.fillRatePct,fr.status)}</td>
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
      <div style="display:grid;grid-template-columns:1fr 1fr 1fr;gap:12px;margin-bottom:12px">
        <div><div style="font-size:10px;color:#9ca3af;font-weight:700;text-transform:uppercase">Row Count</div>
             <div style="font-size:18px;font-weight:800">${fmt(t.rowCount)}</div>
             <div style="font-size:11px">${rowChangeHtml}</div></div>
        <div><div style="font-size:10px;color:#9ca3af;font-weight:700;text-transform:uppercase">Stitching</div>
             <div style="font-size:18px;font-weight:800">${t.stitchingCoveragePct!=null?t.stitchingCoveragePct+"%":"N/A"}</div>
             ${t.stitchingCoveragePct!=null?badge(t.stitchingCoveragePct>=90?"GREEN":t.stitchingCoveragePct>=70?"AMBER":"RED"):""}</div>
        <div><div style="font-size:10px;color:#9ca3af;font-weight:700;text-transform:uppercase">Dup Rate</div>
             <div style="font-size:18px;font-weight:800">${t.duplicateRatePct!=null?t.duplicateRatePct+"%":"N/A"}</div>
             ${t.duplicateRatePct!=null?badge(t.duplicateRatePct<=0.5?"GREEN":t.duplicateRatePct<=1?"AMBER":"RED"):""}</div>
      </div>
      ${fillRows.length?`<table class="metric-table">
        <thead><tr><th>Column</th><th>Fill Rate</th><th>Bar</th><th>Status</th></tr></thead>
        <tbody>${fillRows}</tbody>
      </table>`:"<div style='font-size:12px;color:#9ca3af'>No key columns identified for fill rate check.</div>"}
    </div>`;
  }).join("");
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
  panel.innerHTML = `
    <div class="section-label">Critical Issues (${red.length})</div>
    ${red.map(a=>`<div class="anomaly-card anomaly-red">
      <div class="anomaly-col" style="color:#dc2626">${esc(a.table)} → ${esc(a.column||a.checkType)}</div>
      <div class="anomaly-meta">Check: ${esc(a.checkType)} · Actual: <strong>${esc(a.actualValue)}</strong> · Threshold: ${esc(a.threshold)}</div>
      <div class="anomaly-reason">${esc(a.severity==="RED"?"Breached critical threshold.":"")}</div>
      <div class="anomaly-tip">Last healthy: ${esc(a.lastHealthy)}</div>
    </div>`).join("")}
    <div class="section-label" style="margin-top:20px">Warnings (${amber.length})</div>
    ${amber.map(a=>`<div class="anomaly-card anomaly-amber">
      <div class="anomaly-col" style="color:#d97706">${esc(a.table)} → ${esc(a.column||a.checkType)}</div>
      <div class="anomaly-meta">Check: ${esc(a.checkType)} · Actual: <strong>${esc(a.actualValue)}</strong> · Threshold: ${esc(a.threshold)}</div>
      <div class="anomaly-tip">Last healthy: ${esc(a.lastHealthy)}</div>
    </div>`).join("")}`;
}

function buildStitching(){
  const panel = document.getElementById("tab-stitching");
  const stitchTables = (D.tables||[]).filter(t=>t.stitchingCoveragePct!=null);
  if(!stitchTables.length){
    panel.innerHTML='<div style="color:#9ca3af;padding:20px">No ID stitching data available — no cdp_unification_* database found for this customer.</div>';
    return;
  }
  panel.innerHTML = `
    <div class="chart-row">
      <div class="chart-card" style="grid-column:1/-1"><h3>ID Stitching Coverage by Table</h3>
        <div class="chart-wrap" style="height:250px"><canvas id="chart-stitch"></canvas></div>
      </div>
    </div>
    <div style="margin-top:20px">
      ${stitchTables.map(t=>{
        const status = t.stitchingCoveragePct>=90?"GREEN":t.stitchingCoveragePct>=70?"AMBER":"RED";
        return `<div class="card">
          <div class="card-header">
            <span class="card-title">${esc(t.table)}</span>
            ${badge(status)}
            <span class="card-db">${esc(t.db)}</span>
          </div>
          <div>Coverage: <strong>${t.stitchingCoveragePct}%</strong> of records have a canonical ID
          ${t.mergeRatePct!=null?` · Merge rate: <strong>${t.mergeRatePct}%</strong>`:""}
          ${t.largestCluster!=null?` · Largest cluster: <strong style="color:${t.largestCluster>100?"#EE2328":"inherit"}">${fmt(t.largestCluster)}</strong>`:""}
          </div>
        </div>`;
      }).join("")}
    </div>`;

  new Chart(document.getElementById("chart-stitch"),{
    type:"bar",
    data:{
      labels: stitchTables.map(t=>t.table),
      datasets:[{
        label:"Stitching Coverage %",
        data: stitchTables.map(t=>t.stitchingCoveragePct),
        backgroundColor: stitchTables.map(t=>
          t.stitchingCoveragePct>=90?"#16a34a":t.stitchingCoveragePct>=70?"#d97706":"#EE2328"),
        borderWidth:0
      }]
    },
    options:{responsive:true,maintainAspectRatio:false,
             plugins:{legend:{display:false}},
             scales:{y:{min:0,max:100,ticks:{callback:v=>v+"%",font:{size:10}}},
                     x:{ticks:{font:{size:10}}}}}
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

json_str = json.dumps(monitor_data, ensure_ascii=False)
html_out = html.replace("__MONITOR_DATA_JSON__", json_str, 1)

filename = f"dq_monitor_{datetime.datetime.utcnow().strftime('%Y%m%d')}.html"
with open(filename, "w") as f:
    f.write(html_out)
print(f"✅ Dashboard written: {filename} ({len(html_out):,} bytes)")
PYEOF
```

### Dashboard Structure

**Tab 1 — Overview**
- KPI row: Tables Monitored / Healthy / Warning / Critical / Anomalies
- "All metrics healthy" state when no issues found (ticket requirement)
- Health distribution doughnut (Chart.js)
- Anomalies by check type bar chart

**Tab 2 — Table Health**
- One card per table: layer badge, row count + trend, stitching coverage, duplicate rate
- Fill rate bar per key column with colour-coded status
- First run: shows "baseline set — trend available next run"

**Tab 3 — Anomalies**
- RED issues first, AMBER below
- Each anomaly: table, column, check type, actual value, threshold breached, last healthy timestamp
- "No anomalies" state clearly shown when all green (ticket requirement)

**Tab 4 — ID Stitching**
- Bar chart: stitching coverage % per table, colour-coded green/amber/red
- Cards: coverage %, merge rate, largest cluster (flags over-stitching risk if cluster > 100)
- "No stitching data" state if no `cdp_unification_*` database found

---

## Step 6 — Show Your Work

After the dashboard, output a plain-text summary:

```
=== DATA QUALITY MONITOR SUMMARY ===
Run: <timestamp> | Run ID: <RECORD_ID>
Databases: <list>
Tables checked: <N>

HEALTH SCORECARD:
  <db>.<table>  [GREEN/AMBER/RED]
    Fill rate email:     98.2%   ✅
    Fill rate phone:     71.3%   🔴 BELOW 80% THRESHOLD
    Row count:           9,726   ✅ (+0.3% vs last run)
    Duplicate rate:      0.2%    ✅
    ID stitching:        89.7%   🟡 BELOW 90% AMBER THRESHOLD

ANOMALIES: <N> total (<N> critical, <N> warnings)
  🔴 <table>.<column>: fill_rate 71.3% < threshold 80% (last healthy: Unknown — first run)
  🟡 <table>: stitching_coverage 89.7% < threshold 90%
```

---

## Acceptance Criteria Checklist

Before saying "done" to the user, verify every item:

- [ ] Works against any TD database — no hardcoded table names
- [ ] Thresholds used are the user's confirmed values (or defaults if not specified)
- [ ] Each anomaly shows: specific value, threshold, last healthy timestamp
- [ ] Dashboard renders in "all green" state (no issues found) with clear message
- [ ] Row count trend shown — "first run / baseline established" if no history
- [ ] ID stitching tab shows "no data" state gracefully if no unification DB found
- [ ] `ai_usage.data_quality_flags` written with RED anomalies (consumed by `uc-data-lineage`)
- [ ] `ai_usage.dq_snapshots` written with current row counts for next-run trend
- [ ] Usage logged to `ai_usage.skills_usage_tracker`
- [ ] HTML file written and path reported to user

---

## Verified CLI Syntax (inherited from ARCH-1335)

```bash
tdx tables "<db_name>.*"              # list tables
tdx describe "<db>.<table>"           # get column schema — NOT information_schema
tdx query --database <db> "<SQL>"     # run query
tdx wf projects                       # list workflows
tdx ps list                           # list parent segments
```
