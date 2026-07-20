---
name: uc-campaign-analysis
description: Use when the user asks a free-form question about campaign performance — cross-channel comparisons, conversion rates by segment, unsubscribe rates, revenue attribution — against data already in Treasure Data. Triggers on "how did our campaign perform", "compare campaign to last month", "which segment converted best", "campaign performance analysis", "uc-campaign-analysis", "cross-channel campaign report". Built on `tdx-skills:parent-segment-analysis` — do not use for building segments, journeys, or briefs (see those skills instead).
---

# uc-campaign-analysis — Conversational Campaign Performance Analysis (UC6)

Answer free-form campaign performance questions in plain English against data already in Treasure Data. The "wow moment" is going from question to answer in seconds, with the SQL shown — not days of analyst work. This skill extends `tdx-skills:parent-segment-analysis` with campaign-specific query patterns and time-to-answer tracking — **do not modify `parent-segment-analysis`**, it's a shared system skill.

**Source:** ARCH-1324 | **Team:** Marketing Ops | **Priority:** P2

---

## Interaction Flow

1. **Scope available campaign data** — discover what campaign tables exist; hard stop if none found
2. **Start the clock** — record the time the question was asked, for the time-to-answer benchmark
3. **Answer the question** — generate SQL, run it, show both the answer and the query
4. **Surface one recommendation** — an actionable next step grounded in the data pattern found
5. **Stop the clock and report elapsed time**
6. **Render the answer into the HTML report and open it** — every question's output is a report, not chat text
7. **Log the invocation**

---

## Step 1 — Scope Available Campaign Data (Feasibility Gate)

This is an explicit hard stop, not a soft warning — the ticket requires it. Campaign data must already be in TD; this skill does not import or backfill it.

Get the parent segment's output schema (per `parent-segment-analysis`):

```bash
tdx ps desc "<parent_segment_name>" -o /tmp/ps_schema.json
```

Then confirm actual tables against the live database — `tdx ps desc` under-reports behavior tables the same way it does in the UCV skill:

```bash
tdx tables "{db}.*" 2>&1
```

**Look for campaign-bearing tables.** There's no fixed naming convention — discover by scanning for a `campaign_name` or `campaign_id` column, not by guessing table names:

```bash
tdx describe {db}.{candidate_table}
```

Common candidates (verified against live demo segments — actual names vary):
- `behavior_email_send` / `behavior_email_open` / `behavior_email_click` / `behavior_email_unsubscribe` / `behavior_email_bounce` — each carries `campaign_name`, `campaign_id`, and a per-event timestamp column (`send_datetime`, `open_datetime`, etc.)
- `behavior_ecommerce_orders` / `behavior_store_orders` — carry `campaign_name` and `total_amount`, the join point for revenue attribution
- `behavior_ad_campaign` / `behavior_sms_send` / `behavior_app_push` — same `campaign_name` pattern for other channels, when present
- Ad spend / cost tables (e.g. `behavior_ad_spend`, `behavior_ad_campaign` with a `spend`/`cost` column) are **not guaranteed** to exist in TD — if the user asks about ROAS or CPA and no spend table is found, say so explicitly rather than assuming a table name. If a spend table **is** found, note it — it enables the ROAS surfacing required in Step 3d.
- **Channel-attribution columns** (e.g. `attribution_channel`, `acquisition_channel`, `utm_source`, `td_utm_campaign`, `first_touch_channel`/`last_touch_channel`) are also **not guaranteed** — scan for them the same way as `campaign_name`/`campaign_id`, don't assume a fixed column name. If the user asks a channel-attribution question (e.g. "unsubscribe rate by acquisition channel") and no such column exists anywhere in the schema, this is a missing-dimension case (see "What NOT to do"), not a hard stop — campaign data may still exist even if attribution doesn't.

**Hard stop condition:** if no table in the segment's database has a `campaign_name` or `campaign_id` column, tell the user:
> "I don't see campaign-level data in this environment — no table has a `campaign_name` or `campaign_id` column. Campaign performance analysis needs campaign data already loaded into TD. This isn't something this skill can query around."

Do not proceed to Step 2 in that case. Do not substitute a different analysis to seem helpful — the acceptance criteria for this skill require a clean stop.

**Confirm scope with the user** once campaign tables are found: which campaigns and date range should be in scope for this session? (Not required before the first question if the user's question already specifies both.)

---

## Step 2 — Start the Clock

Before generating SQL, record a start timestamp (wall-clock, not query time) for the time-to-answer benchmark:

```python
import time
question_start = time.time()
```

This is a per-question timer, not a per-session one — track it for every question asked in the conversation, since SC1's benchmark is "seconds per question," not "seconds for the whole session."

---

## Step 3 — Answer the Question

### 3a. Map the question to tables and a time window

Read the user's question for:
- **Campaigns named** — match against `campaign_name` values (run a quick `SELECT DISTINCT campaign_name` if the user's phrasing doesn't exactly match — names are rarely exact, e.g. user says "March campaign," actual value is `"12 Days of State"`)
- **Time periods** — "last month," "Q1," "March vs February" — convert to explicit `td_time_string(..., 'M!')` groupings or `td_interval` filters. See `sql-skills:time-filtering` for offset syntax (`0M/now`, `-1M/0M`, etc.)
- **Metric requested** — sends, opens, clicks, unsubscribes, conversions, revenue, or a rate derived from two of these
- **Dimension requested** — by campaign, by segment/audience, by channel — if the dimension the user asks for isn't a column that exists anywhere in the schema (e.g. "acquisition channel" when no such column exists), say so and offer the closest available dimension instead of silently substituting one

### 3b. Write and run the SQL

Verified query patterns (Trino, tested against live campaign data):

**Volume/trend by month:**
```sql
SELECT td_time_string(send_datetime, 'M!') AS month, COUNT(*) AS emails_sent
FROM {db}.behavior_email_send
WHERE campaign_name = '<campaign>'
GROUP BY 1 ORDER BY 1
```

**Cross-channel funnel for a campaign in a given month (send → open → click):**
```sql
WITH sends AS (
  SELECT campaign_name, td_time_string(send_datetime, 'M!') AS month, COUNT(*) AS sends
  FROM {db}.behavior_email_send
  WHERE campaign_name = '<campaign>' AND td_time_string(send_datetime,'M!') IN ('<month1>','<month2>')
  GROUP BY 1,2
),
opens AS (
  SELECT campaign_name, td_time_string(open_datetime, 'M!') AS month, COUNT(*) AS opens
  FROM {db}.behavior_email_open
  WHERE campaign_name = '<campaign>' AND td_time_string(open_datetime,'M!') IN ('<month1>','<month2>')
  GROUP BY 1,2
),
clicks AS (
  SELECT campaign_name, td_time_string(click_datetime, 'M!') AS month, COUNT(*) AS clicks
  FROM {db}.behavior_email_click
  WHERE campaign_name = '<campaign>' AND td_time_string(click_datetime,'M!') IN ('<month1>','<month2>')
  GROUP BY 1,2
)
SELECT s.month, s.sends, o.opens, c.clicks
FROM sends s
LEFT JOIN opens o ON o.campaign_name = s.campaign_name AND o.month = s.month
LEFT JOIN clicks c ON c.campaign_name = s.campaign_name AND c.month = s.month
ORDER BY 1
```

**Revenue by campaign (attribution via `campaign_name` on the orders table):**
```sql
SELECT campaign_name, COUNT(*) AS orders, ROUND(SUM(total_amount), 2) AS revenue
FROM {db}.behavior_ecommerce_orders
WHERE td_interval(order_datetime, '<period>')
GROUP BY 1 ORDER BY 2 DESC
```

**Verified formatting trap:** without `ROUND(..., 2)`, large `SUM(total_amount)` values come back in scientific notation (e.g. `2.803091263E7` instead of `28030912.63`) — confirmed live. Always wrap monetary sums in `ROUND(..., 2)` before presenting, or the "answer" will show a value the user has to mentally convert.

**Unsubscribe rate for a campaign:**
```sql
WITH s AS (SELECT COUNT(*) AS sends FROM {db}.behavior_email_send WHERE campaign_name = '<campaign>'),
     u AS (SELECT COUNT(*) AS unsubs FROM {db}.behavior_email_unsubscribe WHERE campaign_name = '<campaign>')
SELECT unsubs, sends, CAST(unsubs AS DOUBLE) / sends AS unsub_rate FROM s, u
```

**Conversion rate by audience segment** (e.g. "which audience segment had the highest conversion rate last quarter?"):

The "segment" here is a dimension on the `customers` table, not the parent segment itself — look for a categorical column like `subscription_type`, `profile_type`, `loyalty_tier`, or similar (verified live: `subscription_type` on a customers table with values like `Core Subscriber` / `Registered User Free Trial` / `Not Subscriber`). If no such column exists, say so per the missing-dimension rule rather than substituting the parent segment name itself as "the segment."

```sql
WITH audience AS (
  SELECT cdp_customer_id, <segment_column>
  FROM {db}.customers
  WHERE <segment_column> IS NOT NULL
),
converters AS (
  SELECT DISTINCT cdp_customer_id
  FROM {db}.<conversion_table>  -- e.g. behavior_purchase, behavior_ecommerce_orders
  WHERE td_interval(<event_datetime_column>, '<period>')
)
SELECT a.<segment_column>,
       COUNT(DISTINCT a.cdp_customer_id) AS audience_size,
       COUNT(DISTINCT c.cdp_customer_id) AS converters,
       ROUND(CAST(COUNT(DISTINCT c.cdp_customer_id) AS DOUBLE) / COUNT(DISTINCT a.cdp_customer_id), 4) AS conversion_rate
FROM audience a
LEFT JOIN converters c ON c.cdp_customer_id = a.cdp_customer_id
GROUP BY 1
ORDER BY 4 DESC
```

**Verified gotchas (do not skip):**
- `campaign_name` values are exact strings from the source system, not slugs — always verify with `SELECT DISTINCT campaign_name FROM {table} LIMIT 20` before filtering, never guess a match
- Each behavior table's per-event timestamp column has a different name (`send_datetime`, `open_datetime`, `click_datetime`, `unsubscribe_datetime`, `order_datetime`) — the generic `time` column exists on every table but is often a constant ingestion timestamp, not the event time (confirmed live: `MIN(time) = MAX(time)` across 70K+ rows on more than one behavior table in this environment). Run `tdx describe` and use the semantic column, never `time`, for date filtering or grouping.
- When joining multiple behavior tables (funnel/cross-channel questions), join on `campaign_name` (and month, if grouping by month) — do not join behavior tables directly to each other on `cdp_customer_id` without also constraining campaign and a time window, or you'll cross-multiply unrelated events
- **`td_interval(..., '-1M/now')` silently returns zero rows against demo/POC data whose events stop years before today** (confirmed live: a segment's `order_datetime` topped out in 2023, `now` is 2026, "-1M/now" returned 0 rows with no error). Before trusting a "no data" result for a relative time question ("last month," "this quarter"), run `SELECT MAX(<event_datetime_column>) FROM {table}` first. If the max event time is far in the past, anchor the interval to that max date instead of `now` (e.g. `-1M/2023-08-22`) and tell the user you did so — do not report "no campaign activity" when the real cause is stale demo data.
- **A pre-computed `roas` (or similarly-named metric) column on a campaign table can be unrelated to the row's own `revenue`/`spend` values — verify before trusting it.** Confirmed live on a `behavior_campaigns` table with `campaign_revenue`, `spend`, and `roas` columns: the stored `roas` averaged ~31x, but `SUM(campaign_revenue)/SUM(spend)` on the same rows computed ~1.4x — a 20x+ discrepancy, and neither individual row's `roas` matched `campaign_revenue/spend` for that row. Always compute the ratio yourself from the raw revenue/spend (or click/impression) columns rather than trusting a same-named summary column; if you show both, label which is stored vs. computed so the user isn't misled by whichever is wrong.

### 3c. Present the answer

Show, in this order:
1. **The answer** — plain-English sentence with the actual numbers, not just a table dump (e.g. "Melt the Ice Campaign sent 1,955 emails in March vs 1,831 in February — a 6.8% increase, but click-through dropped from 11.6% to 9.9%.")
2. **The supporting data** — a small table or the raw query result
3. **The SQL used** — full query text, exactly as run, in a code block. This is not optional — the acceptance criteria require the SQL to be shown alongside every answer, no exceptions, even for simple questions.

Never present a number without the counts and query behind it — a bare percentage or bare "revenue is up" claim isn't acceptable; show the raw counts that produced any rate (e.g. "194/1,955 = 9.9%" not just "9.9%").

### 3d. Surface ROAS and CTR alongside conversion metrics, when available

**Customer-driven requirement (ARCH-1324, Sébastien Pujalte, 2026-06-26)** — flagged as the #1 hero pick from Eurowings' Frankfurt bootcamp research: combine campaign performance with ROAS and CTR, not just raw sends/opens/conversions.

For **any** campaign-performance answer (not only when the user explicitly asks for ROAS/CTR):
- **CTR** — always computable from `behavior_email_click` / `behavior_email_send` (or the send/click tables for the relevant channel). Include it whenever the question touches email/digital campaign performance, even if not explicitly requested.
- **ROAS** — only computable if an ad spend/cost table was found in Step 1. If found, join it in and surface ROAS alongside conversion metrics. If not found, do not silently omit it — say explicitly that ROAS can't be shown because no spend table exists in this environment (same rule as the missing-dimension case in "What NOT to do").

**CTR pattern (click-through rate for a campaign/month):**
```sql
WITH sends AS (
  SELECT campaign_name, COUNT(*) AS sends
  FROM {db}.behavior_email_send
  WHERE campaign_name = '<campaign>' AND td_interval(send_datetime, '<period>')
  GROUP BY 1
),
clicks AS (
  SELECT campaign_name, COUNT(*) AS clicks
  FROM {db}.behavior_email_click
  WHERE campaign_name = '<campaign>' AND td_interval(click_datetime, '<period>')
  GROUP BY 1
)
SELECT s.campaign_name, s.sends, c.clicks, CAST(c.clicks AS DOUBLE) / s.sends AS ctr
FROM sends s LEFT JOIN clicks c ON c.campaign_name = s.campaign_name
```

**ROAS pattern (return on ad spend — only if a spend table exists):**

Always compute ROAS as `SUM(revenue) / SUM(spend)` from raw columns — do not use a pre-existing `roas` column on the spend/campaign table even if one exists, until you've spot-checked it against the raw ratio (see gotcha above; a stored `roas` column was confirmed unrelated to `revenue/spend` on live data). If the spend table has its own revenue field (e.g. `campaign_revenue`), prefer it over joining to a separate orders table when both exist for the same campaign, to avoid double-counting or mismatched attribution windows.

```sql
WITH spend AS (
  SELECT campaign_name, ROUND(SUM(<spend_column>), 2) AS ad_spend
  FROM {db}.<ad_spend_table>
  WHERE td_interval(<spend_datetime_column>, '<period>')
  GROUP BY 1
),
revenue AS (
  SELECT campaign_name, ROUND(SUM(total_amount), 2) AS revenue
  FROM {db}.behavior_ecommerce_orders
  WHERE td_interval(order_datetime, '<period>')
  GROUP BY 1
)
SELECT r.campaign_name, r.revenue, s.ad_spend, ROUND(r.revenue / s.ad_spend, 2) AS roas
FROM revenue r JOIN spend s ON s.campaign_name = r.campaign_name
```

Add ROAS/CTR to the "answer" sentence and the supporting table in 3c, not as a separate afterthought — e.g. "...a 6.8% increase, but click-through dropped from 11.6% to 9.9% (194/1,955 clicks)."

---

## Step 4 — Surface One Recommendation

For each answer, add one actionable next step grounded in the specific pattern found in the data — not a generic marketing platitude. Base it on what the numbers actually show:

- Click-through dropped while sends rose → suggest reviewing subject line/creative refresh, or checking for list fatigue in the segment
- Unsubscribe rate spiked in a specific month → suggest checking send frequency or a specific campaign's content around that date
- One segment/campaign has a meaningfully higher conversion rate → suggest reallocating budget or testing that segment's messaging on underperforming campaigns
- Revenue attribution is concentrated in a small number of campaigns → suggest a controlled test pausing or reducing spend on the long tail

If the data doesn't support a clear recommendation (e.g. flat trend, small sample size), say that explicitly rather than inventing one — a forced recommendation is worse than none for a customer-facing skill.

---

## Step 5 — Stop the Clock and Report

```python
elapsed_seconds = time.time() - question_start
```

Report elapsed time to the user at the point of answering (not just at session end) — e.g. "Answered in 4.2 seconds." This is the SC1 benchmark evidence: baseline is analyst-hours, this skill's answer is elapsed seconds, for the exact same question.

At the end of the session (or if the user asks for a summary), report the full list of questions asked and their individual elapsed times.

---

## Step 6 — Render the HTML Report

Every answer is delivered as an HTML report opened in the artifact panel via `mcp__work__preview_document` — not as plain chat text. This is the customer-facing deliverable; chat text alone does not satisfy the audit-trail requirement in a reviewable, shareable form.

**Report location:** write to `{working_directory}/uc-campaign-analysis/reports/{session_id}.html` (one file per session — append, don't overwrite, so the "full list of questions" from Step 5 stays in one place). Do not write to `/tmp` — write tools are restricted to the working directory in this environment.

**On the first question in a session:** write a new HTML file with a header (parent segment name, session start time) and one "answer card" for the question.

**On each subsequent question:** read the existing file, append a new answer card before the closing tags, and rewrite it. Never regenerate old cards — they are the audit trail.

**Each answer card must contain, in this order** (mirrors Step 3c/3d and Step 4 — the report is not a new set of content requirements, it's a rendering of what was already produced):
1. The question as asked, and the elapsed time badge (e.g. "4.2s") from Step 5
2. The plain-English answer sentence
3. The supporting data table (raw counts, not just the derived rate)
4. CTR/ROAS row, when applicable per Step 3d
5. The one recommendation from Step 4 (or an explicit "no clear recommendation" note)
6. The full SQL query in a `<pre><code>` block — collapsed/expandable is fine, but never omitted

**Structure guidance:**
- Plain HTML + inline `<style>`, no external assets, no JS framework — this must render standalone when opened via `preview_document` and remain a static artifact the user can save or share
- One card per question, most recent first or in chronological order — pick one and stay consistent within a session
- Use color/badges sparingly to flag the hard-stop case (Step 1) or a missing-dimension case distinctly from a normal answer card, so a scan of the report shows successes vs. gaps at a glance

After writing/updating the file, call `preview_document` with the file path so the user sees it immediately — do this after every question, not just at session end, so the panel stays current.

If the hard-stop condition in Step 1 fires before any question is answered, still render a minimal report containing just that hard-stop message — don't skip the report because there's no data to show.

---

## Step 7 — Log Invocation

Get the current user and account from `tdx auth`, then INSERT into the shared tracking table. Log once per question, not once per session.

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
VALUES ('<uuid>', '<user_email_from_tdx_auth>', '<account_id_from_tdx_auth>', 'uc-campaign-analysis', <unix_timestamp>)
```

The `account_id` groups all runs for the same TD console together — use the numeric account ID from `tdx auth` output (e.g. `61`), not the parent segment name.

If the INSERT fails:
> "Usage tracking failed — `ai_usage.skills_usage_tracker` not writable in this environment. Answer delivered successfully."

Always report the outcome — never skip silently. Log once per question, not once per session.

---

## Acceptance Criteria (ARCH-1324)

- [ ] Works against any TD database containing campaign data — no hardcoded table or campaign names (Step 1's discovery-by-column, not by name, satisfies this)
- [ ] Always shows the SQL query used alongside the answer — full audit trail, no black box (Step 3c, non-negotiable)
- [ ] Time-to-answer recorded per question and surfaced to the user (Step 2 + Step 5)
- [ ] Hard stop with a clear message if no campaign data is found in TD (Step 1's feasibility gate)
- [ ] CTR surfaced alongside conversion metrics for email/digital campaign answers; ROAS surfaced when a spend table exists, otherwise explicitly flagged as unavailable (Step 3d — added 2026-06-26 per Eurowings/Frankfurt bootcamp feedback)
- [ ] Every answer is rendered as an HTML report opened via `preview_document`, not left as plain chat text (Step 6)

---

## What NOT to do

- Do NOT hardcode campaign names, table names, or a fixed schema — discover by column presence (`campaign_name`/`campaign_id`), per segment, per session
- Do NOT answer a question about a dimension that doesn't exist in the schema (e.g. acquisition channel, ad spend) by silently substituting a different dimension — say what's missing and offer the closest available alternative
- Do NOT filter or group on a behavior table's generic `time` column — it is frequently a constant ingestion timestamp, not the event time; use the semantic per-event column instead
- Do NOT omit CTR from an email/digital campaign answer just because the user didn't ask for it by name — surface it by default (Step 3d)
- Do NOT silently drop ROAS if no spend table exists — say so explicitly, same as any other missing dimension
- Do NOT show a rate without its raw numerator/denominator counts
- Do NOT invent a recommendation when the data doesn't support one
- Do NOT skip showing the SQL, even for a simple one-line answer
- Do NOT answer only in chat text — every answer must be rendered into the HTML report and opened via `preview_document` (Step 6)

## Related Skills

- **parent-segment-analysis** — schema discovery workflow this skill extends; **do not modify**
- **trino** — TD Trino SQL syntax and functions
- **time-filtering** — `td_interval` / `td_time_string` patterns for "last month," "Q1," "March vs February" style time windows
- **uc-campaign-brief-generator** — for planning a new campaign, not analyzing an existing one

## POC Context

- **Jira:** ARCH-1324
- **Tier:** 2 — Use Case (UC6, Campaign Performance Analysis)
- **Phase:** Weeks 4–6 skill build, then self-serve period
- **Value pillar:** Operational Efficiency — reporting hours saved (analyst hours → seconds)
- **SC1 benchmark:** "How did our March campaign perform compared to February?" — baseline = analyst hours, after = seconds (Step 5 evidence)
- **Repo:** `treasure-data/ai-poc-framework` — `skills/uc-campaign-analysis/SKILL.md`
- **Do NOT modify** `tdx-skills:parent-segment-analysis` — this skill extends it, not replaces it
