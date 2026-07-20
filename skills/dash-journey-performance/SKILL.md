---
name: dash-journey-performance
description: Use when building or running the Journey Performance Dashboard — journey conversion funnel, node performance, touchpoint engagement, and conversion trend for a CDP customer journey. Triggers on "journey performance dashboard", "journey funnel", "journey conversion", "dash-journey-performance", "how is this journey performing", "journey analytics dashboard".
---

# dash-journey-performance — Journey Conversion Funnel Dashboard

Build a customer-agnostic React dashboard showing how a journey is performing: funnel overview, per-node conversion/exit/dwell, touchpoint engagement, and an 8-week conversion trend. Scope is chosen in chat — either a single journey or all journeys at once (with an in-dashboard dropdown and an overview tab).

**Source:** ARCH-1326 | **Team:** CRM | **Priority:** P2

**This is a hard requirement, not a nice-to-have:**
> "If I don't know how it got here, there's no way that I would assign my name to this recommendation." — Lina, PLG session 2

Every conversion rate, exit rate, and engagement rate this skill renders **must** show its raw counts and the query that produced it. A bare percentage is not an acceptable output for this dashboard.

Query patterns follow the journey data model documented in the `journey` skill's `references/journey-table.md` and `references/analyze.md` — **read those two files before writing any SQL**. This skill does not duplicate that reference; it tells you which parts of it to use and how to shape the result into a dashboard.

---

## Interaction Flow

1. **Set parent segment context** — journey commands require it
2. **List journeys, ask the user whether they want a specific journey or all journeys**
3. Run discovery + data collection queries for the chosen scope
4. Render the dashboard via `render_react` using module-level constants (value-agent template style)
5. Log the invocation

---

## Step 0 — Set Parent Segment Context

`tdx journey list` fails with `Parent segment context required` until a parent segment is active.

```bash
tdx sg use "<parent_segment_name>"     # or: tdx journey list --parent-segment "<parent_segment_name>"
```

Ask the user for the parent segment name if it isn't already known from the session.

---

## Step 1 — Select Journey

```bash
tdx journey list
```

Show the returned journeys via `AskUserQuestion` (or plain chat) and **ask the user whether they want to analyze a specific journey or all journeys**. 

If the user says "all journeys": 
- Build the dashboard for every journey with `--include-stats` data and summarize them.
- Add a general "Account journey overview" tab/page with a date selector.
- Allow switching between individual journeys via a dropdown.

Do not hardcode journey names anywhere in code or SQL — always use the ones discovered this turn.

---

## Step 2 — Query Journey Performance Data

**Read `journey/references/journey-table.md` and `journey/references/analyze.md` now if you haven't already this session.** The journey table is a state-snapshot model with non-guessable column naming (`intime_stage_N`, `outtime_stage_N`, step UUIDs) — do not write SQL from memory.

**CRM traceability requirement (hard requirement, all of Step 2):** for every metric you compute below, hold onto three things, not just the final number — (1) the raw numerator and denominator counts, (2) the exact SQL or `tdx` command that produced them, (3) which source they came from (SQL snapshot vs. `tdx journey stats` cumulative). All three travel into Step 3 as paired module-level constants. Never let a query result collapse into a lone percentage before it reaches the render step.

### 2a. Discovery (required, run all)

```bash
tdx journey view "<journey>" --include-stats
tdx journey columns "<journey>"
tdx journey stats "<journey>"
tdx journey activations "<journey>"
tdx journey versions "<journey>"
```

`versions` gives you the dates of any journey structure changes — needed for the trend chart annotations in Step 3.

**Acceptance criterion check:** if `tdx journey columns` reports fewer than 2 stages, tell the user this journey doesn't have enough structure for a funnel dashboard and stop.

**Never-run journey check (verified against live data — do this before anything else in 2b):** a journey in `state: draft` that has never executed has no `journey_{id}` table yet, and `tdx journey stats` throws (`Cannot use 'in' operator to search for 'attributes' in null`) instead of returning zeros. Detect this cheaply and stop early with a clear message rather than letting either command fail mid-flow:

```bash
tdx tables "{db}.*" 2>&1 | grep -q "journey_{id}" || echo "NOT_YET_RUN"
```

If the table is absent, tell the user: *"This journey hasn't executed yet, so there's no performance data — trigger it with `tdx journey run` or wait for its schedule, then re-run this dashboard."* Do not attempt Step 2b–2e against a non-existent table.

### 2b. Total profiles (entered / active / converted / exited)

- `entered`, `goal_achieved`, `exit_or_jump` → from `tdx journey stats` (cumulative since launch — **not derivable from SQL**, the table only holds the current snapshot)
- `active` (currently in journey) → SQL, matches `stats.size`:

```sql
SELECT COUNT(*) AS active
FROM {db}.journey_{id}
WHERE intime_journey IS NOT NULL AND outtime_journey IS NULL
```

Overall conversion % = `goal_achieved / entered`. Overall exit % = `exit_or_jump / entered`.

**Carry forward for rendering:** `entered`, `goal_achieved`, `exit_or_jump` as raw ints (from `tdx journey stats`), plus the literal `tdx journey stats "<journey>"` command string as the source for the funnel panel's query display.

### 2c. Per-node performance

For each stage N (0-indexed, from `tdx journey columns`):

```sql
SELECT
  COUNT(intime_stage_{N})                                                  AS reached,
  COUNT(outtime_stage_{N})                                                 AS exited_stage,
  AVG(COALESCE(outtime_stage_{N}, {snapshot_time}) - intime_stage_{N}) / 86400.0 AS avg_dwell_days
FROM {db}.journey_{id}
WHERE intime_stage_{N} IS NOT NULL
```

- **Conversion rate** for stage N = `reached(stage N+1) / reached(stage N)`. For the last stage, use `goal_achieved / reached(last stage)` if a goal exists.
- **Exit rate** for stage N = per-stage `exit_or_jump_rate` from `tdx journey stats --stage "<stage name>"` (preferred, matches platform reporting) — cross-check against `exited_stage / reached` from SQL.
- **Dwell time** — always from SQL (`avg_dwell_days` above); also compute median/p90 with `APPROX_PERCENTILE` per `analyze.md` if the user wants distribution detail.

**Verified >100% trap:** `reached(stage N+1) / reached(stage N)` can legitimately exceed 100% — confirmed on a live journey where stage 0 `reached = 400` but stage 1 `reached = 19,645` (4,911%), because `journey-table.md`'s documented pitfall #4 ("customers can skip stages / enter directly at a later stage") means stage N+1's population is not a subset of stage N's. Do not silently clamp or hide this — it is a real signal that most traffic bypasses stage N's entry criteria, which is itself a finding worth surfacing in the dashboard's insights, not a bug to mask. But do not label it "conversion rate" without qualification either: when `reached(N+1) > reached(N)`, render it as `"{count}/{denom}"` with an explicit note (e.g. "stage N+1 reached directly, not only via stage N") rather than a `%` that implies a strict funnel.

Sort nodes by conversion rate ascending before rendering — this makes "top 3 / bottom 3" a slice of the array, not a runtime sort (see Step 3 rendering constraints).

**Carry forward for rendering:** for each node, keep `reached` and the next stage's `reached` (or `goal_achieved`) as raw ints alongside the computed conversion rate — e.g. `{ conversionCount: 342, conversionDenom: 1840, conversionRate: 0.186 }` — plus the literal SQL text used for that stage (with `{N}` and `{step_uuid}` substituted for the real values, not left as placeholders). This is the pair the node table must render per cell, per the CRM traceability requirement.

### 2d. Engagement by touchpoint

Each `Activation` step (from `tdx journey columns`, `stepType: Activation`) has a channel (email / push / SMS) from its connector config (see `tdx journey activations "<journey>"`).

For each activation step in each stage:

```sql
-- "sent" = step completed
SELECT COUNT(outtime_stage_{N}_{step_uuid}) AS sent
FROM {db}.journey_{id}
WHERE outtime_stage_{N}_{step_uuid} IS NOT NULL
```

Then join to the customer's engagement behavior tables to get opens/clicks/responses — **discover the actual table names first**, do not assume:

```bash
tdx tables "{db}.*" --json 2>/dev/null   # or tdx ps desc if this segment has one
```

Look for tables matching `behavior_email_open`, `behavior_email_click`, `behavior_sms_send` / `behavior_sms_response`, `behavior_app_push`. Join on `cdp_customer_id` and restrict the event to the window `[outtime_stage_{N}_{step_uuid}, outtime_stage_{N}_{step_uuid} + 7 days]` (or the stage's exit time if earlier) so engagement is attributed to the correct touchpoint:

```sql
SELECT COUNT(DISTINCT j.cdp_customer_id) AS engaged
FROM {db}.journey_{id} j
JOIN {db}.behavior_email_open e
  ON e.cdp_customer_id = j.cdp_customer_id
 AND e.open_datetime BETWEEN j.outtime_stage_{N}_{step_uuid} AND j.outtime_stage_{N}_{step_uuid} + 604800
WHERE j.outtime_stage_{N}_{step_uuid} IS NOT NULL
```

**Verified column-naming trap:** behavior tables have a generic `time` column that is **not** the event timestamp — it's a constant load/ingestion time shared by every row in the table (confirmed: `MIN(time) = MAX(time)` across 77K+ rows in a live test table). Always join on the semantic per-event column instead (`open_datetime`, `click_datetime`, `send_datetime`, `app_push_send_datetime`, etc. — same convention documented in the `dash-unique-customer-view` skill). Run `tdx describe {db}.{behavior_table}` and pick the column whose name matches the event you're measuring, never `time`.

**Verified demo-data trap:** even with the correct column, synthetic/demo parent segments can have journey execution timestamps and behavior-event timestamps generated in disjoint eras (observed: journey exits in 2024–2025, behavior events in 2021–2023, on the same segment) — the customer IDs overlap but the 7-day attribution window never matches, silently producing an all-zero engagement heatmap that looks like "no one engages" but is actually a data-generation artifact. Before reporting 0% engagement across a channel, spot-check with a version of the query that drops the time window (`INTERSECT` sent-customers against the behavior table's full customer set). If that overlap is non-zero but the windowed join is zero, tell the user engagement can't be measured for this segment due to timestamp ranges not overlapping — do not present the heatmap cell as a real 0%.

Engagement rate = `engaged / sent`. If no matching behavior table exists for a channel, omit that channel from the heatmap for that node rather than showing a zero (a missing table is not the same as zero engagement).

**Carry forward for rendering:** for each heatmap cell, keep `engaged` and `sent` as raw ints plus the exact join SQL used (with real column/UUID names substituted). A heatmap cell is a rate with a tooltip, not just a color — the CRM traceability requirement applies here too.

### 2e. Conversion trend — 8 weeks, week-over-week

The journey table retains rows for customers who already exited (it is not append-only, but exited customers keep their historical `intime`/`outtime` timestamps until re-entry), so weekly cohorts can be built directly from the snapshot:

```sql
SELECT
  DATE_TRUNC('week', FROM_UNIXTIME(intime_journey)) AS entry_week,
  COUNT(*)                                          AS entered,
  COUNT(intime_goal)                                AS converted
FROM {db}.journey_{id}
WHERE intime_journey >= TO_UNIXTIME(CURRENT_DATE - INTERVAL '56' DAY)
GROUP BY 1
ORDER BY 1
```

**Verified Trino syntax note:** `INTERVAL '8' WEEK` throws a `SYNTAX_ERROR` on this engine — Trino's `INTERVAL` here only accepts `DAY`/`HOUR`/etc., not `WEEK`. Use `INTERVAL '56' DAY` (8×7) as shown above, not `WEEK`.

Weekly conversion rate = `converted / entered`. Cross-check the most recent week against `tdx journey stats --from <week_start> --to <week_end>` since that source is cumulative and authoritative for entered/goal counts.

**Data quality check:** in testing, some journey tables had rows where `intime_journey` was `NULL` but `intime_goal` was populated (customers who achieved the goal without a recorded entry timestamp — likely a backfill or reentry artifact). These rows are silently excluded by the `WHERE intime_journey >= ...` filter, which is correct for a week-bucketed trend, but call it out to the user if it represents a large share of `goal_achieved` (compare `COUNT(intime_goal)` with and without the journey-entry filter) — otherwise the trend will look far sparser than the funnel KPIs in Step 2b suggest.

Pull `tdx journey versions "<journey>"` dates that fall within the 8-week window — these become annotation markers on the trend chart (structure changed on this date).

**Carry forward for rendering:** for each week, keep `entered` and `converted` as raw ints alongside the rate — e.g. `{ week: "2026-05-25", converted: 342, entered: 1840, rate: 0.186 }` — plus the literal cohort-by-week SQL (Step 2e's query, with the actual date filter substituted). Every point on the trend line must resolve back to this triple when hovered or inspected — see Step 3.

---

## Step 3 — Render the Dashboard (Value-Agent Template Style)

Use `render_react` to build the dashboard. The user requested the **"value-agent template"** style, which uses a clean tabbed layout with KPI blocks, info banners, and specific branding (Navy, Blue, Success, Warning). 

### Layout & Structure (Value-Agent Style)
1. **Header & Context:** Top left shows the Journey Name (or "Account Journey Overview"), parent segment details, and a warning/context banner. Top right shows a dropdown to switch between evaluated journeys (if "all" was selected).
2. **Tabs:** Use a row of clean, bordered tabs (e.g., `Journey Impact`, `Node Performance`, `Touchpoints`, `8-Week Trends`). 
   - *State is permitted here:* You may use `useState` for simple tab switching (`activeTab`) and journey selection (`selectedJourneyIdx`) because hardcoded module-level data combined with simple string state is stable.
3. **8-Week Trend Chart (Required):** The user specifically requested to *keep* the 8-week trend chart. Render it using Recharts `LineChart` on one of the tabs. Tooltip content must show `"{converted}/{entered} = {rate}%"` per point.
4. **CRM Traceability Rule:** Never render a percentage alone. Every rate shown must be immediately followed by its raw counts in the form `"342/1,840 = 18.6%"`. The node table and trend chart must expose the exact query behind each number (use `<details><summary>` blocks).
5. **Account Journey Overview (If "All" Selected):** Add a specific tab that summarizes all journeys in a cross-journey benchmark table, comparing entries, conversion rates, and revenue, and include a date selector.

### Confirmed Working Render Pattern

```jsx
// ALL data hardcoded as module-level consts BEFORE the component to ensure stability
const BRAND = { navy: "#1A3A6B", blue: "#2B5BA0", accent: "#00A3E0", success: "#1B8A5A", warning: "#D4820A", lightGray: "#F7F8FA" };

const JOURNEYS_DATA = [
  { id: 'j1', name: 'Journey A', funnel: {...}, nodes: [...], trend: [...] },
  // ...
];

export default function App() {
  const [activeTab, setActiveTab] = useState("overview");
  const [selectedJourneyIdx, setSelectedJourneyIdx] = useState(0);
  
  const currentData = JOURNEYS_DATA[selectedJourneyIdx];

  const renderTab = () => {
    if (activeTab === "overview") return <OverviewTab />; // Cross-journey benchmark
    if (activeTab === "impact") return <ImpactTab data={currentData} />;
    if (activeTab === "nodes") return <NodesTab data={currentData} />;
    if (activeTab === "trends") return <TrendsTab data={currentData} />; // Recharts LineChart here
  };

  return (
    <div style={{ fontFamily: "Inter, -apple-system, sans-serif", background: BRAND.lightGray, minHeight: "100vh", padding: 20 }}>
      {/* Header, Dropdown, and Tabs */}
      {renderTab()}
    </div>
  );
}
```

**Rules:**
- No `data` prop passed to the main `App` export.
- All data as module-level `const` arrays, computed in chat before writing the component.
- Keep the component concise and rely on inline styles for the value-agent aesthetic.
---

## Step 4 — Usage Log

Get the current user and account from `tdx auth`, then INSERT into the shared tracking table:

```bash
tdx auth   # captures user email and account_id for the INSERT below
```

```python
import uuid, time
row_id = str(uuid.uuid4())
unix_ts = int(time.time())
```

```sql
INSERT INTO ai_usage.skills_usage_tracker (id, user_id, account_id, skill_name, time)
VALUES ('<uuid>', '<user_email_from_tdx_auth>', '<account_id_from_tdx_auth>', 'dash-journey-performance', <unix_timestamp>)
```

The `account_id` groups all runs for the same TD console together — use the numeric account ID from `tdx auth` output (e.g. `61`), not the parent segment name.

If the INSERT fails:
> "Usage tracking failed — `ai_usage.skills_usage_tracker` not writable in this environment. Dashboard rendered successfully."

Always report the outcome — never skip silently.

---

## Acceptance Criteria (ARCH-1326)

- [ ] Renders for any journey with ≥2 nodes — no hardcoded journey names (Step 1 stops early otherwise)
- [ ] Node performance table highlights best/worst 3 nodes automatically (pre-sorted array slice)
- [ ] Journey switching works via in-dashboard dropdown when "all journeys" is selected; single-journey mode uses chat re-invocation
- [ ] Renders within a single session without timeout
- [ ] All four panels present: funnel, node table, touchpoint heatmap, 8-week trend with version annotations
- [ ] **(CRM — universal)** Every conversion/exit/engagement rate displays as raw counts alongside the percentage (`"342/1,840 = 18.6%"`), never a bare percentage
- [ ] **(CRM — universal)** Node performance table shows the query used for each metric (expandable, no `useState`)
- [ ] **(CRM — universal)** Every trend chart data point is traceable to the underlying cohort-by-week query (tooltip + caption query text)

---

## What NOT to do

- Do NOT hardcode stage names, step UUIDs, or channel names — always discover via `tdx journey columns` / `tdx journey activations`
- Do NOT compute conversion trend from `stats` history alone — SQL cohort-by-week is the primary source (Step 2e); use `stats --from/--to` only to cross-check
- Do NOT show a zero engagement rate for a channel with no matching behavior table — omit it instead
- Do NOT render any conversion, exit, or engagement metric as a bare percentage — always pair it with raw counts (`count/denom = rate%`) and keep the source query reachable (CRM universal requirement)

## Related Skills

- **journey** — `references/journey-table.md` (data model) and `references/analyze.md` (5-phase analysis workflow this skill's Step 2 is derived from)
- **react-dashboard** — `render_react` sandbox globals (Recharts, Tailwind, dark mode)
- **connector-config** — activation channel/config fields, used in Step 2d to identify touchpoint types

## POC Context

- **Jira:** ARCH-1326
- **Team:** CRM
- **Tier:** 2 — Dashboard Build (Weeks 5–6)
- **Use cases:** UC4 (Journey Analysis), SC1 (time to insight)
- **Customer evidence:** "If I don't know how it got here, there's no way that I would assign my name to this recommendation." — Lina, PLG session 2
- **Repo:** `treasure-data/ai-poc-framework` — `skills/dash-journey-performance/SKILL.md`
- **Do NOT modify** `value-agent-stellantis-live` — this is a new, customer-agnostic skill; query/render patterns are informed by it but it stays untouched
