# dash-unique-customer-view — Customer 360 Profile Dashboard

A Treasure Work skill that delivers a complete individual customer 360 profile as an interactive scrollable dashboard. Marketing Ops users can look up any customer in seconds with no SQL knowledge required.

**Jira:** ARCH-1328 | **Team:** Marketing Ops / CRM | **Priority:** P2

---

## Prompts to Invoke

Type any of the following in Treasure Work chat:

```
"Build a UCV dashboard"
"Unique customer view"
"Customer 360"
"360 profile"
"Look up a customer"
"Show me a customer"
"Customer profile dashboard"
"dash-unique-customer-view"
"UCV"
```

---

## Interaction Flow

The skill is entirely chat-driven — there is no search form in the dashboard itself.

### Step 1 — Provide the parent segment name
The skill asks which parent segment to use (e.g. `Gartner France Demo`).

### Step 2 — Choose a search attribute
The skill asks how you want to find the customer:

| Attribute | Description | Notes |
|---|---|---|
| `email` | Customer email address | May return multiple matches — pick-list shown |
| `td_id` | Numeric Treasure Data ID | Unique — fastest lookup |
| `cdp_customer_id` | Hashed UUID primary key | Unique |
| `account_id` | Account number | May return multiple matches |
| `username` | Login name | May return multiple matches |

> `phone_number` is not offered — it is non-unique and returns too many false matches.

### Step 3 — Provide the value
Type the identifier value in chat. For example:
```
natasha.evans@nash.com
```
or
```
49499
```

If the value matches more than one customer, a numbered pick-list is shown in chat. Pick a number to continue.

### Step 4 — Dashboard renders automatically
The skill runs ~12 parallel queries, then renders the full 360 profile in the artifact panel.

---

## What the Dashboard Shows

A single scrollable page with the following sections:

### Header
- Full name (or email / CDP ID if name is null)
- Email, city/country, age, gender
- Consent badges (Email ✓/✗, Phone ✓/✗, Mail ✓/✗)
- RFM segment badge, profile type

### Overview
- **KPI cards:** RFM Segment, Lifetime Value, Total Orders, Recency
- **Identity:** CDP ID, TD ID, address, phone, member since, last access
- **RFM breakdown:** Score, R/F/M quartiles, cluster rank
- **Affinities:** TD-computed interest categories
- **Transparency footer:** source table + SQL + data freshness

### Engagement
- **KPI cards:** emails sent, opened (with open rate %), clicked, store visits
- **Email send history:** campaign, date, platform
- **SMS sends:** campaign, date, platform
- **Store visits:** store name, entry date, purchase flag, device type
- **Transparency footer**

### Purchases
- **KPI cards:** total orders, total spend, online vs in-store split
- **Online orders:** date, amount, campaign, fulfillment status, delivery method
- **In-store orders:** date, amount, campaign, payment method
- **Transparency footer**

### Loyalty
- Membership cards with type, number, status badge (Active / Terminated), enrollment date
- **Transparency footer**

### Social & Reviews
- **KPI cards:** social post count, review count, average star rating
- **Product reviews:** product name, star rating, date
- **Social posts:** date, sentiment, message type, analyzer score
- **Transparency footer**

### Next Action
- **Next Best Action panel:** best vendor, best product, product type
- **Engagement Risk panel:** RFM segment, risk interpretation, recency/frequency/monetary

---

## Transparency Footer (ARCH-1328 Hard Requirement)

Every section shows at the bottom:
- **Source table:** exact `{db}.{table}` name
- **Last updated:** real `from_unixtime(MAX(time))` from that table — never today's date
- **SQL used:** collapsible block with the exact query

> "Never present a customer attribute without showing which table it came from." — Sébastien Pujalte, ARCH-1328

---

## Usage Logging

After rendering, the skill attempts to log the invocation:

```sql
INSERT INTO {customer_slug}_ai_poc_tracking.skill_usage
  (skill_name, invoked_at, customer_identifier, parent_segment_name, tabs_rendered, status)
VALUES
  ('dash-unique-customer-view', <unix_timestamp>, '<identifier>', '<ps_name>', 'all', 'success')
```

If the tracking table does not exist, you will see:
> "Usage tracking is not active for this environment. Dashboard rendered successfully."

---

## Technical Details

| Item | Detail |
|---|---|
| Parent segment tested | Gartner France Demo |
| Output database | `cdp_audience_465858` |
| Search UI | In chat only — no search form in the dashboard |
| Rendering | `render_react` with all data as module-level constants (no props, no useState) |
| Timestamps | All Unix bigint timestamps formatted to `DD Mon YYYY` |
| Duplicate handling | Pick-list shown in chat when >1 record matches |
| Journey tab | Hidden if no journey behavior table exists in the segment |

---

## Example Session

```
User:   "Build a UCV dashboard"

Skill:  "What is the parent segment name?"

User:   Gartner France Demo

Skill:  [runs schema discovery]
        "Which attribute would you like to search by?"
        [email] [td_id] [cdp_customer_id] [account_id]

User:   email

Skill:  "What is the email address?"

User:   natasha.evans@nash.com

Skill:  [runs 12 parallel queries]
        [renders Customer 360 dashboard in artifact panel]
        "Usage tracking not active for this environment."
```

---

## Acceptance Criteria (ARCH-1328)

- [x] Works with any parent segment — column names discovered dynamically
- [x] Returns a result within one session for any valid customer identifier
- [x] Gracefully handles missing data — section hidden if no data available
- [x] Dashboard readable by a Marketing Ops user with no TD knowledge
- [x] Every section shows source table, real data freshness, and SQL used
- [x] Duplicate identifiers → pick-list shown, never silently picks one
- [x] `phone_number` not offered as a lookup field
- [x] Header never blank — degrades: name → email → cdp_customer_id
- [x] Usage log attempted and outcome reported

---

## Known Limitations

- No journey tab for Gartner France Demo — no journey behavior table in this segment
- `render_react` with `data` prop or `useState` causes blank screen — skill uses module-level constants pattern instead
- HTML file approach not used — artifact panel blocks script execution
- `phone_number` excluded from lookup — non-unique in synthetic/demo datasets
