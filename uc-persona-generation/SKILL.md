---
name: uc-persona-generation
description: Use when the user wants AI-generated customer personas built from real behavioral and attribute data in Treasure Data — for campaign targeting, content strategy, or replacing manual persona workshops. Triggers on "generate personas", "create customer personas", "data-driven personas", "who are our customer segments", "persona generation", "uc-persona-generation". Built as a standalone skill adapted from `analysis-skills:persona-generation` — do not use that system skill's web-research/industry-context flow here, and do not modify it.
---

# uc-persona-generation — Data-Driven Persona Creation (UC5)

Generate 3–5 customer personas from a customer's own Treasure Data — not from training-data generalizations about "typical" customers in their industry. This is a new, standalone skill adapted from `analysis-skills:persona-generation`; it scopes that skill's approach to the customer's own TD data and adds a positioning branch, a value calculation, and usage logging. **Do not modify `analysis-skills:persona-generation`** — it's a shared system skill with a different job (industry research + web context).

**Source:** ARCH-1323 | **Team:** Marketing Ops | **Priority:** P3

---

## Interaction Flow

1. **Determine positioning** — efficiency (replacing manual persona work) or performance (net-new capability); this framing carries through every later step
2. **Identify behavioral data** — discover what's actually populated in the parent segment; flag gaps that would weaken persona quality
3. **Generate personas** — cluster on real data, name each persona, show the data profile behind it, ground campaign implications in that data
4. **Quantify value** — hours saved (efficiency) or projected revenue uplift (performance)
5. **Render the React dashboard** — persona cards with name, size, attributes, campaign recommendation, value
5b. **Next steps recommendations** — concrete, prioritised activation plan for each persona (required last step)
6. **Log the invocation**

---

## Step 1 — Determine Positioning

Ask, before touching any data:
1. "Do you currently build customer personas today?" (yes/no)
2. If yes: "How? Workshops, gut-feel/anecdotal, market research, something else?" and "Roughly how many hours does a persona cycle take, end to end?"
3. If no: proceed on the performance branch — this is a new capability for them, not a replacement for existing effort.

**This sets the positioning for the entire session — do not let it drift:**
- **Efficiency positioning** (they already build personas manually) → Step 4 uses the hours-saved formula, and Step 3's campaign implications should be framed as "faster version of what you already do," not a totally new deliverable.
- **Performance positioning** (no existing persona process) → Step 4 uses the revenue-uplift formula, and Step 3 should be framed as new targeting capability, not time savings (there's no manual baseline to save time against).

If the user's answer is ambiguous (e.g. "we sort of do this informally"), ask one clarifying follow-up rather than guessing — the acceptance criteria require this branch to be "determined at Step 1 and maintained throughout," so get it right before Step 3 or 4 build anything on top of it.

---

## Step 2 — Identify Behavioral Data (Feasibility Check, Not a Hard Stop)

Unlike `uc-campaign-analysis`'s feasibility gate, this is not a hard stop — personas can still be generated from a thinner data set, just with a caveat about quality. Discover what's actually there before promising anything.

Get the parent segment's output schema (per `parent-segment-analysis`):

```bash
tdx ps desc "<parent_segment_name>" -o /tmp/ps_schema.json
```

Confirm against the live database — `tdx ps desc` under-reports behavior tables:

```bash
tdx tables "{db}.*" 2>&1
```

**Check for each of these attribute categories** (scan by column presence, not by assuming table names — naming conventions vary by segment, confirmed across multiple demo segments in this environment):
- **Purchase history** — order/transaction tables with `total_amount`/`price`, `order_id`/`purchase_datetime`
- **Email engagement** — `behavior_email_send`/`_open`/`_click` or equivalent, for open rate / click rate per customer
- **Site behavior** — pageview/clickstream tables (`behavior_pageviews`, `behavior_clickstream`)
- **Product affinity** — a `product`/`category` column on purchase or pageview tables, aggregable per customer
- **Tenure** — a signup/registration date on `customers`, or `MIN(event_datetime)` across behavior tables as a proxy if no explicit signup date exists
- **Acquisition channel** — `acquisition_channel`, `utm_source`, or similar (not guaranteed — see `uc-campaign-analysis`'s same gotcha; if absent, note it as a gap, don't fabricate it)

**Report back to the user** which of the six are populated, which are missing, and — if more than half are missing — say explicitly that persona quality will be limited to what's available (e.g. "I only have purchase history and email engagement for this segment — personas will be built on those two dimensions, not the full behavioral picture"). Do not silently proceed as if a thin data set is equivalent to a rich one.

---

## Step 3 — Generate Personas

### 3a. Choose clustering dimensions

Use the 2–4 richest, most differentiating attributes found in Step 2 — not every available column. More dimensions doesn't mean better personas; it means noisier, harder-to-name clusters. Prioritize dimensions that vary meaningfully across the customer base (check with a quick `SELECT approx_distinct(...)` or a percentile spread before committing to a dimension).

### 3b. Cluster with percentile-bucket SQL, not a fixed number of arbitrary segments

There's no ML runtime guaranteed in this environment — do this with SQL percentile buckets on the chosen dimensions, which is transparent, auditable, and defensible as "grounded in actual data" (the acceptance criteria explicitly rule out training-data generalizations, and a black-box clustering step would undercut the same audit-trail requirement `uc-campaign-analysis` established for this POC).

**Pattern — bucket customers by purchase value and engagement, then cross them:**
```sql
WITH customer_metrics AS (
  SELECT
    c.cdp_customer_id,
    COALESCE(p.total_spend, 0) AS total_spend,
    COALESCE(p.order_count, 0) AS order_count,
    COALESCE(e_o.opens, 0) AS email_opens,
    COALESCE(e.sends, 0) AS email_sends
  FROM {db}.customers c
  LEFT JOIN (
    SELECT cdp_customer_id, SUM(total_amount) AS total_spend, COUNT(*) AS order_count
    FROM {db}.<purchase_table>
    GROUP BY 1
  ) p ON p.cdp_customer_id = c.cdp_customer_id
  LEFT JOIN (
    SELECT cdp_customer_id, COUNT(*) AS opens
    FROM {db}.behavior_email_open
    GROUP BY 1
  ) e_o ON e_o.cdp_customer_id = c.cdp_customer_id
  LEFT JOIN (
    SELECT cdp_customer_id, COUNT(*) AS sends
    FROM {db}.behavior_email_send
    GROUP BY 1
  ) e ON e.cdp_customer_id = c.cdp_customer_id
),
spenders_only AS (
  SELECT * FROM customer_metrics WHERE total_spend > 0
),
percentiles AS (
  SELECT
    approx_percentile(total_spend, 0.66) AS p66,
    approx_percentile(total_spend, 0.33) AS p33
  FROM spenders_only
),
buckets AS (
  SELECT m.*,
    CASE
      WHEN m.total_spend = 0 THEN 'no_purchase'
      WHEN m.total_spend >= pc.p66 THEN 'high_value'
      WHEN m.total_spend >= pc.p33 THEN 'mid_value'
      ELSE 'low_value'
    END AS value_tier,
    CASE
      WHEN m.email_sends > 0 AND CAST(m.email_opens AS DOUBLE) / m.email_sends >= 0.3 THEN 'engaged'
      ELSE 'unengaged'
    END AS engagement_tier
  FROM customer_metrics m
  CROSS JOIN percentiles pc
)
SELECT value_tier, engagement_tier,
       COUNT(*) AS segment_size,
       ROUND(AVG(total_spend), 2) AS avg_spend,
       ROUND(AVG(order_count), 2) AS avg_orders,
       ROUND(AVG(CAST(email_opens AS DOUBLE) / NULLIF(email_sends, 0)), 4) AS avg_open_rate
FROM buckets
GROUP BY 1, 2
ORDER BY avg_spend DESC
```

This naturally produces up to 8 candidate segments (`no_purchase` + 3 value tiers × 2 engagement tiers) — pick the 3–5 most distinct and largest as the personas; merge or drop tiny/near-duplicate segments rather than forcing exactly 5 if the data only supports fewer meaningfully different groups. `no_purchase` is frequently the largest single group in a demo/POC segment (confirmed live: 14.98M of 15M customers in one segment) — don't discard it as noise; if it's the majority of the base, it may itself be a valid persona ("never converted despite exposure") worth including.

**Verified gotcha — percentiles computed across the whole base collapse when the metric is sparse.** Confirmed live: in a segment where only ~0.14% of customers (21,670 of 15M) had any purchase at all, `approx_percentile(total_spend, 0.25/0.5/0.75) OVER ()` computed across **all** customers returned `0` for every threshold — because the 25th/50th/75th percentile of an overwhelmingly-zero column is 0. The result: `total_spend >= 0` was true for virtually everyone, and ~200K of ~203K customers landed in "high_value," including many with `total_spend = 0`. **Fix:** compute percentile thresholds only over the subset with a non-zero value for that metric (`spenders_only` CTE above), and give the "zero" group its own explicit tier (`no_purchase`) rather than letting it fall through into whichever bucket a broken threshold produces. Apply this same pattern to any other sparse metric (e.g. email engagement in a segment where most customers were never sent an email) — check what fraction of customers have a non-zero value before trusting a whole-base percentile split.

**Gotcha (same family as `uc-campaign-analysis`'s Step 3b gotchas):** `LEFT JOIN` against a customer with zero purchases/opens produces `NULL`, not `0` — always `COALESCE` before bucketing, or those customers silently vanish from every tier instead of landing in the lowest one.

### 3c. Name and describe each persona

For each retained segment, write:
- **Persona name** — short, memorable, grounded in the actual bucket (e.g. "High-Value Engaged Shopper," not a generic industry archetype pulled from training data)
- **Archetype description** — 1–2 sentences characterizing behavior, written from the numbers in front of you
- **Defining attributes** — the specific metrics that distinguish this persona from the others, with their actual values (not ranges lifted from general marketing knowledge)

### 3d. Show the data profile behind every persona (non-negotiable)

**Customer-driven requirement (ARCH-1323, Sébastien Pujalte, 2026-06-26):** *"Each persona must display the specific behavioural attributes and their values... that defined it... Never present a persona without showing the data profile behind it."* Customer evidence: *"I'm not completely sure where the data is coming from."* — Daniela, HarperCollins.

For every persona, show, in this order:
1. **The metrics and values** that defined the cluster — e.g. `avg_order_value: $240, email_open_rate: 62%, purchase_frequency: 4.2/quarter, segment_size: 3,140 customers`
2. **The SQL query** that produced those numbers — same non-negotiable rule as `uc-campaign-analysis`: no black box, ever, even for a "simple" persona
3. **Segment size as a count and a % of the base**, not just a count — a persona representing 40 customers out of 2M reads very differently than the same 40 out of 5,000

Never write a campaign implication ("this persona responds well to premium messaging") without it being traceable to one of the values shown in step 1 above. If a suggested implication isn't grounded in a specific metric from this persona's data, cut it.

### 3e. Campaign implications, grounded

For each persona, state: message angle, best channel, and offer type — each justified by a specific number from 3d (e.g. "62% email open rate supports leading with email over paid social for this group" — not "millennials respond well to email," which is a generalization, not a finding).

---

## Step 4 — Quantify Value

**These formulas are verified against the POC Value Translation Calculator (`/Users/gandhi.yellapu/Downloads/POC Value Translation Calculator.xlsx`, sheet "5. Persona Generation", cells B31–B51).**

**Efficiency path** (positioning = they already build personas manually, per Step 1):
```
hours_saved_per_persona  = manual_hours_per_persona - agent_time_hours
total_hours_saved        = hours_saved_per_persona × number_of_personas
operational_saving       = total_hours_saved × analyst_hourly_cost
```
- `manual_hours_per_persona` and `number_of_personas` — from Step 1 answers; ask directly if not given ("roughly how many hours per persona, and how many personas per cycle?")
- `agent_time_hours` — wall-clock time from Step 1 start to Step 5 completion (track same way as `uc-campaign-analysis`)
- `analyst_hourly_cost` — ask the user for their internal rate; do not assume a figure

**Performance path** (positioning = no existing persona process, per Step 1):
```
projected_campaign_uplift = monthly_campaign_revenue × expected_uplift_%
```
- `monthly_campaign_revenue` — ask the user directly, or derive from TD if campaign revenue data is available (see `uc-campaign-analysis` revenue-by-campaign pattern) — state which source was used
- `expected_uplift_%` — do not invent a number. Ask the user for their own assumption rather than asserting an industry figure. If they have none, leave it as an explicit placeholder variable and say so.

**Annual scaled value (both paths — show this alongside the session value):**
```
annual_scaled_value = session_value × scaling_lever
```
- `scaling_lever` — number of brands/markets/BUs this persona model could be replicated across; ask the user ("how many markets or brands would you roll this out to?"). This is the annualisation multiplier in the calculator (B47 × B50).

Show the formula and every input value used, not just the final number — same audit-trail principle as the rest of this skill.

---

## Step 5 — Render the React Dashboard

Use `render_react` to build persona cards: name, size (count + %), defining attributes, campaign recommendation, and per-persona value contribution.

**This environment's `render_react` sandbox is unreliable for anything beyond a simple, static component — confirmed through extensive prior testing, and reconfirmed by successfully rendering a 5-persona dashboard using this exact pattern:**
- ❌ `useState` combined with a `data` prop → blank screen every time
- ❌ A large data object passed as a `data` prop, especially nested → blank screen
- ❌ Components over ~150 lines, or with many internal helper functions → unreliable
- ❌ Tab systems, CSS-only interactivity → blank screen
- ✅ **The only confirmed-working pattern:** hardcode all persona data as module-level `const` declarations at the top of the file, no `data` prop, no `useState`, inline styles only, single scrollable page, under ~150 lines. Verified live: a 5-card persona dashboard (module-level `PERSONAS` array + `VALUE` object, `.map()` over the array, inline `style` objects, no hooks) rendered correctly on the first attempt.

**Note:** `render_react` is only available in Treasure Work (the desktop app). If this skill is running in Treasure Studio, fall back to writing a static HTML report instead — use the same card layout as a self-contained `.html` file.

**Workflow:**
1. Finish all SQL and value-calculation work in chat first — get every number finalized before writing any React.
2. Write the persona data as plain `const` objects/arrays at the top of the component file (not fetched, not passed as a prop).
3. Keep the component itself simple: map over the const array, render a card per persona, no state, no effects.
4. If the dashboard needs to show more personas than comfortably fit in ~150 lines, prefer a compact card layout over adding interactive show/hide state — state is exactly what breaks this sandbox.

If `render_react` produces a blank screen despite following this pattern, don't keep guessing — fall back to writing the same persona cards as a static HTML report (mirroring the `uc-campaign-analysis` Step 6 HTML pattern) and note to the user that the React dashboard render failed and this is the fallback.

---

## Step 5b — Next Steps Recommendations (Required Last Step)

After rendering the dashboard, **always conclude with a plain-language next steps section** directed at the persona for which this analysis is most actionable, or across all personas if the user hasn't specified a focus.

For each persona (or the selected one), provide **3–5 concrete, prioritised recommendations** covering:

1. **Immediate activation** — what campaign or journey to launch first, for which persona, and why (grounded in the data from Step 3d). Include recommended channel and messaging angle.
2. **Segment to create** — suggest a specific CDP child segment rule to isolate this persona in Treasure Data (e.g. "create a segment: `total_spend > P66_threshold AND email_open_rate >= 0.3`"). If the `segment` skill is available, offer to build it.
3. **Journey to configure** — recommend a journey type suited to this persona's behavior (e.g. "re-engagement journey for the 'Never Converted' persona — entry: no purchase in 365 days, goal: first order").
4. **Data gap to address** — if Step 2 identified a missing attribute category (e.g. acquisition channel, site behavior), recommend how to fill it and what new dimension it would unlock for persona refinement.
5. **Follow-up analysis** — suggest the next analytical question to answer with this data (e.g. "now that we know Persona A has 62% open rate, run `uc-campaign-analysis` to measure whether any existing campaign is already reaching them effectively").

Frame each recommendation in terms of the business outcome, not the technical action — the audience is Marketing Ops, not a TD engineer.

---

## Step 6 — Log Invocation

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
VALUES ('<uuid>', '<user_email_from_tdx_auth>', '<account_id_from_tdx_auth>', 'uc-persona-generation', <unix_timestamp>)
```

The `account_id` groups all runs for the same TD console together — use the numeric account ID from `tdx auth` output (e.g. `61`), not the parent segment name.

If the INSERT fails:
> "Usage tracking failed — `ai_usage.skills_usage_tracker` not writable in this environment. Personas delivered successfully."

Always report the outcome — never skip silently.

Always report the outcome — never skip silently.

---

## Acceptance Criteria (ARCH-1323)

- [ ] Positioning branch (efficiency vs. performance) determined at Step 1 and maintained through Steps 3–4 without drifting
- [ ] Personas generated from actual TD data via SQL (Step 3b), not training-data generalizations about "typical" customers
- [ ] Value calculation formula shown with all inputs (Step 4) — verified against POC Value Translation Calculator sheet "5. Persona Generation" (B31–B51): efficiency path uses hours_saved_per_persona × personas × hourly_cost; performance path uses monthly_revenue × uplift_%; annual scaled value = session_value × scaling_lever
- [ ] Output usable by a Marketing Ops user without TD context — plain-English persona descriptions and campaign implications, SQL available but not required reading
- [ ] Each persona displays its specific behavioral attribute values and the query that produced them (Step 3d — added 2026-06-26 per Sébastien Pujalte / HarperCollins feedback)
- [ ] Campaign implications are grounded in that persona's specific data, never generic (Step 3e)
- [ ] Next steps recommendations delivered after dashboard render — covers activation, CDP segment, journey, data gap, and follow-up analysis (Step 5b)

---

## What NOT to do

- Do NOT generate personas from industry knowledge or training-data assumptions about "typical retail customers" — every persona must trace back to a SQL query against this customer's TD data
- Do NOT present a persona without its underlying metric values and the query that produced them
- Do NOT let the positioning (efficiency vs. performance) drift between steps — decide once at Step 1, apply consistently
- Do NOT assert a projected uplift % as fact when quantifying performance-path value — ask the user for their own assumption, or leave it as an explicit placeholder
- Do NOT claim the value calculation "matches the POC Value Calculator exactly" without showing all formula inputs — the calculator is verified at `/Users/gandhi.yellapu/Downloads/POC Value Translation Calculator.xlsx` sheet "5. Persona Generation"
- Do NOT pass persona data into `render_react` as a `data` prop or use `useState` — confirmed to blank-screen in this environment; hardcode as module-level consts instead
- Do NOT use `render_react` when running in Treasure Studio — it is not available there; fall back to a static HTML file instead
- Do NOT force exactly 3–5 personas if the clustering only supports fewer meaningfully distinct groups — merge or drop near-duplicates instead of padding the count

## Related Skills

- **parent-segment-analysis** — schema discovery workflow this skill uses for Step 2; do not modify
- **uc-campaign-analysis** — sibling POC skill; shares the audit-trail (always-show-SQL), gotcha style, and HTML-fallback-reporting conventions used here
- **analysis-skills:persona-generation** — the system skill this is adapted from (adds web research/industry context); **do not modify**, and do not use its research flow here — this skill is TD-data-only

## POC Context

- **Jira:** ARCH-1323
- **Tier:** 2 — Use Case (UC5, Persona Generation)
- **Phase:** Weeks 4–6 skill build, then self-serve period
- **Value pillar:** Marketing Efficiency (if replacing manual persona work) or Operational Efficiency (hours saved) — determined by Step 1's positioning branch
- **Repo:** `treasure-data/ai-poc-framework` — `skills/uc-persona-generation/SKILL.md`
- **Do NOT modify** `analysis-skills:persona-generation` — this is a new, standalone skill, not a patch to the system one
