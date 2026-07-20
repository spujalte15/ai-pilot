---
name: uc-campaign-brief-generator
description: Use when the user wants to create a campaign brief, marketing brief, creative brief, or campaign plan. Triggers on: "create a campaign brief", "generate a brief", "campaign planning", "write a marketing brief", "help me brief my campaign", "I need a campaign plan", "draft a brief for my campaign". Covers audience targeting, messaging, channel strategy, creative direction, timelines, and optional TD Audience Studio segment recommendations. Works with or without a TD data connection.
---

# Campaign Brief Generator

Generate a structured, agency-ready campaign brief from scattered marketing inputs. Works with or without a TD data connection — pure-prompt mode produces a complete, marketer-usable brief every time.

## Step 1 — Intake

Ask the user for the following inputs. Collect all in one message if not already provided:

- **Campaign goal** — what business outcome or KPI should this campaign drive?
- **Target audience** — describe who you're trying to reach (demographics, behaviors, segments, or a plain-language description)
- **Key products or offers** — what are you promoting?
- **Channels** — which channels are in scope? (email, paid social, SMS, push, display, etc.)
- **Budget** — optional; include if available
- **Timeline** — campaign start/end dates or key milestones
- **Constraints or brand guidelines** — any restrictions (tone, regulatory, brand voice, exclusions)?

If the user has already provided some of these, skip asking for them. Do not ask for optional fields (budget) if not volunteered.

### Sample audience descriptions (use these if the user wants to test without real data)
- "High-value loyalty members who haven't purchased in 90 days"
- "Mobile app users who browsed but didn't convert in the last 30 days"
- "Email subscribers aged 25–44 with household income over $75K"

## Step 2 — Generate structured brief

Produce a complete campaign brief with all 7 sections. Never leave a section blank — if information is missing, make a reasonable marketer-grade assumption and note it.

### Brief template

```
## Campaign Brief: [Campaign Name]
**Prepared:** [today's date]
**Status:** Draft

---

### 1. Campaign Objective & Success Metric
[One clear objective statement. One primary KPI with target if available.]

### 2. Target Audience
**Who:** [Segment name or description]
**Why this audience:** [Why they are the right target for this campaign]
**What they care about:** [Key motivations, pain points, or desires]
**Estimated size:** [If TD parent segment is available, query result. Otherwise: "Not available — TD connection not configured."]

### 3. Key Message & Value Proposition
**Core message:** [Single sentence a customer should remember]
**Value proposition:** [Why this offer matters to them specifically]
**Supporting proof points:**
- [Point 1]
- [Point 2]

### 4. Creative Direction
**Tone:** [e.g., Urgent, Warm, Aspirational, Informative]
**Format guidance:**
- Email: [subject line approach, CTA placement, hero image concept]
- [Other channels in scope: channel-specific guidance]

### 5. Offer & Call to Action
**Primary offer:** [What the customer receives]
**CTA copy:** [Exact or suggested CTA text]
**Landing page / destination:** [If known]

### 6. Timeline & Key Milestones
| Milestone | Date |
|-----------|------|
| Brief approved | [date] |
| Creative assets due | [date] |
| QA / review | [date] |
| Campaign launch | [date] |
| Campaign end / analysis | [date] |

### 7. Suggested Audience Studio Segment
[See Step 3 below]
```

## Step 3 — Audience Studio segment recommendation

**If a TD parent segment is available:**
- Map the audience description to parent segment fields and behavior tables
- **Before writing segment YAML, verify against the live schema — do not guess field names or values:**
  - Run `tdx sg fields` and use the bare column name as the `attribute` (e.g. `r_quartile`, not `rfm.r_quartile` — the attribute group prefix is for display only, not the YAML key)
  - For any categorical field used in an `Equal`/`In`/`NotIn` condition, run a quick `tdx query "SELECT <field>, COUNT(*) FROM <audience_table> GROUP BY <field>"` to confirm actual values and casing (e.g. `'Yes'/'No'` not `'true'/'false'`, `'Active'` not `'active'`) — demo and customer data rarely match assumed enum values
  - Validate with `tdx sg validate` and a `tdx sg push --dry-run` before the real push; after pushing, run `tdx sg show "<name>"` and confirm it returns a nonzero count before presenting the brief as final
- Show the suggested segment rule YAML using `tdx sg` syntax
- Show the query or field references that informed the recommendation
- Include estimated audience size if queryable

**If no TD connection is available (pure-prompt mode):**
- Describe what the segment would look like in plain English
- List which parent segment attributes and behavior tables would be relevant (e.g., `last_purchase_date`, `loyalty_tier`, `app_event_type`)
- Note: "To build this segment in Audience Studio, connect a TD parent segment and run `tdx sg push`."

Always make this section actionable — never leave it as "N/A".

## Step 4 — Output format

Render the brief as formatted markdown in chat **and** write a self-contained HTML file to `/Users/gandhi.yellapu/Downloads/ai_pilot/campaign-brief-{campaign-slug}.html`, then open it with `mcp__work__preview_document`. This makes the brief visible as a shareable artifact, not just chat text.

If the SE or user asks for a PPTX / slide version, use `document-skills:pptx` to produce slide content with one slide per brief section (header + 3–5 bullet points per slide).

## Step 5 — Log invocation

After producing the brief, get the current user and account from `tdx auth`, then INSERT into the shared tracking table:

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
VALUES ('<uuid>', '<user_email_from_tdx_auth>', '<account_id_from_tdx_auth>', 'uc-campaign-brief-generator', <unix_timestamp>)
```

The `account_id` groups all runs for the same TD console together — use the numeric account ID from `tdx auth` output (e.g. `61`), not the customer slug.

If no TD connection is active or the INSERT fails:
> "Usage tracking skipped — not writable in this environment. Brief delivered successfully."

## Acceptance criteria (built-in quality checks)

Before presenting the brief, verify:
- All 7 sections are populated — no blank sections
- The brief reads as a finished, marketer-usable document (not a skeleton)
- If a parent segment is in scope, Section 7 references real TD fields with the query shown
- The brief can be produced in under 3 minutes from first input

## Edge cases

- **No TD connection:** Skip Section 7 query, describe the segment in plain English, note how to activate it. Produce the rest of the brief normally.
- **Minimal input (user just says "create a brief"):** Use the sample audience descriptions to demo the skill, clearly labeling it as a sample.
- **Multiple audiences:** Produce a separate audience block in Section 2 for each, and a separate Section 7 segment per audience.
- **Compliance-restricted customers:** Do not ask for or reference real PII. Use segment-level descriptions only.
