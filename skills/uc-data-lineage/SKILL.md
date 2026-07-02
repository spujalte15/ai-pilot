---
name: uc-data-lineage
description: |
  Customer-facing skill for Data Operations pilot users (ARCH-1335).
  Visualises full pipeline lineage from raw CDP landing tables through staging,
  golden layer, ID unification, and into a parent segment — as an interactive
  React dashboard. Answers "where does this column come from?" without tracing
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
golden layer, ID unification, and into a parent segment. Renders as an interactive
React dashboard. Each lineage edge **must cite the schema query or SQL that proved
the connection** — never infer silently.

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
| `treasure-work-skills:react-dashboard` | ✅ Available | Used in Step 7 for rendering. |
| `sql-skills:trino` | ✅ Available | Used throughout for TD Trino SQL patterns. |

---

## Step 0 — Log Invocation (Inline Tracking — No Hooks)

Run silently before any other work. Do not block on failure.
This mirrors the `skill-usage-tracker.md` schema exactly: `(id, user_id, account_id, skill_name)`.

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_eu01)
export TDX_SITE=eu01

# Resolve user identity — same method as skill-usage-tracker.md
TD_STATUS=$(tdx status 2>/dev/null)
USER_ID=$(echo "$TD_STATUS" | grep "^User:" | awk '{print $2}')
ACCOUNT_ID=$(echo "$TD_STATUS" | grep "^Account ID:" | awk '{print $3}')
RECORD_ID=$(python3 -c "import uuid; print(uuid.uuid4())" 2>/dev/null || cat /proc/sys/kernel/random/uuid 2>/dev/null || echo "$(date +%s)-$$")

# Ensure ai_usage database and table exist (idempotent)
tdx api -X POST /v3/database/create/ai_usage 2>/dev/null || true
tdx api -X POST /v3/table/create/ai_usage/skills_usage_tracker/log 2>/dev/null || true

# Set schema if not already set
tdx api -X POST \
  --data '{"schema":"[[\"id\",\"string\"],[\"user_id\",\"string\"],[\"account_id\",\"string\"],[\"skill_name\",\"string\"]]"}' \
  /v3/table/update-schema/ai_usage/skills_usage_tracker 2>/dev/null || true

# Insert usage record via streaming write API (tdx query INSERT is read-only in TD Trino)
# Use bulk import API endpoint for the row insert
tdx api -X POST \
  --data "{\"columns\":[{\"name\":\"id\",\"value\":\"$RECORD_ID\"},{\"name\":\"user_id\",\"value\":\"$USER_ID\"},{\"name\":\"account_id\",\"value\":\"$ACCOUNT_ID\"},{\"name\":\"skill_name\",\"value\":\"uc-data-lineage\"}]}" \
  /v3/table/import_with_id/ai_usage/skills_usage_tracker/json 2>/dev/null || true
```

> **Note for Treasure AI Studio:** Pre/post hooks are not available. This inline Step 0 is the only tracking mechanism. The `{customer_slug}_ai_poc_tracking.skill_usage` table referenced in the JIRA ticket maps to this same tracking pattern pending clarification from Sébastien Pujalte (ticket reporter) on whether a separate customer-scoped table is intended.
>
> **Note on INSERT:** TD Trino (`tdx query`) is read-only and does not support DML INSERT. Row writes use the `/v3/table/import_with_id` REST endpoint above.

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

Check whether the quality flags table exists:

```bash
tdx tables "<customer_db>.*" --json  # look for data_quality_flags or similar
```

If a table named `data_quality_flags` (or equivalent) exists in the customer's tracking database:

```sql
SELECT table_name, issue_type, severity
FROM <tracking_db>.data_quality_flags
WHERE severity IN ('HIGH', 'CRITICAL')
```

Attach the results to the relevant layer nodes in the dashboard as ⚠️ badges.
If the table does not exist, skip silently — do not show an error to the user.

---

## Step 7 — Render the Interactive React Dashboard

Build the complete data object from all steps above, then render with `mcp__work__render_react`.

All numbers must come from real queries — no placeholder values.

### Data Object to Populate

```js
const lineageData = {
  parentSegment: {
    name:        "<TARGET_SEGMENT>",
    id:          "<parent_id>",
    outputDb:    "cdp_audience_<id>",
    columnCount: 0,   // from ps_schema.json customers.columns length
    rowCount:    0,   // from SELECT COUNT(*) on customers table
  },
  layers: [
    // One entry per discovered layer — include all 5, mark missing ones
    {
      id:              "RAW",
      db:              "<raw_db_name>",
      tables:          [{ name: "<table>", rowCount: 0, keyColumns: ["email","id"] }],
      totalRows:       0,
      found:           true,   // false if no matching DB found
      hasQualityFlag:  false,
    },
    { id: "STAGING",        db: "...", tables: [...], totalRows: 0, found: true,  hasQualityFlag: false },
    { id: "GOLDEN",         db: "...", tables: [...], totalRows: 0, found: true,  hasQualityFlag: false },
    {
      id:              "UNIFICATION",
      db:              "cdp_unification_<sub>",
      tables:          [...],
      canonicalGroups: 0,
      totalIds:        0,
      mergeRatePct:    0,
      largestCluster:  0,
      found:           true,
    },
    { id: "PARENT_SEGMENT", db: "cdp_audience_<id>", tables: [...], totalRows: 0, found: true },
  ],
  columnLineage: [
    // One entry per column in TARGET_COLUMNS (or ALL_COLUMNS)
    {
      column:   "email",
      resolved: true,
      hops: [
        {
          layer:      "RAW",
          db:         "<raw_db>",
          table:      "<raw_table>",
          column:     "email_address",
          transform:  "pass-through",
          confidence: "CONFIRMED",
          evidence:   "stg_contacts.sql:14 — SELECT email_address FROM raw.contacts",
        },
        {
          layer:      "STAGING",
          db:         "<stg_db>",
          table:      "<stg_table>",
          column:     "trfmd_email",
          transform:  "LOWER(TRIM(email_address))",
          confidence: "CONFIRMED",
          evidence:   "stg_contacts.sql:18 — LOWER(TRIM(email_address)) AS trfmd_email",
        },
        {
          layer:      "GOLDEN",
          db:         "<gld_db>",
          table:      "<gld_table>",
          column:     "email",
          transform:  "pass-through",
          confidence: "CONFIRMED",
          evidence:   "gld_master.sql:8 — trfmd_email AS email",
        },
        {
          layer:      "PARENT_SEGMENT",
          db:         "cdp_audience_<id>",
          table:      "customers",
          column:     "email",
          transform:  "attribute join on cdp_customer_id",
          confidence: "CONFIRMED",
          evidence:   "parent segment attribute definition",
        },
      ],
    },
  ],
  gaps: [
    // One entry per UNRESOLVED column
    {
      column:         "loyalty_score",
      lastLayerFound: "GOLDEN",
      checkedDbs:     ["stg_contacts", "stg_transactions"],
      reason:         "Column found in golden layer but no staging table contains it",
      recommendation: "Pull orchestration workflow SQL for evidence",
    },
  ],
  workflowEvidence: {
    projectName:   "<workflow_project or null>",
    sqlFilesRead:  0,
    digFilesRead:  0,
    evidenceLevel: "CONFIRMED",  // or "SCHEMA-INFERRED"
  },
  // IMPORTANT: Derive kpis from the arrays above BEFORE calling render_react
  // Do not leave as zeros — compute these values from columnLineage and layers
  kpis: {
    totalColumns:    0,   // → columnLineage.length
    fullyTraced:     0,   // → columnLineage.filter(c => c.resolved).length
    confirmedEdges:  0,   // → columnLineage.flatMap(c => c.hops).filter(h => h.confidence === "CONFIRMED").length
    schemaOnly:      0,   // → hops with confidence LIKELY or POSSIBLE
    unresolved:      0,   // → gaps.length
    layersPresent:   0,   // → layers.filter(l => l.found).length
  },
};
```

### React Component

```jsx
export default function DataLineageDashboard() {
  // lineageData is populated above from real queries before rendering
  const [activeTab, setActiveTab]       = useState("overview");
  const [selectedColumn, setSelectedColumn] = useState(null);
  const [expandedEvidence, setExpandedEvidence] = useState({});

  // isDark is a pre-loaded global in render_react sandbox — use directly
  // Treasure AI brand colors
  const C = {
    primary:  "#494FFF",
    sec1:     "#8753FF",
    sec2:     "#C466D4",
    lime:     "#9DCC4C",
    yellow:   "#D9E47C",
    red:      "#EE2328",
    darkRed:  "#9E363E",
    salmon:   "#F69068",
  };

  const LAYER_COLORS = {
    RAW:            "#f97316",
    STAGING:        "#6366f1",
    GOLDEN:         "#a855f7",
    UNIFICATION:    "#10b981",
    PARENT_SEGMENT: "#f59e0b",
  };

  const confStyle = (c) => ({
    CONFIRMED:  { bg: C.lime,    label: "✅ CONFIRMED" },
    LIKELY:     { bg: C.yellow,  label: "⚠️ LIKELY"    },
    POSSIBLE:   { bg: C.salmon,  label: "⚠️ POSSIBLE"  },
    UNRESOLVED: { bg: C.red,     label: "❌ UNRESOLVED" },
  }[c] || { bg: "#6b7280", label: c });

  const tabs = [
    { id: "overview",  label: "Pipeline Overview"     },
    { id: "explorer",  label: "Field Lineage Explorer" },
    { id: "layers",    label: "Layer Detail"           },
    { id: "gaps",      label: "Lineage Gaps & Risks"   },
  ];

  // ── Reusable components ──────────────────────────────────────────────────

  const KPI = ({ title, value, sub, color }) => (
    <div className="p-4 rounded-xl border border-gray-200 dark:border-gray-700
                    bg-gray-50 dark:bg-gray-800/50">
      <div className="text-[10px] font-bold uppercase tracking-widest
                      text-gray-400 mb-1">{title}</div>
      <div className="text-2xl font-extrabold"
           style={{ color: color || C.primary }}>{value}</div>
      {sub && <div className="text-[11px] text-gray-500 mt-1">{sub}</div>}
    </div>
  );

  const ConfBadge = ({ level }) => {
    const s = confStyle(level);
    return (
      <span className="inline-block text-[10px] font-bold px-2 py-0.5
                       rounded-full text-white"
            style={{ background: s.bg }}>{s.label}</span>
    );
  };

  // Layer node — shows table name, row count, AND key columns (ticket requirement)
  const LayerNode = ({ layerData, isSelected, onColumnClick }) => {
    const color = LAYER_COLORS[layerData.id] || "#6b7280";
    if (!layerData.found) {
      return (
        <div className="flex-1 rounded-xl border-2 border-dashed p-3 text-center opacity-40"
             style={{ borderColor: color }}>
          <div className="text-[10px] font-bold uppercase tracking-widest"
               style={{ color }}>{layerData.id}</div>
          <div className="text-xs text-gray-400 mt-1">Layer not found</div>
        </div>
      );
    }
    return (
      <div className="flex-1 rounded-xl border-2 p-3"
           style={{
             borderColor: isSelected ? C.primary : color,
             background:  isDark ? color + "18" : color + "0d",
             boxShadow:   isSelected ? `0 0 0 3px ${C.primary}44` : "none",
           }}>
        <div className="text-[10px] font-bold uppercase tracking-widest mb-1"
             style={{ color }}>{layerData.id}</div>
        <div className="text-sm font-semibold text-gray-800 dark:text-gray-100
                        truncate">{layerData.db}</div>
        <div className="text-xs text-gray-500 mt-1">
          {layerData.tables?.length} tables ·{" "}
          {layerData.totalRows?.toLocaleString()} rows
        </div>
        {/* Key columns — ticket requirement */}
        {layerData.tables?.slice(0, 2).map(t => (
          <div key={t.name} className="mt-2">
            <div className="text-[10px] font-semibold text-gray-500">{t.name}</div>
            <div className="flex flex-wrap gap-1 mt-0.5">
              {(t.keyColumns || []).slice(0, 4).map(col => (
                <button
                  key={col}
                  onClick={() => onColumnClick && onColumnClick(col)}
                  className="text-[9px] px-1.5 py-0.5 rounded border
                             hover:opacity-80 cursor-pointer transition-opacity"
                  style={{ borderColor: color + "66", color,
                           background: isDark ? color + "22" : color + "11" }}>
                  {col}
                </button>
              ))}
              {(t.keyColumns || []).length > 4 && (
                <span className="text-[9px] text-gray-400">
                  +{t.keyColumns.length - 4} more
                </span>
              )}
            </div>
          </div>
        ))}
        {layerData.hasQualityFlag && (
          <div className="text-[10px] mt-2 font-semibold text-amber-500">
            ⚠️ Quality issues flagged
          </div>
        )}
        {layerData.id === "UNIFICATION" && layerData.found && (
          <div className="text-[10px] text-gray-400 mt-1">
            {layerData.canonicalGroups?.toLocaleString()} groups ·{" "}
            {layerData.mergeRatePct}% merged
          </div>
        )}
      </div>
    );
  };

  // Arrow between layers — coloured by min confidence of edges passing through
  const FlowArrow = ({ confidence }) => {
    const color = confStyle(confidence || "CONFIRMED").bg;
    return (
      <div className="flex flex-col items-center justify-center w-6 shrink-0">
        <div className="w-0.5 flex-1" style={{ background: color + "66" }} />
        <div className="text-sm" style={{ color }}>▶</div>
        <div className="w-0.5 flex-1" style={{ background: color + "66" }} />
      </div>
    );
  };

  // ── Tab: Pipeline Overview ───────────────────────────────────────────────
  const OverviewTab = () => {
    const kpis = lineageData.kpis || {};
    // Determine worst confidence per inter-layer arrow
    const arrowConf = (fromId, toId) => {
      const hops = lineageData.columnLineage
        ?.flatMap(c => c.hops)
        .filter(h => h.layer === toId);
      if (!hops?.length) return "CONFIRMED";
      const order = ["UNRESOLVED","POSSIBLE","LIKELY","CONFIRMED"];
      return hops.reduce((worst, h) => {
        return order.indexOf(h.confidence) < order.indexOf(worst)
          ? h.confidence : worst;
      }, "CONFIRMED");
    };

    return (
      <div className="space-y-6">
        {/* KPI row */}
        <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-6 gap-3">
          <KPI title="Total Columns"   value={kpis.totalColumns  || 0} color={C.primary} />
          <KPI title="Fully Traced"    value={kpis.fullyTraced   || 0} color={C.lime}    />
          <KPI title="Confirmed Edges" value={kpis.confirmedEdges|| 0} color={C.lime}    />
          <KPI title="Schema Only"     value={kpis.schemaOnly    || 0} color={C.yellow}  />
          <KPI title="Unresolved"      value={kpis.unresolved    || 0} color={C.red}     />
          <KPI title="Layers Found"    value={kpis.layersPresent || 0} color={C.sec1}    />
        </div>

        {/* Evidence level banner */}
        {lineageData.workflowEvidence?.evidenceLevel === "SCHEMA-INFERRED" && (
          <div className="p-3 rounded-lg border border-amber-400 bg-amber-50
                          dark:bg-amber-900/20 text-amber-700 dark:text-amber-300
                          text-sm font-medium">
            ⚠️ No orchestration workflow SQL found — all lineage edges are
            SCHEMA-INFERRED. Pull workflow project{" "}
            <strong>{lineageData.workflowEvidence.projectName || "(unknown)"}</strong>{" "}
            for SQL-confirmed lineage.
          </div>
        )}

        {/* Pipeline flow diagram — left to right, clickable columns */}
        <div>
          <div className="text-xs font-bold uppercase tracking-widest
                          text-gray-400 mb-3">
            Pipeline Flow — click any column to trace its lineage
          </div>
          <div className="flex items-stretch gap-0">
            {lineageData.layers?.map((layer, i) => (
              <React.Fragment key={layer.id}>
                <LayerNode
                  layerData={layer}
                  isSelected={selectedColumn &&
                    lineageData.columnLineage?.find(c => c.column === selectedColumn)
                      ?.hops?.some(h => h.layer === layer.id)}
                  onColumnClick={(col) => {
                    setSelectedColumn(col);
                    setActiveTab("explorer");
                  }}
                />
                {i < lineageData.layers.length - 1 && (
                  <FlowArrow
                    confidence={arrowConf(
                      layer.id,
                      lineageData.layers[i + 1]?.id
                    )}
                  />
                )}
              </React.Fragment>
            ))}
          </div>
        </div>

        {/* Confidence legend */}
        <div className="flex flex-wrap gap-3 text-xs">
          {["CONFIRMED","LIKELY","POSSIBLE","UNRESOLVED"].map(c => (
            <span key={c} className="flex items-center gap-1">
              <span className="w-3 h-3 rounded-full inline-block"
                    style={{ background: confStyle(c).bg }} />
              {confStyle(c).label}
            </span>
          ))}
        </div>
      </div>
    );
  };

  // ── Tab: Field Lineage Explorer ──────────────────────────────────────────
  // Dropdown AND clickable column nodes in overview both feed this tab
  const ExplorerTab = () => {
    const allCols = lineageData.columnLineage?.map(c => c.column) || [];
    const trace   = lineageData.columnLineage?.find(c => c.column === selectedColumn);

    return (
      <div className="space-y-4">
        <div className="flex items-center gap-3">
          <label className="text-sm font-medium text-gray-600 dark:text-gray-300">
            Select column:
          </label>
          <select
            value={selectedColumn || ""}
            onChange={e => setSelectedColumn(e.target.value)}
            className="text-sm rounded-lg border border-gray-300 dark:border-gray-600
                       bg-white dark:bg-gray-800 px-3 py-1.5 text-gray-800
                       dark:text-gray-100">
            <option value="">— choose a column —</option>
            {allCols.map(c => (
              <option key={c} value={c}>{c}</option>
            ))}
          </select>
        </div>

        {!selectedColumn && (
          <div className="text-gray-400 text-sm">
            Select a column above or click a column pill in the Pipeline Overview tab.
          </div>
        )}

        {trace && (
          <div className="space-y-3">
            <div className="flex items-center gap-2">
              <span className="font-bold text-base">{trace.column}</span>
              {trace.resolved
                ? <span className="text-xs text-green-500 font-semibold">✅ Fully Traced</span>
                : <span className="text-xs text-red-500 font-semibold">❌ Unresolved</span>}
            </div>

            {/* Hop-by-hop lineage path — left to right */}
            <div className="flex items-start gap-0 overflow-x-auto pb-2">
              {trace.hops.map((hop, i) => {
                const color = LAYER_COLORS[hop.layer] || "#6b7280";
                const evKey = `${trace.column}-${i}`;
                return (
                  <React.Fragment key={i}>
                    <div className="min-w-[180px] rounded-xl border-2 p-3"
                         style={{
                           borderColor: color,
                           background:  isDark ? color + "18" : color + "0d",
                         }}>
                      <div className="text-[9px] font-bold uppercase tracking-widest"
                           style={{ color }}>{hop.layer}</div>
                      <div className="text-[10px] text-gray-500 truncate">
                        {hop.db}.{hop.table}
                      </div>
                      <div className="font-semibold text-sm mt-1
                                      text-gray-800 dark:text-gray-100">
                        {hop.column}
                      </div>
                      {hop.transform !== "pass-through" && (
                        <div className="text-[10px] mt-1 font-mono text-gray-500
                                        bg-gray-100 dark:bg-gray-700 rounded p-1
                                        break-all">
                          {hop.transform}
                        </div>
                      )}
                      <div className="mt-2">
                        <ConfBadge level={hop.confidence} />
                      </div>
                      {/* Show/hide evidence toggle */}
                      <button
                        onClick={() => setExpandedEvidence(prev => ({
                          ...prev, [evKey]: !prev[evKey]
                        }))}
                        className="text-[9px] mt-1 underline text-gray-400
                                   hover:text-gray-600">
                        {expandedEvidence[evKey] ? "Hide" : "Show"} evidence
                      </button>
                      {expandedEvidence[evKey] && (
                        <div className="text-[10px] mt-1 font-mono text-gray-500
                                        bg-gray-100 dark:bg-gray-700 rounded p-1
                                        break-all border border-gray-200
                                        dark:border-gray-600">
                          {hop.evidence}
                        </div>
                      )}
                    </div>
                    {i < trace.hops.length - 1 && (
                      <div className="flex items-center self-center px-1">
                        <span style={{ color: confStyle(trace.hops[i+1].confidence).bg }}>
                          →
                        </span>
                      </div>
                    )}
                  </React.Fragment>
                );
              })}
            </div>
          </div>
        )}
      </div>
    );
  };

  // ── Tab: Layer Detail ────────────────────────────────────────────────────
  const LayersTab = () => (
    <div className="space-y-4">
      {lineageData.layers?.map(layer => {
        const color = LAYER_COLORS[layer.id] || "#6b7280";
        return (
          <div key={layer.id}
               className="rounded-xl border p-4"
               style={{ borderColor: color + "44" }}>
            <div className="flex items-center gap-2 mb-3">
              <span className="w-2 h-2 rounded-full inline-block"
                    style={{ background: color }} />
              <span className="font-bold text-sm" style={{ color }}>
                {layer.id}
              </span>
              {!layer.found && (
                <span className="text-xs text-gray-400">(not found in this account)</span>
              )}
              <span className="text-xs text-gray-500 ml-auto">
                {layer.db}
              </span>
            </div>
            {layer.found && (
              <div className="overflow-x-auto">
                <table className="w-full text-xs">
                  <thead>
                    <tr className="text-gray-400">
                      <th className="text-left py-1 pr-4">Table</th>
                      <th className="text-right py-1 pr-4">Rows</th>
                      <th className="text-left py-1">Key Columns</th>
                    </tr>
                  </thead>
                  <tbody>
                    {layer.tables?.map(t => (
                      <tr key={t.name}
                          className="border-t border-gray-100 dark:border-gray-800">
                        <td className="py-1.5 pr-4 font-mono">{t.name}</td>
                        <td className="py-1.5 pr-4 text-right text-gray-500">
                          {t.rowCount?.toLocaleString()}
                        </td>
                        <td className="py-1.5">
                          <div className="flex flex-wrap gap-1">
                            {(t.keyColumns || []).map(col => (
                              <span key={col}
                                    className="text-[9px] px-1.5 py-0.5 rounded"
                                    style={{ background: isDark
                                      ? color + "22" : color + "11",
                                      color }}>
                                {col}
                              </span>
                            ))}
                          </div>
                        </td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            )}
            {layer.id === "UNIFICATION" && layer.found && (
              <div className="mt-3 grid grid-cols-4 gap-2">
                <KPI title="Canonical Groups" value={layer.canonicalGroups?.toLocaleString()} color={C.sec1} />
                <KPI title="Total IDs"        value={layer.totalIds?.toLocaleString()}        color={C.sec1} />
                <KPI title="Merge Rate"       value={`${layer.mergeRatePct}%`}                color={C.lime} />
                <KPI title="Largest Cluster"  value={layer.largestCluster}                    color={layer.largestCluster > 100 ? C.red : C.lime} />
              </div>
            )}
          </div>
        );
      })}
    </div>
  );

  // ── Tab: Lineage Gaps & Risks ────────────────────────────────────────────
  const GapsTab = () => {
    const schemaOnly = lineageData.columnLineage?.filter(c =>
      c.hops?.some(h => h.confidence === "LIKELY" || h.confidence === "POSSIBLE")
    ) || [];

    return (
      <div className="space-y-6">
        {/* Unresolved columns */}
        <div>
          <div className="text-xs font-bold uppercase tracking-widest
                          text-gray-400 mb-2">
            Unresolved Columns ({lineageData.gaps?.length || 0})
          </div>
          {!lineageData.gaps?.length ? (
            <div className="text-green-500 text-sm font-medium">
              ✅ All columns successfully traced end-to-end.
            </div>
          ) : (
            <div className="space-y-2">
              {lineageData.gaps.map(gap => (
                <div key={gap.column}
                     className="rounded-lg border border-red-300 dark:border-red-700
                                p-3 bg-red-50 dark:bg-red-900/20">
                  <div className="font-semibold text-sm text-red-600
                                  dark:text-red-400">{gap.column}</div>
                  <div className="text-xs text-gray-500 mt-0.5">
                    Last traced: <strong>{gap.lastLayerFound}</strong>
                  </div>
                  <div className="text-xs text-gray-500">
                    Checked: {gap.checkedDbs?.join(", ")}
                  </div>
                  <div className="text-xs text-gray-600 dark:text-gray-300 mt-1">
                    {gap.reason}
                  </div>
                  <div className="text-xs text-blue-500 mt-1">
                    💡 {gap.recommendation}
                  </div>
                </div>
              ))}
            </div>
          )}
        </div>

        {/* Schema-inferred only */}
        {schemaOnly.length > 0 && (
          <div>
            <div className="text-xs font-bold uppercase tracking-widest
                            text-gray-400 mb-2">
              Schema-Inferred Only — No SQL Evidence ({schemaOnly.length})
            </div>
            <div className="overflow-x-auto rounded-lg border
                            border-gray-200 dark:border-gray-700">
              <table className="w-full text-xs">
                <thead>
                  <tr style={{ background: isDark
                    ? "rgba(73,79,255,0.15)" : "rgba(73,79,255,0.06)" }}>
                    <th className="px-3 py-2 text-left">Column</th>
                    <th className="px-3 py-2 text-left">Weakest Confidence</th>
                    <th className="px-3 py-2 text-left">Layer</th>
                  </tr>
                </thead>
                <tbody>
                  {schemaOnly.map((c, i) => {
                    const weak = c.hops?.reduce((w, h) => {
                      const o = ["UNRESOLVED","POSSIBLE","LIKELY","CONFIRMED"];
                      return o.indexOf(h.confidence) < o.indexOf(w.confidence)
                        ? h : w;
                    }, c.hops[0]);
                    return (
                      <tr key={c.column}
                          className={i % 2 === 0
                            ? "bg-gray-50 dark:bg-gray-800/30" : ""}>
                        <td className="px-3 py-2 font-mono">{c.column}</td>
                        <td className="px-3 py-2">
                          <ConfBadge level={weak?.confidence} />
                        </td>
                        <td className="px-3 py-2 text-gray-500">{weak?.layer}</td>
                      </tr>
                    );
                  })}
                </tbody>
              </table>
            </div>
            <div className="text-xs text-amber-600 dark:text-amber-400 mt-2">
              💡 Pull orchestration workflow SQL to confirm these edges.
            </div>
          </div>
        )}
      </div>
    );
  };

  // ── Render ───────────────────────────────────────────────────────────────
  return (
    <div className="min-h-screen bg-white dark:bg-gray-900
                    text-gray-900 dark:text-gray-100 font-sans">
      {/* Header */}
      <div className="px-6 py-5 border-b border-gray-200 dark:border-gray-800"
           style={{ background: isDark
             ? "linear-gradient(135deg, rgba(73,79,255,0.12), rgba(135,83,255,0.08))"
             : "linear-gradient(135deg, rgba(73,79,255,0.05), rgba(135,83,255,0.03))" }}>
        <div className="text-xl font-extrabold" style={{ color: C.primary }}>
          Pipeline Lineage Dashboard
        </div>
        <div className="text-sm text-gray-500 mt-0.5">
          {lineageData.parentSegment?.name} ·{" "}
          {lineageData.parentSegment?.outputDb} ·{" "}
          Evidence: {lineageData.workflowEvidence?.evidenceLevel}
        </div>
      </div>

      {/* Tab bar */}
      <div className="flex border-b border-gray-200 dark:border-gray-800 px-6 overflow-x-auto">
        {tabs.map(t => (
          <button key={t.id}
            onClick={() => setActiveTab(t.id)}
            className={`px-4 py-3 text-sm font-medium mr-1 border-b-2
                        whitespace-nowrap transition-colors ${
              activeTab === t.id
                ? "border-[#494FFF] text-[#494FFF]"
                : "border-transparent text-gray-500 hover:text-gray-700 dark:hover:text-gray-300"
            }`}>{t.label}</button>
        ))}
      </div>

      {/* Tab content */}
      <div className="p-6">
        {activeTab === "overview"  && <OverviewTab />}
        {activeTab === "explorer"  && <ExplorerTab />}
        {activeTab === "layers"    && <LayersTab />}
        {activeTab === "gaps"      && <GapsTab />}
      </div>
    </div>
  );
}
```

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
