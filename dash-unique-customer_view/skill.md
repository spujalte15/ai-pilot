---
name: dash-unique-customer-view
description: Use when building or running the Unique Customer View (360 profile) dashboard for a POC or trial. Delivers a complete individual customer profile — identity, subscriptions, email engagement, site visits, purchase history, journey membership, and RFM tier — as a tabbed HTML dashboard with full data transparency (source table, SQL, freshness shown per tab). Triggers on: "unique customer view", "customer 360", "customer profile dashboard", "look up a customer", "show me a customer", "UCV", "dash-unique-customer-view", "360 profile", "customer lookup dashboard".
---

# dash-unique-customer-view — 360 Customer Profile Dashboard

Build a complete individual customer 360 profile as a tabbed HTML dashboard. The search happens **in chat** — the dashboard is a pure results viewer with no search UI.

**Source:** ARCH-1328 | **Team:** Marketing Ops / CRM | **Priority:** P2

---

## Interaction Flow

1. **Ask the user for the parent segment name** — plain text in chat: "What is the parent segment name?"
2. Run schema discovery using that segment name
3. Ask the user which attribute to search by (use `AskUserQuestion`): `email`, `td_id`, `cdp_customer_id`, `account_id`, `username`
4. Ask for the identifier value in chat
5. Resolve to `cdp_customer_id`, handle duplicates
6. Run all behavior queries in parallel
7. Render the dashboard as a self-contained HTML file (clientelling style) and open with `mcp__work__open_file`

---

## Step 1 — Schema Discovery

Run once per session when the parent segment is known. **Never hardcode column or table names.**

```bash
tdx ps desc "<parent_segment_name>" -o /tmp/ps_schema.json
```

```python
import json
with open('/tmp/ps_schema.json') as f:
    schema = json.load(f)

db = schema['database']
customer_cols = [c['name'] for c in schema['customers']['columns']]
behavior_tables = [b['table'] for b in schema['behaviors'] if len(b.get('columns', [])) > 0]
```

**Cross-reference with actual DB tables** — `tdx ps desc` under-reports behavior tables:

```bash
tdx tables "{db}.*" --json 2>/dev/null
```

For any table in the DB not in schema JSON (or with 0 columns), run `tdx describe {db}.{table}` to verify real columns. If describe succeeds, include it.

---

## Step 2 — Customer Lookup (in chat)

After schema discovery, inspect the actual customer columns to find available identifier columns:

```python
# From the schema, find identifier columns present in this segment
IDENTIFIER_CANDIDATES = ['email', 'td_id', 'cdp_customer_id', 'account_id', 'username']
available_identifiers = [c for c in IDENTIFIER_CANDIDATES if c in customer_cols]
```

Then ask the user using `AskUserQuestion` with **only the identifiers that actually exist** in this segment's customers table. Do NOT hardcode or show identifiers that are not in the schema.

**Do NOT offer `phone_number`** even if present — non-unique, collision-prone.

```sql
SELECT cdp_customer_id, email, first_name, last_name, city, td_country_name, rfm_segment
FROM {db}.customers
WHERE {attr} = '{value}'
LIMIT 10
```

- 0 rows → tell user, ask to retry
- 1 row → proceed
- >1 rows → show numbered pick-list in chat, wait for user selection

**Never silently use LIMIT 1** when multiple rows are returned.

---

## Step 3 — Pull All Profile Data

Run all queries in parallel. Use the resolved `cdp_customer_id`.

### Identity & RFM

```sql
SELECT cdp_customer_id, td_id, email, first_name, last_name, city, td_country_name,
  gender, age, profile_type, salutation, street_address, postcode, phone_number,
  email_consent, phone_consent, mail_consent, is_blacklisted,
  next_best_product, next_best_vendor, next_best_product_type, next_action,
  recency, frequency, monetary_value, rfm_score, rfm_segment, rfm_cluster_rank,
  r_quartile, f_quartile, m_quartile,
  td_affinity_categories, td_last_access_date, create_datetime
FROM {db}.customers
WHERE cdp_customer_id = '<id>'
LIMIT 1
```

### Email Engagement

```sql
-- Always describe table first to get real column names
SELECT campaign_name, send_datetime, platform FROM {db}.behavior_email_send WHERE cdp_customer_id='<id>' ORDER BY time DESC LIMIT 20
SELECT campaign_name, open_datetime, platform FROM {db}.behavior_email_open WHERE cdp_customer_id='<id>' ORDER BY time DESC LIMIT 20
SELECT campaign_name, click_datetime, url FROM {db}.behavior_email_click WHERE cdp_customer_id='<id>' ORDER BY time DESC LIMIT 20
```

### In-Store Visits

```sql
SELECT store_entry_datetime, store_name, store_id, purchase, platform, device_type
FROM {db}.behavior_instore_visits
WHERE cdp_customer_id = '<id>'
ORDER BY time DESC LIMIT 20
```

### Purchase History

```sql
-- Online orders
SELECT order_datetime, total_amount, currency, campaign_name, fulfillment_status, delivery_method
FROM {db}.behavior_ecommerce_orders
WHERE cdp_customer_id = '<id>'
ORDER BY time DESC LIMIT 20

-- In-store orders
SELECT order_datetime, total_amount, currency, campaign_name, payment_method
FROM {db}.behavior_store_orders
WHERE cdp_customer_id = '<id>'
ORDER BY time DESC LIMIT 20
```

### Journey History

```sql
-- Only query if a behavior table matching *journey* exists with > 0 columns
-- Skip entirely if no journey table found in schema
SELECT <journey_cols>
FROM {db}.<behavior_journey_table>
WHERE cdp_customer_id = '<id>'
ORDER BY time DESC LIMIT 10
```

Common journey columns: `journey_name`, `entry_datetime`, `stage_name`, `conversion`, `exit_datetime`

### Subscriptions

```sql
-- Only query if a behavior table matching *subscription* exists with > 0 columns
-- Skip entirely if no subscription table found in schema
SELECT <subscription_cols>
FROM {db}.<behavior_subscription_table>
WHERE cdp_customer_id = '<id>'
ORDER BY time DESC LIMIT 10
```

Common subscription columns: `product_name`, `brand`, `tier`, `subscription_status`, `start_date`, `end_date`, `renewal_date`

If no dedicated subscription table exists, surface active products/brands/tiers from the `customers` table columns (e.g. `next_best_product`, `next_best_vendor`, `next_best_product_type`) and note the source. Do NOT show the Subscriptions tab if both sources return 0 rows.

### Loyalty

```sql
SELECT member_type, membership_number, member_status, enrollment_date
FROM {db}.behavior_loyalty
WHERE cdp_customer_id = '<id>'
LIMIT 10
```

### SMS & Push

```sql
SELECT send_datetime, campaign_name, platform FROM {db}.behavior_sms_send WHERE cdp_customer_id='<id>' ORDER BY time DESC LIMIT 20
SELECT app_push_send_datetime, campaign_name, platform FROM {db}.behavior_app_push WHERE cdp_customer_id='<id>' ORDER BY time DESC LIMIT 20
```

### Social & Reviews

```sql
SELECT post_datetime, sentiment, message_type, analyzer_score FROM {db}.behavior_social_posts WHERE cdp_customer_id='<id>' ORDER BY time DESC LIMIT 20
SELECT review_datetime, product_name, stars FROM {db}.behavior_product_review WHERE cdp_customer_id='<id>' ORDER BY time DESC LIMIT 20
```

---

## Step 4 — Data Freshness (REQUIRED on every tab)

For every behavior table used, query the actual latest event timestamp using the semantic column (e.g., `order_datetime`, `send_datetime`). **Do NOT use the generic `time` column** — it is an ingestion timestamp, not the event time, and is often constant across all rows. **Do NOT use today's date.**

```sql
SELECT from_unixtime(MAX(<event_datetime_column>)) as last_updated
FROM {db}.<behavior_table>
```

For the `customers` table: use note `"Refreshed on parent segment last run"` — the `time` column contains unreliable values.

---

## Step 5 — Transparency Footer (REQUIRED on every tab)

**This is a hard requirement from ARCH-1328 (Sébastien Pujalte):**
> "Never present a customer attribute without showing which table it came from."

Every tab in the dashboard **must** show at the bottom:
- **Source table:** exact `{db}.{table}` name
- **Last updated:** result of `from_unixtime(MAX(<event_datetime_column>))` using the semantic column per the Known Schema Patterns table — NOT the generic `time` column and NOT today's date
- **SQL used:** collapsible block with the exact query run

This applies to ALL tabs including Overview (source: `customers` table).

---

## Step 6 — Render the Clientelling Dashboard

The dashboard is a **clientelling app** — it should feel like a sales rep briefing card, not a BI report. The goal is to give a sales advisor or CRM user everything they need to have a meaningful, personalised conversation with this customer, at a glance.

Generate a self-contained HTML file at `/Users/gandhi.yellapu/Downloads/ai_pilot/ucv.html` (write it fresh each time — do not inject data into a pre-existing template). Open it with `mcp__work__open_file` after writing.

### Visual style (clientelling reference)

Follow the pattern from the reference HTML (`ucv_antoine_pierucci_20260716_0844_1.html`):

- **Header:** full-bleed gradient banner (`linear-gradient(135deg,#1E3A5F,#2563EB)`) with a large avatar circle (initials), customer name, title/role/location, status badges (engagement tier, segment, consent status), and a prominent lead/RFM score in the top-right corner alongside key CRM metadata (owner, last activity, created date).
- **Tabs:** CSS-only radio button tab switcher (no JavaScript required) — 5–8 tabs depending on data available.
- **Cards:** white rounded cards with a subtle box-shadow; card titles use a left blue accent bar (`::before` with `width:3px; background:#2563EB`).
- **KPI row:** 4-column grid of centered KPI tiles at the top of the first tab.
- **Insight boxes:** left-border color-coded callouts — blue (neutral), amber (warning), green (success) — with a bold title line and body text.
- **Tables:** clean, compact, alternating row shading (`#f8fafc`), uppercase small-caps headers.
- **Tags:** small rounded pills (`.tag-green`, `.tag-red`, `.tag-amber`, `.tag-blue`, `.tag-gray`, `.tag-purple`).
- **Progress bars:** labeled horizontal bars for engagement/score dimensions.
- **Timeline:** dot + text list for recent activity, dots color-coded by event type.
- All layout via CSS Grid; responsive with `max-width:1200px; margin:0 auto`.

### Tab structure

| Tab | Content | Show when |
|---|---|---|
| **Overview** | 4 KPIs (score, orders, emails, channel) + strong intent signals + recent activity timeline | Always |
| **Profile** | Identity & contact table + commercial qualification table + predictive/RFM scores + linked account | Always |
| **Purchases** | KPI row + pipeline/order table + spend progression bars | Orders or pipeline data available |
| **Engagement** | Email + SMS + push send/open/click table + web visits table + per-brand engagement chart | Any engagement data |
| **Subscriptions** | Active products/brands/tiers table; falls back to `next_best_product`/`next_best_vendor` from customers table | Subscription behavior table OR customers columns available |
| **Loyalty** | Loyalty membership table + status badges + tier progression | Loyalty data available |
| **Journeys** | Journey membership, stage, entry/exit dates, conversion | Journey table available |
| **Consent / GDPR** | Legal basis, per-channel consent status, expiry dates, analysis & recommendations | Always (consent is always relevant) |
| **Next Actions** | Priority-tagged recommended actions table (P1/P2/P3) + CDP segment membership + 360° engagement summary bars | Always — this is the sales rep's call-to-action tab |

### Header fallback (required)
- Name present → show full name
- Name NULL, email present → show email
- Both NULL → show `ID: {cdp_customer_id.slice(0,12)}…`

Never show a blank header.

### Transparency footer (REQUIRED on every tab)
Every tab must include at the bottom:
- **Source table:** exact `{db}.{table}` name
- **Last updated:** result of `from_unixtime(MAX(<event_column>))` — NOT today's date
- **SQL used:** collapsible `<details><summary>Show SQL</summary>` block with the exact query

### Next Actions tab — clientelling actions (required)

The "Next Actions" tab is the **most important tab** for a sales user. It must contain:

1. **A priority-ranked actions table** with columns: Priority (P1 Urgent / P2 / P3), Action (plain-language instruction), Channel, Owner/Responsible, Deadline, and Rationale (specific data point that justifies this action). Generate actions from the actual data — e.g. high open rate → qualify email interest; consent refused → recommend direct call; open opportunity → close follow-up.

2. **CDP segment membership** — which segments this customer currently belongs to, with folder/segment name and status tag.

3. **360° engagement summary** — progress bars summarising each engagement dimension (email engagement %, web intent %, CRM activity %, consent coverage %) with labels and percentages.

---

## Step 7 — Usage Log

After rendering, get the current user and account from `tdx auth`, then INSERT into the shared tracking table:

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
VALUES ('<uuid>', '<user_email_from_tdx_auth>', '<account_id_from_tdx_auth>', 'dash-unique-customer-view', <unix_timestamp>)
```

The `account_id` groups all runs for the same TD console together — use the numeric account ID from `tdx auth` output (e.g. `61`), not the parent segment name.

If the INSERT fails:
> "Usage tracking failed — `ai_usage.skills_usage_tracker` not writable in this environment. Dashboard rendered successfully."

Do not silently skip — always report the outcome.

---

## Acceptance Criteria (from ARCH-1328)

- [ ] Works with any parent segment — column names discovered dynamically
- [ ] Returns a result within one session for any valid customer identifier
- [ ] Gracefully handles missing data — tab hidden if no data available
- [ ] Dashboard readable by a Marketing Ops user with no TD knowledge
- [ ] **Every tab shows source table, real data freshness, and SQL used** ← hard requirement
- [ ] Journey tab present if journey behavior table exists with data
- [ ] Duplicate identifiers → pick-list shown, never silently picks one
- [ ] `phone_number` NOT offered as lookup field
- [ ] Header never blank — degrades: name → email → cdp_customer_id
- [ ] Usage log attempted and outcome reported

---

## Known Schema Patterns (verified against Gartner France Demo / cdp_audience_465858)

| Section | Behavior table | Key timestamp column |
|---|---|---|
| Email sends | `behavior_email_send` | `send_datetime` (Unix bigint) |
| Email opens | `behavior_email_open` | `open_datetime` (Unix bigint) |
| Email clicks | `behavior_email_click` | `click_datetime` (Unix bigint) |
| In-store visits | `behavior_instore_visits` | `store_entry_datetime` (Unix bigint) |
| Online orders | `behavior_ecommerce_orders` | `order_datetime` (Unix bigint) |
| Store orders | `behavior_store_orders` | `order_datetime` (Unix bigint) |
| Loyalty | `behavior_loyalty` | `enrollment_date` (Unix bigint) |
| Subscriptions | `behavior_subscriptions` (if present — verify via `tdx tables`) | `start_date` (Unix bigint) — fallback: `next_best_product`/`next_best_vendor` from `customers` |
| SMS | `behavior_sms_send` | `send_datetime` (Unix bigint) |
| App push | `behavior_app_push` | `app_push_send_datetime` (Unix bigint) |
| Social | `behavior_social_posts` | `post_datetime` (Unix bigint) |
| Reviews | `behavior_product_review` | `review_datetime` (Unix bigint) |
| App sessions | `behavior_app_session` | `app_session_start_datetime` (Unix bigint) — found via `tdx tables`, absent from schema JSON |
| Engagement score | `behavior_engagement` | `time` — cols: `engagement_tier`, `last_engaged_channel` — surface as badge on Overview |

**Identifier reference:**

| Identifier | Coverage | Unique? | Use as lookup? |
|---|---|---|---|
| `cdp_customer_id` | 100% | YES | Primary key after resolution |
| `td_id` | 100% | YES | Best user-facing lookup |
| `email` | ~64% | NO | Show pick-list if >1 result |
| `account_id` | ~63% | Not verified | Show pick-list if >1 result |
| `username` | ~63% | Not verified | Show pick-list if >1 result |
| `phone_number` | ~63% | NO | **Do NOT offer** |

---

## What NOT to do

- Do NOT put a search form inside the HTML — search happens in chat via `AskUserQuestion`
- Do NOT use `render_react` for this dashboard — Treasure Studio does not have a `render_react` function; always generate a self-contained HTML file and open it with `mcp__work__open_file`
- Do NOT inject data into a pre-existing template — write the full HTML fresh each invocation
- Do NOT hardcode column or table names — always discover from schema first
- Do NOT use today's date for data freshness — always query `from_unixtime(MAX(<event_column>))`
- Do NOT show a tab with 0 rows — hide it entirely

## Confirmed Working Render Pattern

Write a **single self-contained HTML file** with all data embedded directly as inline JavaScript variables. No external dependencies — all CSS and JS inline.

```html
<!DOCTYPE html><html lang="en"><head><meta charset="UTF-8">
<title>UCV — {customer_name}</title>
<style>
/* full inline CSS — follow clientelling visual style above */
</style>
</head><body>
<div class="wrap">
  <!-- HEADER (gradient banner + avatar + badges + score) -->
  <!-- TABS (CSS-only radio button switcher) -->
  <!-- TAB CONTENT (c1…c8 divs, hidden/shown by :checked selector) -->
</div>
</body></html>
```

**Rules:**
- No JavaScript framework or library — pure HTML + CSS only
- Tab switching via CSS radio button trick (`:checked ~ .wrap-c .cN { display:block }`)
- All data values interpolated directly as HTML strings — no `window.__UCV_DATA__` injection pattern
- Write the full file in one `Write` call — do not patch an existing file
- Keep total file size reasonable — prefer concise inline data over verbose formatting

---

## POC Context

- **Jira:** ARCH-1328
- **Tier:** 2 — Dashboard Build (Weeks 5–6)
- **Team:** Marketing Ops / CRM
- **SC evidence:** SC1 (time to insight), SC3 (self-serve adoption)
- **Dashboard file:** `/Users/gandhi.yellapu/Downloads/ai_pilot/ucv.html`
- **Parent segment tested:** Gartner France Demo (`cdp_audience_465858`)
- **Repo:** `treasure-data/ai-poc-framework` — `skills/dash-unique-customer-view/SKILL.md`
