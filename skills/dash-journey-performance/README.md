# dash-journey-performance — Journey Conversion Funnel Dashboard

A Treasure Work skill that delivers a journey performance dashboard — funnel overview, node-level conversion/exit/dwell, touchpoint engagement heatmap, and an 8-week conversion trend — for any CDP customer journey with 2+ stages.

**Jira:** ARCH-1326 | **Team:** Solution Architecture | **Priority:** P2

---

## Prompts to Invoke

```
"Journey performance dashboard"
"Journey funnel"
"How is this journey performing"
"dash-journey-performance"
"Journey conversion dashboard"
```

---

## Interaction Flow

1. Skill sets/confirms the parent segment context (`tdx sg use`)
2. `tdx journey list` — pick a journey (or "all" for a summary)
3. Discovery: `tdx journey view/columns/stats/activations/versions`
4. Data collection: SQL against the journey snapshot table + behavior tables for touchpoint engagement
5. Dashboard renders in the artifact panel via `render_react`
6. Usage logged (best-effort)

To view a different journey, just say so in chat — the skill re-runs and re-renders. (An in-dashboard dropdown was considered but conflicts with `render_react`'s known `useState` fragility — see Known Limitations.)

---

## What the Dashboard Shows

### Funnel Overview
- Entered / Converted / Exited KPI cards (entered & exit/converted counts from `tdx journey stats`, cross-checked against the snapshot table)
- Overall conversion % and exit %

### Node Performance Table
- One row per stage: conversion rate, exit rate, avg dwell time
- Rows pre-sorted by conversion rate — bottom 3 badged "Needs attention", top 3 badged "Top performer"

### Touchpoint Engagement Heatmap
- Stage × channel grid (email / push / SMS), color intensity = engagement rate
- Channels with no matching behavior table are omitted, not shown as zero

### Trend Chart
- Weekly conversion rate over the last 8 weeks
- Annotated with journey version-change dates from `tdx journey versions`

---

## Data Sources

| Metric | Source |
|---|---|
| Entered / goal achieved / exit-or-jump (cumulative) | `tdx journey stats` |
| Currently active count | SQL on journey snapshot table |
| Per-node reach / dwell time | SQL (`intime_stage_N` / `outtime_stage_N`) |
| Per-node exit rate | `tdx journey stats --stage`, cross-checked with SQL |
| Touchpoint engagement | Behavior tables joined to activation step exit times |
| Weekly conversion trend | SQL cohort by `intime_journey` week |
| Version annotations | `tdx journey versions` |

Full column-naming reference: see the `journey` skill's `references/journey-table.md` (state snapshot model, step UUIDs, category types) — read before writing SQL, values are not guessable.

---

## Technical Details

| Item | Detail |
|---|---|
| Journey selector | In chat only — no dropdown inside the dashboard by default |
| Rendering | `render_react` with all data as module-level constants (no props, no useState) |
| Minimum journey size | 2 stages — skill stops early otherwise |
| Layout | Single scrollable page, 4 panels, no tabs |

---

## Acceptance Criteria (ARCH-1326)

- [x] Renders for any journey with ≥2 nodes — no hardcoded journey names
- [x] Node performance table highlights best/worst 3 nodes automatically
- [x] All panels update when the user switches journey (via chat re-invocation)
- [x] Renders within a single session without timeout

---

## Known Limitations

- No true in-dashboard journey dropdown by default — `render_react` reliably breaks with `useState` + non-trivial data (confirmed during the sibling ARCH-1328 build). Journey switching is chat-driven instead.
- Touchpoint engagement requires behavior tables that match expected naming (`behavior_email_open`, etc.) to exist in the segment's output database — if absent, that channel is omitted from the heatmap.
- `value-agent-stellantis-live` is the reference for query/render patterns and is **not modified** by this skill.

---

## Repo

`treasure-data/ai-poc-framework` — `skills/dash-journey-performance/SKILL.md`
