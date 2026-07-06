---
name: uc-data-lineage
description: |
  Customer-facing skill for Data Operations pilot users (ARCH-1335).
  Visualises full pipeline lineage from raw CDP landing tables through staging,
  golden layer, ID unification, and into a parent segment — as an interactive self-contained HTML dashboard (Chart.js + D3.js CDN — no React required). Answers "where does this column come from?" without tracing
  SQL by hand. Extends the id-unification-lineage patterns for the unification
  layer.
  Triggers on: "data lineage", "pipeline lineage", "where does this column come from",
  "trace this field", "what feeds the parent segment", "show me the data flow",
  "lineage dashboard", "raw to parent segment", "uc-data-lineage", "column origin",
  "pipeline map", "full lineage".
  Team: Data Ops. JIRA: ARCH-1335. POC Framework Phase: Weeks 4–6.
---

# uc-data-lineage — Full Pipeline Lineage Dashboard

Visualise how data flows from raw landing tables in the CDP through staging,
golden layer, ID unification, and into a parent segment. Generates a fully
self-contained interactive HTML file (Chart.js + D3.js via CDN — no install needed).
Each lineage edge **must cite the schema query or SQL that proved the connection** — never infer silently.

---

## ⚠️ Marketplace Skill Availability Notice

Before starting, note these gaps:

| Skill Referenced by Ticket | Status in Treasure AI Studio | Resolution |
|---|---|---|
| `id-unification-lineage` | ❌ NOT in marketplace (local custom skill) | Step 5d replicates its unification graph-query patterns inline. When/if added to marketplace, replace Step 5d with a direct call. |
| `uc-data-quality-monitor` | ❌ NOT in marketplace (sibling POC skill) | Step 6 queries its expected output table if present; skips silently if not. |
| `aps_claude_tools:data-lineage` | ✅ Available | Used for full orchestration workflow SQL parsing in Step 4. |
| `tdx-skills:tdx-basic` | ✅ Available | Used throughout for all `tdx` CLI operations. |
| `tdx-skills:parent-segment-analysis` | ✅ Available | Used in Step 2 for master table discovery. |
| `treasure-work-skills:react-dashboard` | ❌ NOT available in TAS | Step 7 generates a self-contained HTML file instead (Chart.js + D3.js CDN). |
| `sql-skills:trino` | ✅ Available | Used throughout for TD Trino SQL patterns. |

---

## Step 0 — Log Invocation

**THIS STEP IS MANDATORY. Execute it completely and confirm it succeeded before moving to Step 1. Do NOT merge this step with any other step.**

### 0a — Request TD credentials (separate tool call)

Call `mcp__tas__request_credential` with credential name `td_api_production_eu01`.
Wait for the green "Authenticated" confirmation before proceeding.
Do not run any bash until this succeeds.

### 0b — Log to tracking table (separate bash call — do NOT combine with Step 1 or Step 2)

> **Treasure AI Studio note:** `skill-usage-tracker` hooks do NOT work in TAS. Run the inline
> bash below directly. Do NOT call `skill-usage-tracker` as a Skill tool in TAS.

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
  "INSERT INTO skills_usage_tracker (id, user_id, account_id, skill_name) VALUES ('$RECORD_ID', '$USER_ID', '$ACCOUNT_ID', 'uc-data-lineage')" \
  2>&1

echo "RECORD_ID=$RECORD_ID"
echo "✅ Step 0 complete — uc-data-lineage invocation logged for $USER_ID (record: $RECORD_ID)"
```

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
  "INSERT INTO skills_usage_tracker (id, user_id, account_id, skill_name) VALUES ('$RECORD_ID', '$USER_ID', '$ACCOUNT_ID', 'uc-data-lineage')" \
  2>&1

echo "RECORD_ID=$RECORD_ID"
echo "✅ Step 0 complete — uc-data-lineage invocation logged for $USER_ID (record: $RECORD_ID)"
```

**After this bash block completes:**
- Copy the `RECORD_ID=<value>` from the output — you will need it in Steps 2–7
- Output this exact line as your next assistant message before doing anything else:
  `✅ Step 0 complete — uc-data-lineage invocation logged for <USER_ID> (record: <RECORD_ID>)`

> **Deviation from ticket:** ARCH-1335 specifies `{customer_slug}_ai_poc_tracking.skill_usage`.
> This skill uses `ai_usage.skills_usage_tracker` because `customer_slug` is not available
> at runtime in TAS.

- **Do NOT proceed to Step 1 until you have output that confirmation line.**

---

## Step 1 — Scope the Lineage

Ask the user exactly these two questions before proceeding:

1. **Which parent segment should be traced?** (provide the exact name or ID shown in the UI)
2. **Which output columns to trace?** (optional — if omitted, all columns in the master table will be traced)

Wait for the user to answer. Store the answers as `TARGET_SEGMENT` and `TARGET_COLUMNS`.

---

## Step 2 — Discover the Parent Segment and Its Output Database

Use `tdx-skills:parent-segment-analysis` patterns for this step.

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01

# IMPORTANT: -o flag must come BEFORE the segment name
tdx ps desc -o ps_schema.json "<TARGET_SEGMENT>"
```

The command writes a JSON file `ps_schema.json` with this structure:
```json
{
  "database": "cdp_audience_<id>",
  "parent_segment": "<name>",
  "parent_id": "<id>",
  "customers": {
    "table": "customers",
    "columns": [{ "name": "...", "type": "..." }]
  },
  "behaviors": [{ "table": "behavior_<name>", "columns": [...] }]
}
```

Read the file immediately after writing:
```bash
cat ps_schema.json
```

Extract and store:
- `OUTPUT_DB` = `cdp_audience_<parent_id>`
- `MASTER_TABLE` = `customers`
- `ALL_COLUMNS` = all column names from `customers.columns`
- `BEHAVIOR_TABLES` = list of behavior table names

If `TARGET_COLUMNS` was specified by the user, filter `ALL_COLUMNS` to that subset.
If `tdx ps desc` fails, fall back to: `tdx describe <OUTPUT_DB>.customers --json`

---

## Step 3 — Discover All Pipeline Layers

List all databases and classify them into pipeline layers.

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
tdx databases --json
```

Classify each database name against these patterns:

| Layer | DB Naming Pattern | Table Naming Pattern | Display Color |
|---|---|---|---|
| **RAW** | `*_raw`, `raw_*`, `*_landing` | any | `#f97316` |
| **STAGING** | `stg_*`, `staging_*`, `*_stg` | any | `#6366f1` |
| **GOLDEN** | `gld_*`, `golden_*`, `*_gld`, `*_golden` | any | `#a855f7` |
| **UNIFICATION** | starts with `cdp_unification_` | `<canonical_id>_id_graph`, `<canonical_id>_id_keys`, `<canonical_id>_id_tables`, `<canonical_id>_id_source_key_stats` | `#10b981` |
| **PARENT SEGMENT** | `cdp_audience_<id>` | `customers`, `behavior_*` | `#f59e0b` |

> **Unification layer note:** The database prefix is `cdp_unification_<sub>`. Inside it, tables follow the pattern `<canonical_id>_*` (e.g., `contact_id_id_graph`, `account_id_id_keys`). The ticket refers to `canonical_id_*` as the **table-level** naming convention within the unification database — not a separate database name.

For each identified layer database, collect table counts and row counts:

```bash
# List tables in a database
tdx tables "<db_name>.*" --json

# Row count per table (run for each table)
tdx query --database <db_name> "SELECT COUNT(*) as row_count FROM <table_name>"
```

Flag any expected layer (RAW, STAGING, GOLDEN, UNIFICATION) with **zero matching databases** — show it as "Layer Not Found" in the dashboard rather than silently omitting it.

---

## Step 4 — Discover Orchestration Workflow SQL (Primary Evidence Source)

Workflow SQL is the **primary source of truth** for transformation evidence. Schema similarity alone is not evidence.

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
# List all workflow projects
tdx wf projects 2>&1
```

Look for projects related to the customer's pipeline (match against identified DB prefixes or parent segment name). If found, **invoke the `aps_claude_tools:data-lineage` skill** to handle full workflow SQL parsing and lineage graph construction — this skill is available in the marketplace and covers orchestration workflow download, SQL file reading, and column-level graph construction. Call it explicitly:

> `Skill: aps_claude_tools:data-lineage` → provide the workflow project name and ask it to build the lineage graph for the identified databases.

If `aps_claude_tools:data-lineage` is unavailable, fall back to manual workflow pull:

```bash
# Pull the workflow — -y flag is NOT supported, use --yes
tdx wf pull <workflow_project> --yes
```

Read **every** `.dig` and `.sql` file completely. Extract:
- `td>` operators mapping SQL files to target tables
- Column renames: `<expr> AS <alias>`
- Transformations: `CAST(...)`, `LOWER(...)`, `TRIM(...)`, `COALESCE(...)`, `REGEXP_*(...)`
- Join conditions between tables
- From `unify.yml` if present: `merge_by_keys` section → unification key definitions

If no relevant workflow is found, state clearly to the user:
> **"⚠️ No orchestration workflow found. Transformation details will be inferred from schema only — all lineage edges will be marked SCHEMA-INFERRED. Pull the orchestration workflow project for SQL-confirmed lineage."**

---

## Step 5 — Trace Field Origins (Column-Level Lineage)

For each column in `TARGET_COLUMNS` (or all `ALL_COLUMNS` if not specified):

### 5a — Identify the Golden Layer Source

Find which golden layer table and column feeds this parent segment attribute.

```bash
# Describe each golden layer table to find the column
tdx describe <golden_db>.<table_name> --json
```

> **Do NOT use `information_schema.columns`** — it is not available in TD Trino. Use `tdx describe` for all schema inspection.

Check all tables in all identified `gld_*` / `golden_*` databases. If workflow SQL is available, parse the parent segment's attribute SQL expression to identify the exact golden source table and column directly.

### 5b — Trace Golden → Staging

For each golden column found in 5a, identify its staging layer origin.

If workflow SQL available: parse the golden SQL file for `SELECT ... FROM <stg_db>.<table>` — find the exact line that selects this column.
If not: describe all staging tables and match column names.

```bash
tdx describe <stg_db>.<table_name> --json
```

### 5c — Trace Staging → Raw

For each staging column found in 5b, identify its raw layer origin.

If workflow SQL available: parse the staging SQL file for the source table reference and any transformation applied to this column.
If not: describe raw tables and match column names (exact match first, then fuzzy).

```bash
tdx describe <raw_db>.<table_name> --json
```

### 5d — Trace Through Unification Layer (when applicable)

> **Note:** This step replicates the patterns from `id-unification-lineage` (local skill, not in marketplace). When that skill is added to the Treasure AI Studio marketplace, replace this step with a direct call to it.

If the parent segment uses identity resolution (i.e., a `cdp_unification_*` database exists), trace how IDs are merged:

```bash
# Find the unification database
tdx databases --json  # look for cdp_unification_<sub>

# List tables inside it
tdx tables "cdp_unification_<sub>.*" --json
```

Query the canonical ID graph stats (these are the `id-unification-lineage` graph queries):
```sql
-- Canonical group stats
SELECT COUNT(DISTINCT leader_id) AS canonical_groups,
       COUNT(*) AS total_ids
FROM cdp_unification_<sub>.<canonical_id>_id_graph

-- Merge rate
SELECT ROUND(
  (COUNT(*) - COUNT(DISTINCT leader_id)) * 100.0 / COUNT(*), 2
) AS merge_rate_pct
FROM cdp_unification_<sub>.<canonical_id>_id_graph

-- Largest clusters (over-stitching check)
SELECT leader_id, COUNT(*) AS member_count
FROM cdp_unification_<sub>.<canonical_id>_id_graph
GROUP BY leader_id
ORDER BY member_count DESC
LIMIT 10

-- Key coverage (check both table names — actual name varies by unification version)
-- First check which exists:  tdx tables "cdp_unification_<sub>.*" --json | grep key_stats
SELECT * FROM cdp_unification_<sub>.<canonical_id>_id_result_key_stats
-- If above fails, try: SELECT * FROM cdp_unification_<sub>.<canonical_id>_id_source_key_stats
```

Record: `canonical_groups`, `total_ids`, `merge_rate_pct`, `largest_cluster_size`.

### 5e — Assign Confidence Level to Each Edge

| Confidence | Meaning | Visual |
|---|---|---|
| `CONFIRMED` | Direct SQL evidence from workflow file | ✅ Green |
| `LIKELY` | Column name match across layers, no SQL | ⚠️ Amber |
| `POSSIBLE` | Weak schema match, name differs slightly | ⚠️ Orange |
| `UNRESOLVED` | No traceable origin found | ❌ Red |

### 5f — Flag Unresolved Columns

If any column's origin cannot be traced back to a raw table:
- Mark it `UNRESOLVED`
- Record: last layer where it was found, all databases checked, all tables scanned
- **Never silently omit** — every column must appear in the output, resolved or not

---

## Step 6 — Integrate with Data Quality Monitor (if available)

> `uc-data-quality-monitor` is not yet in the Treasure AI Studio marketplace. This step is conditional.

Check whether the quality flags table exists in `ai_usage` (written by `uc-data-quality-monitor`):

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01
tdx tables "ai_usage.*" --json 2>&1 | grep "data_quality_flags" || echo "NOT FOUND"
```

If found, query only the latest run's flags (use MAX run_id to avoid stale flags from previous runs):

```sql
SELECT table_name, issue_type, severity, actual_value, threshold
FROM ai_usage.data_quality_flags
WHERE run_id = (SELECT MAX(run_id) FROM ai_usage.data_quality_flags)
  AND severity IN ('HIGH', 'CRITICAL')
```

Attach the results to the relevant layer nodes in the dashboard as ⚠️ badges.
If the table does not exist, skip silently — do not show an error to the user.

---

## Step 7 — Generate the Interactive HTML Dashboard

Build the complete `lineageData` object from all steps above, then generate a
**self-contained HTML file** and write it to disk. All numbers must come from
real queries — no placeholder values.

> **Why HTML, not React:** Treasure AI Studio does not have access to
> `mcp__work__render_react`. The output must be a standalone `.html` file
> that works in any browser with no server or build step.
> CDNs used: Chart.js, D3.js, Plotly — all loaded from CDN, no installation needed.

### 7a — Populate the Data Object

After completing Steps 1–6, assemble all collected data into a single JavaScript object.
**Compute the `kpis` values — do not leave them as zero.**

```js
// All values come from real queries — no placeholders
const lineageData = {
  parentSegment: {
    name:        "<TARGET_SEGMENT>",           // from Step 1
    id:          "<parent_id>",                // from tdx ps list
    outputDb:    "cdp_audience_<id>",
    columnCount: 0,   // columnLineage.length
    rowCount:    0,   // SELECT COUNT(*) on customers table
  },
  layers: [
    // One entry per layer — include ALL 5, set found:false if not discovered
    { id:"RAW",            db:"<raw_db>",              found:true,  totalRows:0, hasQualityFlag:false,
      tables:[{ name:"<table>", rowCount:0, keyColumns:["email","id","..."] }] },
    { id:"STAGING",        db:"<stg_db>",              found:true,  totalRows:0, hasQualityFlag:false,
      tables:[{ name:"<table>", rowCount:0, keyColumns:["cdp_customer_id","..."] }] },
    { id:"GOLDEN",         db:"<gld_db>",              found:true,  totalRows:0, hasQualityFlag:false,
      tables:[{ name:"<table>", rowCount:0, keyColumns:["cdp_customer_id","..."] }] },
    { id:"UNIFICATION",    db:"cdp_unification_<sub>", found:true,
      tables:[],  canonicalGroups:0, totalIds:0, mergeRatePct:0, largestCluster:0 },
    { id:"PARENT_SEGMENT", db:"cdp_audience_<id>",     found:true,  totalRows:0,
      tables:[{ name:"customers", rowCount:0, keyColumns:["cdp_customer_id","email","..."] }] },
  ],
  columnLineage: [
    // One entry per column — example structure:
    {
      column:"email", resolved:true,
      hops:[
        { layer:"RAW",            db:"<raw_db>",  table:"contacts",    column:"email_address",  transform:"pass-through",           confidence:"CONFIRMED", evidence:"stg_contacts.sql:14" },
        { layer:"STAGING",        db:"<stg_db>",  table:"stg_contacts",column:"trfmd_email",    transform:"LOWER(TRIM(...))",        confidence:"CONFIRMED", evidence:"stg_contacts.sql:18" },
        { layer:"GOLDEN",         db:"<gld_db>",  table:"gld_master",  column:"email",          transform:"pass-through",           confidence:"CONFIRMED", evidence:"gld_master.sql:8"    },
        { layer:"PARENT_SEGMENT", db:"cdp_audience_<id>", table:"customers", column:"email",   transform:"attribute join",         confidence:"CONFIRMED", evidence:"ps attribute def"    },
      ]
    },
    // ... one entry per column
  ],
  gaps: [
    // One entry per UNRESOLVED column
    { column:"loyalty_score", lastLayerFound:"GOLDEN",
      checkedDbs:["stg_contacts","stg_transactions"],
      reason:"Found in golden layer but no staging source",
      recommendation:"Pull orchestration workflow for SQL evidence" },
  ],
  workflowEvidence: {
    projectName:   "<project or null>",
    sqlFilesRead:  0,
    digFilesRead:  0,
    evidenceLevel: "CONFIRMED",   // or "SCHEMA-INFERRED"
  },
  // DERIVE these — do not leave as zero
  kpis: {
    totalColumns:    0,   // columnLineage.length
    fullyTraced:     0,   // columnLineage.filter(c=>c.resolved).length
    confirmedEdges:  0,   // all hops with confidence==="CONFIRMED"
    schemaOnly:      0,   // hops with confidence LIKELY or POSSIBLE
    unresolved:      0,   // gaps.length
    layersPresent:   0,   // layers.filter(l=>l.found).length
  },
};
```

### 7b — Generate and Write the HTML File

Once `lineageData` is fully populated, write the complete HTML file to disk.
Output filename: `lineage_<parent_segment_name_sanitised>.html`
(lowercase, spaces→underscores, strip special chars)

```bash
cat > lineage_<segment_name>.html << 'HTMLEOF'
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Pipeline Lineage — <PARENT_SEGMENT_NAME></title>
<!-- CDN libraries — no install needed -->
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/d3@7.9.0/dist/d3.min.js"></script>
<style>
  /* ── Reset & base ───────────────────────────────────────── */
  *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
  body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
         background: #f6f7ff; color: #1e1e2e; font-size: 14px; }

  /* ── Brand colors ───────────────────────────────────────── */
  :root {
    --primary:  #494FFF;
    --sec1:     #8753FF;
    --sec2:     #C466D4;
    --lime:     #9DCC4C;
    --yellow:   #D9E47C;
    --red:      #EE2328;
    --salmon:   #F69068;
    --raw:      #f97316;
    --stg:      #6366f1;
    --gld:      #a855f7;
    --unif:     #10b981;
    --ps:       #f59e0b;
  }

  /* ── Header ─────────────────────────────────────────────── */
  .header {
    background: linear-gradient(135deg, rgba(73,79,255,0.07), rgba(135,83,255,0.04));
    border-bottom: 1px solid rgba(73,79,255,0.15);
    padding: 20px 28px;
  }
  .header h1 { font-size: 20px; font-weight: 800; color: var(--primary); }
  .header .sub { font-size: 12px; color: #6b7280; margin-top: 4px; }

  /* ── Tab bar ─────────────────────────────────────────────── */
  .tabs { display: flex; border-bottom: 1px solid #e5e7eb;
          padding: 0 28px; background: #fff; overflow-x: auto; }
  .tab-btn { padding: 12px 18px; font-size: 13px; font-weight: 500;
             border: none; background: none; cursor: pointer; color: #6b7280;
             border-bottom: 2px solid transparent; white-space: nowrap; transition: .15s; }
  .tab-btn:hover { color: #374151; }
  .tab-btn.active { color: var(--primary); border-bottom-color: var(--primary); }

  /* ── Content area ────────────────────────────────────────── */
  .content { padding: 24px 28px; }
  .tab-panel { display: none; }
  .tab-panel.active { display: block; }

  /* ── KPI row ─────────────────────────────────────────────── */
  .kpi-row { display: grid; grid-template-columns: repeat(6,1fr); gap: 12px; margin-bottom: 24px; }
  @media(max-width:900px){ .kpi-row { grid-template-columns: repeat(3,1fr); } }
  .kpi { background: #fff; border: 1px solid #e5e7eb; border-radius: 12px;
         padding: 14px 16px; }
  .kpi-label { font-size: 10px; font-weight: 700; text-transform: uppercase;
               letter-spacing: .08em; color: #9ca3af; margin-bottom: 4px; }
  .kpi-value { font-size: 24px; font-weight: 800; }
  .kpi-sub   { font-size: 11px; color: #6b7280; margin-top: 2px; }

  /* ── Alert banner ────────────────────────────────────────── */
  .banner-warn { background: #fffbeb; border: 1px solid #fcd34d;
                 border-radius: 8px; padding: 12px 16px;
                 font-size: 13px; color: #92400e; margin-bottom: 20px; }

  /* ── Pipeline flow ───────────────────────────────────────── */
  .flow-label { font-size: 10px; font-weight: 700; text-transform: uppercase;
                letter-spacing: .08em; color: #9ca3af; margin-bottom: 12px; }
  .pipeline-flow { display: flex; align-items: stretch; gap: 0;
                   overflow-x: auto; padding-bottom: 8px; }
  .layer-node { flex: 1; min-width: 150px; border-radius: 12px; border: 2px solid;
                padding: 12px; position: relative; }
  .layer-node.missing { opacity: .4; border-style: dashed; }
  .layer-id   { font-size: 9px; font-weight: 700; text-transform: uppercase;
                letter-spacing: .08em; margin-bottom: 4px; }
  .layer-db   { font-size: 12px; font-weight: 600; color: #1e1e2e;
                white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .layer-meta { font-size: 11px; color: #6b7280; margin-top: 2px; }
  .layer-unif-meta { font-size: 10px; color: #6b7280; margin-top: 2px; }
  .col-pills  { display: flex; flex-wrap: wrap; gap: 3px; margin-top: 8px; }
  .col-pill   { font-size: 9px; padding: 2px 6px; border-radius: 4px;
                border: 1px solid; cursor: pointer; transition: opacity .15s; }
  .col-pill:hover { opacity: .7; }
  .quality-flag { font-size: 10px; font-weight: 600; color: #d97706; margin-top: 6px; }
  .flow-arrow { display: flex; flex-direction: column; align-items: center;
                justify-content: center; width: 28px; flex-shrink: 0; }
  .flow-arrow-line { width: 2px; flex: 1; }
  .flow-arrow-head { font-size: 14px; }

  /* ── Confidence legend ───────────────────────────────────── */
  .conf-legend { display: flex; flex-wrap: wrap; gap: 12px;
                 font-size: 12px; margin-top: 16px; }
  .conf-dot { width: 10px; height: 10px; border-radius: 50%;
              display: inline-block; margin-right: 4px; vertical-align: middle; }

  /* ── Confidence badge ────────────────────────────────────── */
  .badge { display: inline-block; font-size: 9px; font-weight: 700;
           padding: 2px 7px; border-radius: 20px; color: #fff; }

  /* ── Explorer ────────────────────────────────────────────── */
  .explorer-controls { display: flex; align-items: center; gap: 12px; margin-bottom: 16px; }
  .explorer-controls label { font-size: 13px; font-weight: 500; color: #374151; }
  .explorer-controls select { font-size: 13px; border: 1px solid #d1d5db;
                               border-radius: 8px; padding: 6px 12px;
                               background: #fff; color: #1e1e2e; min-width: 200px; }
  .explorer-hint { font-size: 13px; color: #9ca3af; }
  .explorer-col-header { display: flex; align-items: center; gap: 8px; margin-bottom: 12px; }
  .explorer-col-name { font-size: 15px; font-weight: 700; }

  /* ── Hop chain ───────────────────────────────────────────── */
  .hop-chain { display: flex; align-items: flex-start; gap: 0;
               overflow-x: auto; padding-bottom: 8px; }
  .hop-card  { min-width: 180px; max-width: 220px; border-radius: 12px;
               border: 2px solid; padding: 12px; }
  .hop-layer { font-size: 9px; font-weight: 700; text-transform: uppercase;
               letter-spacing: .08em; }
  .hop-table { font-size: 10px; color: #6b7280; white-space: nowrap;
               overflow: hidden; text-overflow: ellipsis; margin-top: 2px; }
  .hop-col   { font-size: 13px; font-weight: 600; margin-top: 4px; }
  .hop-transform { font-size: 10px; font-family: monospace; color: #6b7280;
                   background: #f3f4f6; border-radius: 4px; padding: 4px 6px;
                   margin-top: 6px; word-break: break-all; }
  .hop-evidence-toggle { font-size: 9px; text-decoration: underline;
                         color: #9ca3af; cursor: pointer; background: none;
                         border: none; margin-top: 6px; display: block; }
  .hop-evidence-toggle:hover { color: #374151; }
  .hop-evidence { font-size: 10px; font-family: monospace; color: #6b7280;
                  background: #f3f4f6; border: 1px solid #e5e7eb;
                  border-radius: 4px; padding: 6px; margin-top: 4px;
                  word-break: break-all; display: none; }
  .hop-arrow { display: flex; align-items: center; padding: 0 4px;
               align-self: center; font-size: 16px; }

  /* ── Layers tab ──────────────────────────────────────────── */
  .layer-card { background: #fff; border: 1px solid #e5e7eb;
                border-radius: 12px; padding: 16px; margin-bottom: 12px; }
  .layer-card-header { display: flex; align-items: center; gap: 8px;
                       margin-bottom: 12px; }
  .layer-dot { width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0; }
  .layer-card-title { font-size: 13px; font-weight: 700; }
  .layer-card-db { font-size: 12px; color: #6b7280; margin-left: auto; }
  .layer-table { width: 100%; border-collapse: collapse; font-size: 12px; }
  .layer-table th { text-align: left; color: #9ca3af; padding: 4px 12px 4px 0;
                    font-weight: 600; font-size: 11px; }
  .layer-table td { padding: 6px 12px 6px 0; border-top: 1px solid #f3f4f6; }
  .layer-table td:nth-child(2) { text-align: right; color: #6b7280; }
  .key-col-tag { font-size: 9px; padding: 2px 6px; border-radius: 4px; }
  .unif-stats { display: grid; grid-template-columns: repeat(4,1fr);
                gap: 8px; margin-top: 12px; }

  /* ── Gaps tab ────────────────────────────────────────────── */
  .section-label { font-size: 10px; font-weight: 700; text-transform: uppercase;
                   letter-spacing: .08em; color: #9ca3af; margin-bottom: 10px; }
  .gap-card { border: 1px solid #fca5a5; background: #fff5f5; border-radius: 8px;
              padding: 12px; margin-bottom: 8px; }
  .gap-col  { font-size: 13px; font-weight: 700; color: #dc2626; }
  .gap-meta { font-size: 11px; color: #6b7280; margin-top: 2px; }
  .gap-reason { font-size: 12px; color: #374151; margin-top: 4px; }
  .gap-tip  { font-size: 11px; color: #2563eb; margin-top: 4px; }
  .schema-table { width: 100%; border-collapse: collapse; font-size: 12px;
                  background: #fff; border: 1px solid #e5e7eb; border-radius: 8px;
                  overflow: hidden; }
  .schema-table th { background: rgba(73,79,255,.06); padding: 8px 12px;
                     text-align: left; font-weight: 700; font-size: 11px; }
  .schema-table td { padding: 7px 12px; border-top: 1px solid #f3f4f6; }
  .schema-table tr:nth-child(even) td { background: #fafafa; }
  .schema-tip { font-size: 11px; color: #d97706; margin-top: 8px; }

  /* ── Chart containers ────────────────────────────────────── */
  .chart-row { display: grid; grid-template-columns: 1fr 1fr; gap: 16px;
               margin-top: 20px; }
  @media(max-width:700px){ .chart-row { grid-template-columns: 1fr; } }
  .chart-card { background: #fff; border: 1px solid #e5e7eb;
                border-radius: 12px; padding: 16px; }
  .chart-card h3 { font-size: 12px; font-weight: 700; color: #374151;
                   margin-bottom: 12px; text-transform: uppercase;
                   letter-spacing: .06em; }
  .chart-wrap { position: relative; height: 200px; }

  /* ── D3 lineage SVG ──────────────────────────────────────── */
  #d3-lineage { width: 100%; overflow-x: auto; }
  #d3-lineage svg { display: block; }
  .d3-node rect { rx: 8; ry: 8; }
  .d3-link { fill: none; stroke-width: 2; }
  .d3-label { font-size: 11px; font-family: -apple-system, sans-serif; }
</style>
</head>
<body>

<!-- HEADER -->
<div class="header">
  <h1 id="h-title">Pipeline Lineage Dashboard</h1>
  <div class="sub" id="h-sub"></div>
</div>

<!-- TABS -->
<div class="tabs">
  <button class="tab-btn active" onclick="showTab('overview')">Pipeline Overview</button>
  <button class="tab-btn"        onclick="showTab('explorer')">Field Lineage Explorer</button>
  <button class="tab-btn"        onclick="showTab('layers')">Layer Detail</button>
  <button class="tab-btn"        onclick="showTab('gaps')">Lineage Gaps &amp; Risks</button>
</div>

<div class="content">

  <!-- TAB: Overview -->
  <div id="tab-overview" class="tab-panel active">
    <div class="kpi-row" id="kpi-row"></div>
    <div id="schema-inferred-banner" class="banner-warn" style="display:none"></div>
    <div class="flow-label">Pipeline Flow — click any column to trace its lineage</div>
    <div class="pipeline-flow" id="pipeline-flow"></div>
    <div class="conf-legend" id="conf-legend"></div>
    <div class="chart-row">
      <div class="chart-card">
        <h3>Confidence Distribution</h3>
        <div class="chart-wrap"><canvas id="chart-confidence"></canvas></div>
      </div>
      <div class="chart-card">
        <h3>Rows per Layer</h3>
        <div class="chart-wrap"><canvas id="chart-rows"></canvas></div>
      </div>
    </div>
  </div>

  <!-- TAB: Explorer -->
  <div id="tab-explorer" class="tab-panel">
    <div class="explorer-controls">
      <label>Select column:</label>
      <select id="col-select" onchange="selectColumn(this.value)">
        <option value="">— choose a column —</option>
      </select>
    </div>
    <div id="explorer-hint" class="explorer-hint">
      Select a column above or click a column pill in the Pipeline Overview tab.
    </div>
    <div id="explorer-trace" style="display:none">
      <div class="explorer-col-header">
        <span class="explorer-col-name" id="trace-col-name"></span>
        <span id="trace-status"></span>
      </div>
      <div class="hop-chain" id="hop-chain"></div>
    </div>
    <!-- D3 full lineage graph for selected column -->
    <div id="d3-lineage" style="margin-top:24px"></div>
  </div>

  <!-- TAB: Layers -->
  <div id="tab-layers" class="tab-panel">
    <div id="layers-content"></div>
  </div>

  <!-- TAB: Gaps -->
  <div id="tab-gaps" class="tab-panel">
    <div class="section-label" id="gaps-unresolved-label"></div>
    <div id="gaps-unresolved"></div>
    <div id="schema-only-section" style="display:none; margin-top:24px">
      <div class="section-label" id="schema-only-label"></div>
      <table class="schema-table" id="schema-only-table">
        <thead><tr>
          <th>Column</th><th>Weakest Confidence</th><th>Layer</th>
        </tr></thead>
        <tbody id="schema-only-body"></tbody>
      </table>
      <div class="schema-tip">💡 Pull orchestration workflow SQL to confirm these edges.</div>
    </div>
  </div>

</div><!-- /content -->

<script>
// ══════════════════════════════════════════════════════════════════════════
// DATA — replace this object with real collected values from Steps 1–6
// ══════════════════════════════════════════════════════════════════════════
const lineageData = __LINEAGE_DATA_JSON__;
// ══════════════════════════════════════════════════════════════════════════

// ── Constants ─────────────────────────────────────────────────────────────
const C = {
  primary:"#494FFF", sec1:"#8753FF", sec2:"#C466D4",
  lime:"#9DCC4C", yellow:"#D9E47C", red:"#EE2328",
  darkRed:"#9E363E", salmon:"#F69068",
};
const LAYER_COLOR = {
  RAW:"#f97316", STAGING:"#6366f1", GOLDEN:"#a855f7",
  UNIFICATION:"#10b981", PARENT_SEGMENT:"#f59e0b",
};
const CONF_STYLE = {
  CONFIRMED:  { bg:C.lime,    label:"✅ CONFIRMED"  },
  LIKELY:     { bg:C.yellow,  label:"⚠️ LIKELY"     },
  POSSIBLE:   { bg:C.salmon,  label:"⚠️ POSSIBLE"   },
  UNRESOLVED: { bg:C.red,     label:"❌ UNRESOLVED"  },
};
const CONF_ORDER = ["UNRESOLVED","POSSIBLE","LIKELY","CONFIRMED"];

// ── Helpers ───────────────────────────────────────────────────────────────
function badge(level) {
  const s = CONF_STYLE[level] || { bg:"#6b7280", label:level };
  return `<span class="badge" style="background:${s.bg}">${s.label}</span>`;
}
function fmt(n) { return n != null ? Number(n).toLocaleString() : "—"; }
function worstConf(hops) {
  return hops.reduce((w, h) => {
    return CONF_ORDER.indexOf(h.confidence) < CONF_ORDER.indexOf(w) ? h.confidence : w;
  }, "CONFIRMED");
}
function sanitise(s) { return (s||"").replace(/[^a-z0-9_]/gi,"_").toLowerCase(); }

// ── Tab switching ─────────────────────────────────────────────────────────
function showTab(id) {
  document.querySelectorAll(".tab-panel").forEach(p => p.classList.remove("active"));
  document.querySelectorAll(".tab-btn").forEach(b => b.classList.remove("active"));
  document.getElementById("tab-" + id).classList.add("active");
  const btns = document.querySelectorAll(".tab-btn");
  const labels = ["overview","explorer","layers","gaps"];
  btns[labels.indexOf(id)].classList.add("active");
  if (id === "explorer") renderD3(window._selectedColumn);
}

// ── Column selection (from dropdown or pill click) ────────────────────────
function selectColumn(col) {
  window._selectedColumn = col;
  document.getElementById("col-select").value = col;
  const trace = (lineageData.columnLineage || []).find(c => c.column === col);
  const hint  = document.getElementById("explorer-hint");
  const panel = document.getElementById("explorer-trace");
  if (!col || !trace) { hint.style.display=""; panel.style.display="none"; return; }
  hint.style.display = "none"; panel.style.display = "";
  document.getElementById("trace-col-name").textContent = trace.column;
  document.getElementById("trace-status").innerHTML = trace.resolved
    ? '<span style="color:#16a34a;font-size:12px;font-weight:600">✅ Fully Traced</span>'
    : '<span style="color:#dc2626;font-size:12px;font-weight:600">❌ Unresolved</span>';

  // Build hop chain
  const chain = document.getElementById("hop-chain");
  chain.innerHTML = "";
  trace.hops.forEach((hop, i) => {
    const col = LAYER_COLOR[hop.layer] || "#6b7280";
    const evId = "ev-" + sanitise(trace.column) + "-" + i;
    const card = document.createElement("div");
    card.className = "hop-card";
    card.style.borderColor = col;
    card.style.background  = col + "0d";
    card.innerHTML = `
      <div class="hop-layer" style="color:${col}">${hop.layer}</div>
      <div class="hop-table">${hop.db}.${hop.table}</div>
      <div class="hop-col">${hop.column}</div>
      ${hop.transform !== "pass-through"
        ? `<div class="hop-transform">${escHtml(hop.transform)}</div>` : ""}
      <div style="margin-top:6px">${badge(hop.confidence)}</div>
      <button class="hop-evidence-toggle" onclick="toggleEv('${evId}',this)">Show evidence</button>
      <div class="hop-evidence" id="${evId}">${escHtml(hop.evidence)}</div>`;
    chain.appendChild(card);
    if (i < trace.hops.length - 1) {
      const arr = document.createElement("div");
      arr.className = "hop-arrow";
      arr.style.color = (CONF_STYLE[trace.hops[i+1].confidence]||{}).bg || "#6b7280";
      arr.textContent = "→";
      chain.appendChild(arr);
    }
  });
  renderD3(col);
}

function toggleEv(id, btn) {
  const el = document.getElementById(id);
  const shown = el.style.display === "block";
  el.style.display = shown ? "none" : "block";
  btn.textContent  = shown ? "Show evidence" : "Hide evidence";
}

function escHtml(s) {
  return String(s||"").replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;");
}

// ── D3 lineage graph for selected column ──────────────────────────────────
function renderD3(col) {
  const container = document.getElementById("d3-lineage");
  container.innerHTML = "";
  if (!col) return;
  const trace = (lineageData.columnLineage || []).find(c => c.column === col);
  if (!trace || !trace.hops.length) return;

  const W = 200, H = 100, PAD = 40;
  const totalW = trace.hops.length * W + (trace.hops.length - 1) * PAD + 60;
  const totalH = H + 80;

  const svg = d3.select(container).append("svg")
    .attr("width",  totalW)
    .attr("height", totalH);

  // Draw edges first (behind nodes)
  trace.hops.forEach((hop, i) => {
    if (i === 0) return;
    const prev = trace.hops[i - 1];
    const x1   = (i - 1) * (W + PAD) + W + 30;
    const x2   = i * (W + PAD) + 30;
    const y    = totalH / 2;
    const conf = CONF_STYLE[hop.confidence] || { bg:"#6b7280" };
    svg.append("path")
      .attr("d", `M${x1},${y} C${x1+PAD/2},${y} ${x2-PAD/2},${y} ${x2},${y}`)
      .attr("class","d3-link")
      .attr("stroke", conf.bg)
      .attr("marker-end","url(#arrowhead-" + sanitise(hop.confidence) + ")");
  });

  // Arrow markers
  ["CONFIRMED","LIKELY","POSSIBLE","UNRESOLVED"].forEach(c => {
    const conf = CONF_STYLE[c] || { bg:"#6b7280" };
    const defs = svg.append("defs");
    defs.append("marker")
      .attr("id","arrowhead-" + sanitise(c))
      .attr("viewBox","0 0 10 10").attr("refX",9).attr("refY",5)
      .attr("markerWidth",6).attr("markerHeight",6).attr("orient","auto")
      .append("path").attr("d","M 0 0 L 10 5 L 0 10 z")
      .attr("fill", conf.bg);
  });

  // Draw nodes
  trace.hops.forEach((hop, i) => {
    const color = LAYER_COLOR[hop.layer] || "#6b7280";
    const x     = i * (W + PAD) + 30;
    const y     = (totalH - H) / 2;
    const g     = svg.append("g").attr("class","d3-node");

    g.append("rect")
      .attr("x",x).attr("y",y).attr("width",W).attr("height",H)
      .attr("rx",8).attr("ry",8)
      .attr("fill", color + "14")
      .attr("stroke", color)
      .attr("stroke-width",2);

    g.append("text").attr("class","d3-label")
      .attr("x", x + W/2).attr("y", y + 18)
      .attr("text-anchor","middle")
      .attr("fill",color).attr("font-weight","700").attr("font-size","9")
      .text(hop.layer);

    g.append("text").attr("class","d3-label")
      .attr("x", x + W/2).attr("y", y + 36)
      .attr("text-anchor","middle")
      .attr("fill","#6b7280").attr("font-size","10")
      .text(hop.table);

    g.append("text").attr("class","d3-label")
      .attr("x", x + W/2).attr("y", y + 54)
      .attr("text-anchor","middle")
      .attr("fill","#1e1e2e").attr("font-weight","600").attr("font-size","12")
      .text(hop.column);

    if (hop.transform && hop.transform !== "pass-through") {
      g.append("text").attr("class","d3-label")
        .attr("x", x + W/2).attr("y", y + 72)
        .attr("text-anchor","middle")
        .attr("fill","#9ca3af").attr("font-size","9").attr("font-family","monospace")
        .text(hop.transform.length > 25
          ? hop.transform.slice(0,22)+"…" : hop.transform);
    }

    // Confidence badge circle
    const conf = CONF_STYLE[hop.confidence] || { bg:"#6b7280" };
    g.append("circle")
      .attr("cx", x + W - 10).attr("cy", y + 10)
      .attr("r", 5).attr("fill", conf.bg);
  });
}

// ── Build Overview ─────────────────────────────────────────────────────────
function buildOverview() {
  const kpi   = lineageData.kpis || {};
  const kpis  = [
    { label:"Total Columns",   value:kpi.totalColumns   || 0, color:C.primary },
    { label:"Fully Traced",    value:kpi.fullyTraced    || 0, color:C.lime    },
    { label:"Confirmed Edges", value:kpi.confirmedEdges || 0, color:C.lime    },
    { label:"Schema Only",     value:kpi.schemaOnly     || 0, color:C.yellow  },
    { label:"Unresolved",      value:kpi.unresolved     || 0, color:C.red     },
    { label:"Layers Found",    value:kpi.layersPresent  || 0, color:C.sec1    },
  ];
  document.getElementById("kpi-row").innerHTML = kpis.map(k =>
    `<div class="kpi">
       <div class="kpi-label">${k.label}</div>
       <div class="kpi-value" style="color:${k.color}">${fmt(k.value)}</div>
     </div>`
  ).join("");

  // Schema-inferred banner
  if ((lineageData.workflowEvidence||{}).evidenceLevel === "SCHEMA-INFERRED") {
    const b = document.getElementById("schema-inferred-banner");
    b.style.display = "";
    b.innerHTML = `⚠️ No orchestration workflow SQL found — all edges are SCHEMA-INFERRED.
                   Pull workflow project <strong>${lineageData.workflowEvidence.projectName || "(unknown)"}</strong>
                   for SQL-confirmed lineage.`;
  }

  // Pipeline flow
  const flow = document.getElementById("pipeline-flow");
  flow.innerHTML = "";
  (lineageData.layers || []).forEach((layer, i) => {
    const color = LAYER_COLOR[layer.id] || "#6b7280";
    const node  = document.createElement("div");
    node.className = "layer-node" + (layer.found ? "" : " missing");
    node.style.borderColor = color;
    node.style.background  = color + "0d";
    node.style.flex        = "1";
    node.style.minWidth    = "150px";

    const keyColsHtml = (layer.tables || []).slice(0,2).map(t =>
      `<div style="font-size:10px;font-weight:600;color:#6b7280;margin-top:6px">${t.name}</div>
       <div class="col-pills">${
         (t.keyColumns||[]).slice(0,4).map(col =>
           `<span class="col-pill"
              style="border-color:${color}88;color:${color};background:${color}11"
              onclick="goExplorer('${escHtml(col)}')">${escHtml(col)}</span>`
         ).join("") +
         ((t.keyColumns||[]).length > 4
           ? `<span style="font-size:9px;color:#9ca3af">+${(t.keyColumns||[]).length-4} more</span>` : "")
       }</div>`
    ).join("");

    node.innerHTML = `
      <div class="layer-id" style="color:${color}">${layer.id}</div>
      <div class="layer-db">${layer.db || "—"}</div>
      <div class="layer-meta">${(layer.tables||[]).length} tables · ${fmt(layer.totalRows)} rows</div>
      ${layer.id==="UNIFICATION" && layer.found
        ? `<div class="layer-unif-meta">${fmt(layer.canonicalGroups)} groups · ${layer.mergeRatePct||0}% merged</div>` : ""}
      ${layer.found ? keyColsHtml : '<div style="font-size:11px;color:#9ca3af;margin-top:4px">Layer not found</div>'}
      ${layer.hasQualityFlag ? '<div class="quality-flag">⚠️ Quality issues</div>' : ""}`;
    flow.appendChild(node);

    if (i < lineageData.layers.length - 1) {
      // Arrow colour = worst confidence of edges going into next layer
      const nextId = lineageData.layers[i+1]?.id;
      const hops   = (lineageData.columnLineage||[]).flatMap(c=>c.hops).filter(h=>h.layer===nextId);
      const conf   = hops.length ? worstConf(hops) : "CONFIRMED";
      const arrowColor = (CONF_STYLE[conf]||{}).bg || "#6b7280";
      const arr = document.createElement("div");
      arr.className = "flow-arrow";
      arr.innerHTML = `<div class="flow-arrow-line" style="background:${arrowColor}66"></div>
                       <div class="flow-arrow-head" style="color:${arrowColor}">▶</div>
                       <div class="flow-arrow-line" style="background:${arrowColor}66"></div>`;
      flow.appendChild(arr);
    }
  });

  // Confidence legend
  document.getElementById("conf-legend").innerHTML =
    Object.entries(CONF_STYLE).map(([k,v]) =>
      `<span><span class="conf-dot" style="background:${v.bg}"></span>${v.label}</span>`
    ).join("");

  // Charts — Confidence distribution (Chart.js doughnut)
  const confCounts = { CONFIRMED:0, LIKELY:0, POSSIBLE:0, UNRESOLVED:0 };
  (lineageData.columnLineage||[]).forEach(c =>
    (c.hops||[]).forEach(h => { if(confCounts[h.confidence]!=null) confCounts[h.confidence]++; })
  );
  new Chart(document.getElementById("chart-confidence"), {
    type: "doughnut",
    data: {
      labels: Object.keys(confCounts),
      datasets:[{ data: Object.values(confCounts),
        backgroundColor: [C.lime, C.yellow, C.salmon, C.red],
        borderWidth: 0 }]
    },
    options:{ responsive:true, maintainAspectRatio:false,
              plugins:{ legend:{ position:"right", labels:{ font:{size:11} } } } }
  });

  // Charts — Rows per layer (Chart.js bar)
  const layerLabels = (lineageData.layers||[]).filter(l=>l.found).map(l=>l.id);
  const layerRows   = (lineageData.layers||[]).filter(l=>l.found).map(l=>l.totalRows||0);
  const layerColors = (lineageData.layers||[]).filter(l=>l.found).map(l=>LAYER_COLOR[l.id]||"#6b7280");
  new Chart(document.getElementById("chart-rows"), {
    type: "bar",
    data: { labels:layerLabels,
            datasets:[{ data:layerRows, backgroundColor:layerColors.map(c=>c+"cc"),
                        borderColor:layerColors, borderWidth:1 }] },
    options:{ responsive:true, maintainAspectRatio:false, plugins:{ legend:{display:false} },
              scales:{ y:{ ticks:{ font:{size:10} } }, x:{ ticks:{ font:{size:10} } } } }
  });
}

// ── Build Explorer ─────────────────────────────────────────────────────────
function buildExplorer() {
  const sel = document.getElementById("col-select");
  (lineageData.columnLineage||[]).forEach(c => {
    const opt = document.createElement("option");
    opt.value = c.column; opt.textContent = c.column;
    sel.appendChild(opt);
  });
}

// Navigate from pipeline flow pill click → explorer tab
function goExplorer(col) {
  showTab("explorer");
  selectColumn(col);
}

// ── Build Layers tab ──────────────────────────────────────────────────────
function buildLayers() {
  const container = document.getElementById("layers-content");
  container.innerHTML = "";
  (lineageData.layers||[]).forEach(layer => {
    const color = LAYER_COLOR[layer.id] || "#6b7280";
    const card  = document.createElement("div");
    card.className = "layer-card";
    card.style.borderLeftColor = color;
    card.style.borderLeftWidth = "4px";

    const tablesHtml = !layer.found ? "<div style='font-size:12px;color:#9ca3af'>Layer not found in this account.</div>" :
      `<table class="layer-table">
         <thead><tr><th>Table</th><th>Rows</th><th>Key Columns</th></tr></thead>
         <tbody>${(layer.tables||[]).map(t =>
           `<tr>
              <td style="font-family:monospace">${escHtml(t.name)}</td>
              <td>${fmt(t.rowCount)}</td>
              <td>${(t.keyColumns||[]).map(col =>
                `<span class="key-col-tag" style="background:${color}11;color:${color}">${escHtml(col)}</span>`
              ).join(" ")}</td>
            </tr>`
         ).join("")}</tbody>
       </table>
       ${layer.id==="UNIFICATION" && layer.found ? `
         <div class="unif-stats">
           <div class="kpi"><div class="kpi-label">Canonical Groups</div><div class="kpi-value" style="color:${C.sec1}">${fmt(layer.canonicalGroups)}</div></div>
           <div class="kpi"><div class="kpi-label">Total IDs</div><div class="kpi-value" style="color:${C.sec1}">${fmt(layer.totalIds)}</div></div>
           <div class="kpi"><div class="kpi-label">Merge Rate</div><div class="kpi-value" style="color:${C.lime}">${layer.mergeRatePct||0}%</div></div>
           <div class="kpi"><div class="kpi-label">Largest Cluster</div>
             <div class="kpi-value" style="color:${(layer.largestCluster||0)>100?C.red:C.lime}">${fmt(layer.largestCluster)}</div></div>
         </div>` : ""}`;

    card.innerHTML = `
      <div class="layer-card-header">
        <span class="layer-dot" style="background:${color}"></span>
        <span class="layer-card-title" style="color:${color}">${layer.id}</span>
        ${!layer.found?'<span style="font-size:11px;color:#9ca3af">(not found)</span>':""}
        <span class="layer-card-db">${layer.db||""}</span>
      </div>
      ${tablesHtml}`;
    container.appendChild(card);
  });
}

// ── Build Gaps tab ─────────────────────────────────────────────────────────
function buildGaps() {
  const gaps = lineageData.gaps || [];
  document.getElementById("gaps-unresolved-label").textContent =
    `Unresolved Columns (${gaps.length})`;

  const div = document.getElementById("gaps-unresolved");
  if (!gaps.length) {
    div.innerHTML = '<div style="color:#16a34a;font-weight:600">✅ All columns successfully traced end-to-end.</div>';
  } else {
    div.innerHTML = gaps.map(g =>
      `<div class="gap-card">
         <div class="gap-col">${escHtml(g.column)}</div>
         <div class="gap-meta">Last traced: <strong>${escHtml(g.lastLayerFound)}</strong></div>
         <div class="gap-meta">Checked: ${(g.checkedDbs||[]).map(escHtml).join(", ")}</div>
         <div class="gap-reason">${escHtml(g.reason)}</div>
         <div class="gap-tip">💡 ${escHtml(g.recommendation)}</div>
       </div>`
    ).join("");
  }

  const schemaOnly = (lineageData.columnLineage||[]).filter(c =>
    (c.hops||[]).some(h => h.confidence==="LIKELY" || h.confidence==="POSSIBLE")
  );
  if (schemaOnly.length) {
    document.getElementById("schema-only-section").style.display = "";
    document.getElementById("schema-only-label").textContent =
      `Schema-Inferred Only — No SQL Evidence (${schemaOnly.length})`;
    document.getElementById("schema-only-body").innerHTML = schemaOnly.map((c, i) => {
      const weak = (c.hops||[]).reduce((w,h) =>
        CONF_ORDER.indexOf(h.confidence) < CONF_ORDER.indexOf(w.confidence) ? h : w
      , c.hops[0]);
      return `<tr>
        <td style="font-family:monospace">${escHtml(c.column)}</td>
        <td>${badge(weak?.confidence)}</td>
        <td style="color:#6b7280">${escHtml(weak?.layer)}</td>
      </tr>`;
    }).join("");
  }
}

// ── Init ──────────────────────────────────────────────────────────────────
(function init() {
  const ps  = lineageData.parentSegment || {};
  const ev  = lineageData.workflowEvidence || {};
  document.getElementById("h-title").textContent =
    `Pipeline Lineage — ${ps.name || ""}`;
  document.getElementById("h-sub").textContent =
    `${ps.outputDb || ""} · ${fmt(ps.rowCount)} customers · Evidence: ${ev.evidenceLevel || "—"}`;

  buildOverview();
  buildExplorer();
  buildLayers();
  buildGaps();
})();
</script>
</body>
</html>
HTMLEOF
```

### 7c — Inject the Data

Replace the `__LINEAGE_DATA_JSON__` placeholder in the HTML file with the actual JSON.
Use Python to safely serialise and inject:

```bash
python3 - << 'PYEOF'
import json, re

# Build lineageData dict from all collected query results
lineage_data = {
  "parentSegment": {
    "name":        "<TARGET_SEGMENT>",
    "id":          "<parent_id>",
    "outputDb":    "cdp_audience_<id>",
    "columnCount": 0,    # replace with real value
    "rowCount":    0,    # replace with real value
  },
  "layers": [ # populated from Steps 3-5
  ],
  "columnLineage": [ # populated from Step 5
  ],
  "gaps": [ # populated from Step 5f
  ],
  "workflowEvidence": {
    "projectName":   None,
    "sqlFilesRead":  0,
    "digFilesRead":  0,
    "evidenceLevel": "CONFIRMED",   # or "SCHEMA-INFERRED"
  },
  "kpis": {
    # Derive from arrays above
    "totalColumns":   0,
    "fullyTraced":    0,
    "confirmedEdges": 0,
    "schemaOnly":     0,
    "unresolved":     0,
    "layersPresent":  0,
  }
}

json_str = json.dumps(lineage_data, ensure_ascii=True)
json_str = json_str.replace("</script>", "<\\/script>")

with open("lineage_<segment_name>.html", "r") as f:
    html = f.read()

html = html.replace("__LINEAGE_DATA_JSON__", json_str, 1)

with open("lineage_<segment_name>.html", "w") as f:
    f.write(html)

print("✅ Dashboard written: lineage_<segment_name>.html")
PYEOF
```

### 7d — Confirm the File

```bash
ls -lh lineage_*.html
echo "Open with: open lineage_<segment_name>.html"
```

Tell the user the full path to the file so they can open it in their browser.

---

## Step 8 — Show Your Work (Evidence Summary)

After the dashboard renders, output a plain-text evidence summary so any technical reviewer can validate the lineage without opening the dashboard.

For each resolved column:
```
email
  RAW    raw_db.contacts.email_address          [pass-through]       ✅ CONFIRMED — stg_contacts.sql:14
  STG    stg_db.stg_contacts.trfmd_email        [LOWER(TRIM(...))]   ✅ CONFIRMED — stg_contacts.sql:18
  GLD    gld_db.gld_master.email                [pass-through]       ✅ CONFIRMED — gld_master.sql:8
  PS     cdp_audience_1234.customers.email      [attribute join]     ✅ CONFIRMED — attribute definition
```

For each unresolved column:
```
⚠️ UNRESOLVED: loyalty_score
   Last layer traced: GOLDEN (gld_db.gld_derived.loyalty_score found)
   Gap: GOLDEN → STAGING — no staging table contains this column
   Checked: stg_contacts, stg_transactions, stg_events
   Recommendation: Pull orchestration workflow for SQL evidence
```

---

## Acceptance Criteria Checklist

Before saying "done" to the user, verify every item:

- [ ] Dashboard renders for parent segments with ≥2 discoverable upstream layers
- [ ] At least one column traced end-to-end from raw to parent segment
- [ ] **Clicking a column in the pipeline flow diagram** highlights its lineage path in the Explorer tab (not just dropdown selection)
- [ ] Each layer node shows table name, row count, AND key columns
- [ ] Unresolved lineage flagged clearly — not silently omitted
- [ ] Every lineage edge cites evidence (SQL snippet, schema query, or explicit SCHEMA-INFERRED label)
- [ ] Unification layer shows canonical group count, merge rate, largest cluster (id-unification-lineage patterns)
- [ ] `id-unification-lineage` gap documented — Step 5d implements its patterns inline
- [ ] Data quality flags from `uc-data-quality-monitor` surfaced if available
- [ ] Usage logged to `ai_usage.skills_usage_tracker` with id, user_id, account_id, skill_name
- [ ] No fabricated statistics — all numbers from real `tdx` queries

---

## Verified CLI Syntax Reference

These were tested against eu01 and confirmed working:

```bash
# Correct: -o flag must come BEFORE segment name
tdx ps desc -o ps_schema.json "<SEGMENT_NAME>"

# Correct: list tables in a database
tdx tables "<db_name>.*" --json

# Correct: list workflow projects
tdx wf projects

# Correct: pull a workflow
tdx wf pull <project_name> --yes          # NOT -y

# Correct: tdx status fields
tdx status | grep "^User:"               # → "User: email@example.com"
tdx status | grep "^Account ID:"         # → "Account ID: 61"

# Correct: schema inspection (NOT information_schema.columns)
tdx describe "<db>.<table>" --json
```
