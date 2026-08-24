---
name: business-glossary-build
description: Use when the user asks to define business terms, metrics, KPIs, events, customer taxonomies, abbreviations, identity semantics, governance rules, or mappings to Treasure Data. Runs a progressive requirements interview, reconciles answers with an architecture map, records provenance and approval state, and produces an evidence-backed glossary for studio-context-file. Supports Treasure Work, Treasure AI Studio, and compatible LLM runtimes. Requires Tier 2 capabilities for Discovery and Tier 3 for Production; performs a preflight and stops with a structured exception instead of guessing.
---

# Business Glossary Build

Conduct a requirements interview and produce `[company]-glossary.md`. Capture the user's language faithfully, but do not treat an unreviewed answer as approved ground truth. Surface disagreements and unknowns rather than resolving them silently.

## 1. Choose Scope and Mode

Use **Discovery** for a short first draft and **Production** for governed organizational context.

| Mode | Interview depth | Required participants | Allowed readiness |
|---|---|---|---|
| Discovery | Core users, terms, one KPI, primary mappings, known gotchas | Requester; owners may be TBD | Draft |
| Production | All applicable blocks, examples/counterexamples, identity, governance, metric semantics, owners and approval | Business and data owners; governance owner where relevant | Pilot-ready input |

If the user does not choose, recommend Discovery and state that its output cannot be production-ready.

## 2. Run the Execution Preflight

Record runtime, active model if exposed, mode, available source artifacts, artifact read/write ability, multi-turn state, and confirmation capability.

Require the model to demonstrate that it can:

- Maintain a structured interview state across turns.
- Ask one question or one coherent block at a time.
- Distinguish observed evidence, participant statements, inference, conflict, and unknowns.
- Read the architecture artifact without dropping evidence references.
- Play answers back and apply corrections.
- Preserve unresolved contradictions.
- Produce structurally valid Markdown.

Use these capability tiers:

| Tier | Suitable use |
|---|---|
| 1 — Basic | Format an already approved glossary only |
| 2 — Standard | Discovery interview and one architecture artifact |
| 3 — Advanced | Production interview, cross-artifact reconciliation, conflicts, governance |
| 4 — High-assurance | Sensitive, multi-market, highly regulated, or independently validated contexts |

Require Tier 2 for Discovery and Tier 3 for Production. The runtime or user may supply a trusted model-tier classification. Otherwise, evaluate only capabilities demonstrated during preflight; do not claim hidden self-knowledge about the model. If the required capability cannot be demonstrated, stop with `MODEL_CAPABILITY_INSUFFICIENT`. If a concrete recommendation is needed, ask the runtime for an available model meeting the required tier; do not depend on a vendor or model family.

If the model loses state, merges contested answers, invents mappings, or cannot read the inputs, set `BLOCKED`; do not “try harder” by fabricating a complete artifact.

## 3. Load Inputs and Start the Interview State

Read `[company]-architecture.md` when available, especially:

- Scope and Limitations
- Identifiers and Join Contracts
- Schema Surprises and Gotchas
- Open Questions and Conflicts
- Glossary Candidates
- Evidence Ledger

If it is missing, recommend the full Discovery interview with unverified TD mappings. Use the Draft-only quick capture only when the user cannot complete the full interview. Production mode requires the architecture artifact or equivalent validated evidence.

If artifacts originate in different runtimes, retain the source runtime on evidence records and flag runtime-specific assumptions for review.

Maintain this state throughout the interview:

```text
SESSION
- Company:
- Mode:
- Runtime/model:
- Intended users/personas:
- Priority tasks and outputs:
- Success criteria:
- Participants and roles:
- Required approvers:
- Architecture source/version:

TERMS
- term_id:
- name:
- status:
- open_fields:

CONFLICTS
- conflict_id:
- claims:
- owners:
- blocking:

GATES
- completed:
- outstanding:
```

Do not expose raw internal reasoning. Show concise state summaries and open items to the user.

## 4. Apply Evidence and Validation States

Use evidence classes:

- `OBSERVED` — directly supported by TD output or a source artifact.
- `USER_CONFIRMED` — explicitly confirmed during playback.
- `INFERRED` — proposed interpretation, not approved.
- `CONTESTED` — conflicting claims.
- `UNKNOWN` — not established.

Use a separate validation status:

- `PROPOSED`
- `CONFIRMED_BY_BUSINESS`
- `CONFIRMED_BY_DATA_OWNER`
- `CONTESTED`
- `TBD`
- `APPROVED_POLICY` — confirmed by the authorized governance/policy owner or an approved policy source

Confidence is not approval. A high-confidence inference remains `PROPOSED` until an authorized owner confirms it.

Classify each unresolved item separately as:

- `BLOCKING` — it affects an in-scope use case's definition, metric, identity/join correctness, governance, safety, or required approval, and no safe defer/qualify behavior exists.
- `NON_BLOCKING` — it can be deferred without making the in-scope result incorrect or unsafe; record the limitation and owner.
- `NOT_APPLICABLE` — it does not apply to the stated scope; record the owner-confirmed reason.

`TBD` is a validation state; it is not automatically blocking. `CONTESTED` is blocking only when it meets the definition above.

## 5. Conduct the Progressive Interview

Ask one question or one coherent block at a time. After each block:

1. Summarize what was captured.
2. Identify assumptions and gaps.
3. Ask the participant to confirm or correct the playback.
4. Update evidence and validation state.
5. Continue only after recording the response.

Do not dump every question at once. Skip a block only when it clearly does not apply, and record why.

### Block 0 — Session setup

Capture:

- Company/business unit and scope.
- Discovery or Production mode.
- Participants, roles, and which domains they can approve.
- Intended users/personas of the resulting context.
- Top tasks, desired outputs, and decisions the context should support.
- Observable success criteria and unacceptable failure modes.
- Markets, products, and use cases in and out of scope.

### Block 1 — Users and goals

For each persona, ask for:

- Questions or workflows they expect the assistant to handle.
- Required level of detail and preferred output.
- Decisions they will make from the answer.
- One successful example and one harmful or misleading answer.

### Block 2 — Core business concepts

In Discovery mode, prioritize conversion or the equivalent outcome, primary customer/entity types, segment taxonomy, the primary KPI, and candidates relevant to the stated use cases. In Production mode, cover conversion, customer/entity types, segment taxonomy, campaign, journey, use case, active customer, churn/at-risk, and every applicable architecture Glossary Candidate; record owner-confirmed reasons for items marked not applicable.

For each architecture Glossary Candidate, present only its observed technical usage and ask the participant for its business definition. Do not propose a meaning from the name. Record rejected candidates as `not a business term` with participant and date.

For each important term, obtain:

- Exact term, aliases, and definition in the participant's words.
- What it explicitly does **not** mean.
- Positive example and counterexample.
- Inclusion and exclusion rules.
- Scope and market variation.
- TD mapping or `TBD`.
- Business owner, data owner, and approver.

Do not infer a definition from a table or column name.

### Block 3 — Metrics and scores

For every KPI, metric, score, or tier, capture:

- Business question it answers.
- Numerator and denominator.
- Grain: person, account, event, campaign, market, day, or other.
- Calculation formula and aggregation behavior.
- Time window and comparison period.
- Inclusion, exclusion, deduplication, and null rules.
- Attribution model/window where applicable.
- Currency and conversion policy.
- Timezone and reporting cutoff.
- Late-arriving data and restatement behavior.
- Market/product variation.
- Source table/column/query and owner.
- Thresholds, tiers, calibration date, and model version for scores.
- Worked example and counterexample.

A metric without a denominator, grain, or time window is incomplete. Mark it blocking when it drives production decisions.

### Block 4 — Data, identity, and joins

Reconcile business semantics with the architecture artifact:

- Entity represented by each identifier.
- Canonical key versus source-system keys.
- Household/account/person/device/product relationships.
- Join direction and expected cardinality.
- Deduplication and merge/split rules.
- Market/source boundaries.
- Identity confidence and known over-/under-stitching.
- Validity dates and late-arriving identity changes.

Do not approve technical claims that contradict architecture evidence. Create a conflict instead.

### Block 5 — Market and organizational variation

Capture:

- Active and planned markets/business units.
- Abbreviations and internal codes.
- Terms or metrics that vary by market.
- Local currency, timezone, language, regulation, and data availability.
- Whether variations are exceptions, overrides, or separate definitions.

### Block 6 — Governance and permitted use

Capture:

- PII and sensitive categories.
- Consent fields, legal basis, suppression rules, and activation restrictions.
- Data residency or market restrictions.
- Role-based visibility or restricted audiences.
- Prohibited joins, analyses, inferences, and exports.
- Retention/deletion requirements.
- Escalation owner for uncertain governance questions.

Never turn uncertain legal or privacy guidance into a hard policy. Mark it `TBD` and require the appropriate owner.

### Block 7 — Failure modes and negative assumptions

Ask what a smart but uninformed analyst or assistant would get wrong. Capture:

- Misleading names and deprecated fields.
- Generic definitions that do not apply.
- Unsafe joins or row-multiplication traps.
- Unsupported markets or use cases.
- Questions the assistant must refuse, qualify, or escalate.
- Expected unknown-handling behavior.

### Block 8 — Playback and approval

Present a concise playback containing:

- Intended users and success criteria.
- Confirmed definitions and mappings.
- Proposed and contested items.
- Governance restrictions.
- Readiness blockers and named owners.

Ask each participant to confirm only the domains they own. Record approver, decision, and date. Do not interpret silence as approval.

### Block 9 — Acceptance scenarios

Collect realistic prompts covering:

- Definition retrieval.
- Negative interpretation.
- Schema/table selection.
- Market variation.
- Metric calculation.
- Join safety.
- Unknown handling.
- Governance behavior.
- Each priority persona/use case.

For each prompt, capture expected elements and prohibited elements. `studio-context-file` converts these into executable acceptance tests.

## 6. Capture Each Term

Use this record:

```text
TERM_ID: GLO-###
TERM: [exact name]
TYPE: metric | segment | event | entity | identifier | abbreviation | score | process | policy
DEFINITION: [participant's words]
NOT: [explicit counter-definition]
ALIASES: [list]
EXAMPLE: [positive example]
COUNTEREXAMPLE: [negative example]
CALCULATION: [full metric semantics or N/A]
TD MAPPING: [database.table.column/query/TBD]
IDENTITY/JOIN SEMANTICS: [if applicable]
MARKET VARIATION: [details]
GOVERNANCE: [restrictions]
BUSINESS OWNER: [name/team/TBD]
DATA OWNER: [name/team/TBD]
APPROVER: [name/team/TBD]
EVIDENCE CLASS: OBSERVED | USER_CONFIRMED | INFERRED | CONTESTED | UNKNOWN
VALIDATION STATUS: PROPOSED | CONFIRMED_BY_BUSINESS | CONFIRMED_BY_DATA_OWNER | APPROVED_POLICY | CONTESTED | TBD
RESOLUTION CLASS: BLOCKING | NON_BLOCKING | NOT_APPLICABLE
CONFIDENCE: high | medium | low
SOURCES: [architecture fact IDs, participant/date, artifact section]
SOURCE RUNTIME: [runtime]
NOTES: [limits and follow-up]
```

Preserve source wording, but add a clearly labelled normalized definition when retrieval clarity requires it. Never silently replace the source wording.

## 7. Resolve Gaps and Conflicts

For each contradiction:

1. Assign a conflict ID.
2. Quote or accurately summarize both claims and sources.
3. Determine whether scopes differ by market, time, persona, or system.
4. Ask the designated owner to resolve it during the current session or by an agreed review date.
5. If the owner is unavailable, assign a fallback domain owner and review date; do not wait indefinitely.
6. Record the resolution and approval, or leave it `CONTESTED` with `BLOCKING` or `NON_BLOCKING` classification.

Never choose the most senior participant's answer automatically. Never allow a contested item to become a normative rule in the context file. A contested item may appear only in the Conflict Register with temporary behavior until resolved.

## 8. Produce `[company]-glossary.md`

Use this structure:

```markdown
# [Company] — Business Glossary

- Version/date:
- Mode: Discovery | Production
- Readiness: Draft | Pilot-ready input | Blocked
- Participants and domains:
- Intended users/personas:
- Business owner / data owner / governance owner:
- Required approvers:
- Architecture source/version:

## Scope, Goals, and Success Criteria
## Persona and Use-Case Requirements
## Core Terms
[Full GLO records in readable Markdown]
## Metrics and Scores
[Include complete calculation semantics]
## Identity and Join Semantics
## Market and Organizational Variations
## Governance and Permitted Use
## What the Assistant Must Not Assume
## Conflicts
| ID | Claims | Sources | Owner | Blocking | Status |
|---|---|---|---|---|---|
## Unresolved Items
| Item | Resolution class | Owner | Review date/next step |
|---|---|---|---|---|
## Acceptance Scenarios
| ID | Persona | Prompt | Expected elements | Prohibited elements |
|---|---|---|---|---|
## Source and Evidence Register
## Approval Record
| Domain | Approver | Decision | Date | Scope/notes |
|---|---|---|---|---|
## Change Log
```

## 9. Apply Readiness Gates

**Draft** requires:

- Session scope and intended users.
- At least one core term and priority use case.
- Unknowns and conflicts explicitly listed.

**Pilot-ready input** additionally requires:

- Conversion/customer taxonomy/primary KPI where applicable.
- Complete semantics for decision-driving metrics.
- Primary identity and join semantics reconciled with architecture.
- Market variations captured.
- Governance restrictions reviewed; production-governing rules are `APPROVED_POLICY` or remain explicitly non-normative.
- Business and data-owner playback completed.
- No unresolved item classified `BLOCKING` for the intended pilot use cases.
- Acceptance scenarios defined.

`Pilot-ready input` is an intermediate handoff status. This skill does not declare final Pilot-ready or Production-ready status. Final merge, approvals, and acceptance results are evaluated by `studio-context-file`.

## Structured Exceptions

Return this block and preserve valid interview state when work cannot continue:

```text
EXECUTION BLOCKED
Code: [stable code]
Step: [step/block]
Runtime: [runtime]
Required capability: [capability/access]
Observed limitation: [exact limitation]
Work completed: [confirmed state]
Data not collected: [remaining state]
Safe fallback: [smaller scope, source export, stronger model, or none]
Recommended model tier: [1–4]
Example model class: [optional, non-binding]
User action: [specific next action]
```

Use: `MODEL_CAPABILITY_INSUFFICIENT`, `MODEL_UNAVAILABLE`, `SOURCE_ARTIFACT_MISSING`, `SOURCE_ARTIFACT_UNREADABLE`, `CONTEXT_LIMIT_EXCEEDED`, `CONFLICT_UNRESOLVED`, `APPROVAL_REQUIRED`, `VALIDATION_FAILED`, `OUTPUT_WRITE_UNAVAILABLE`, or `RUNTIME_UNSUPPORTED`.

A draft may contain TBDs. A pilot-ready artifact may not hide blockers, unresolved decision-driving definitions, or missing approvals.
