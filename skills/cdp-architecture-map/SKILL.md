---
name: cdp-architecture-map
description: Use when the user asks to explore, map, document, or explain a Treasure Data environment, parent segments, output databases, tables, identifiers, joins, behaviors, freshness, or schema. Produces an evidence-backed architecture document and glossary candidates for business-glossary-build. Supports discovery and production modes in Treasure Work, Treasure AI Studio, and compatible tool-enabled LLM runtimes. Requires Tier 2 capabilities for Discovery and Tier 3 for Production; performs a preflight and stops with a structured exception rather than guessing when access, tools, context, or model capability is insufficient.
---

# CDP Architecture Map

Explore a live Treasure Data environment and produce `[company]-architecture.md`. Keep technical observations separate from business interpretations. Never infer business meaning from a table or column name.

## 1. Select the Operating Mode

Infer the mode from the request; if it is unclear, recommend **Discovery** and state the choice.

| Mode | Use for | Required depth | Allowed readiness |
|---|---|---|---|
| Discovery | Orientation, demo preparation, early onboarding | Primary parent segment and 2–3 representative tables | Draft |
| Production | Governed context, operational assistants, production use | Relevant parent segments, identity and join checks, quality checks, owners, provenance | Pilot-ready input |

Discovery normally takes 10–15 minutes. It is not a full audit and must not be labelled production-ready.

## 2. Run the Execution Preflight

Before querying data, record:

- **Runtime:** Treasure Work, Treasure AI Studio, or other/unknown.
- **Model:** model name if exposed; otherwise `unknown`.
- **Available interfaces:** TD/`tdx` access, artifact read/write, structured output, and user confirmation.
- **Input artifacts:** existing architecture, glossary, schema export, or parent-segment name.
- **Requested mode:** Discovery or Production.

Assess capabilities by demonstrated behavior and available tools, not vendor name.

### Capability tiers

| Tier | Demonstrated capabilities | Suitable use |
|---|---|---|
| 1 — Basic | Follow direct instructions; format Markdown; extract from one small source | Draft formatting only |
| 2 — Standard | Reliable tool use; multi-turn state; inspect multiple moderate sources; separate facts from assumptions | Discovery mode |
| 3 — Advanced | Cross-source synthesis; SQL/schema reasoning; contradiction and provenance tracking; long context | Production mode |
| 4 — High-assurance | Long-horizon tool workflows; identity/join and governance analysis; independent validation | High-risk production review |

Require Tier 2 for Discovery and Tier 3 for Production. Tier labels are capability requirements, not provider rankings. The runtime or user may supply a trusted model-tier classification. Otherwise, evaluate only capabilities observable during preflight; do not claim hidden self-knowledge about the model. If the required capability cannot be demonstrated, stop with `MODEL_CAPABILITY_INSUFFICIENT`. If a concrete recommendation is needed, ask the runtime for an available model that meets the required tier; do not depend on a vendor or model family.

Set the preflight result to:

- `READY`
- `READY_WITH_LIMITATIONS` — list limitations, skip only the affected checks, and reduce readiness; do not emit a blocking exception unless a required gate cannot be completed
- `BLOCKED` — emit the exception format below and stop the affected step

Do not claim a capability merely because a model is marketed as supporting it. If it cannot preserve the evidence ledger, reconcile schemas, or follow tool results accurately, downgrade or stop.

### Runtime and authentication handling

- **Treasure Work:** use available tools or local `tdx`. On 401/403 or expired access, tell the user to open **Settings** (gear icon) and use **Add Account** or **Re-authenticate**. Do not ask them to type credentials into a terminal or chat.
- **Treasure AI Studio:** use the runtime's credential-request or connected-account flow before commands that require TD access. If no credential interface is available, tell the user to connect an authorized TD credential in Studio. Never request an API key in chat.
- **Other/unknown runtime:** use an available read-only TD connector or ask for a sanitized schema export. Mark live validation unavailable.

If artifacts from multiple runtimes will be merged, retain the source runtime on each evidence record and flag runtime-specific assumptions for review.

## 3. Maintain an Evidence Ledger

Assign every material fact an evidence class:

- `OBSERVED` — returned directly by TD tools or supplied source artifacts.
- `USER_CONFIRMED` — explicitly confirmed by a named participant.
- `INFERRED` — plausible but not verified; include the reasoning.
- `CONTESTED` — sources or participants disagree.
- `UNKNOWN` — not established.

For each fact record:

```text
FACT_ID: ARC-###
CLAIM: [single factual claim]
STATUS: OBSERVED | USER_CONFIRMED | INFERRED | CONTESTED | UNKNOWN
SOURCE: [command and object, artifact section, or participant]
SOURCE_RUNTIME: [runtime]
OBSERVED_AT: [timestamp/date or unknown]
SCOPE: [account, parent segment, database, table, market]
NOTES: [limits, sample size, or conflict]
```

Do not convert `INFERRED` into `OBSERVED`. Do not put an inferred business definition in the architecture map; add it to Glossary Candidates instead.

## 4. Discover the TD Structure

Use a read-only Treasure Data interface. Prefer an available native TD connector or exploration tool in the runtime. If `tdx` is available, adapt syntax to the installed version by checking command help when needed:

```bash
tdx status
tdx ps list
tdx ps desc "[parent segment name]" --json
```

If `tdx` is unavailable, use equivalent read-only TD API/tools. If neither exists, request a sanitized parent-segment/schema export and set live validation to unavailable. Do not block merely because one named interface is absent when an equivalent authorized interface exists.

If JSON is not supported for a command, use its documented structured-output option or capture the text result with its command and timestamp.

For each in-scope output database, use equivalent native tools or these `tdx` commands:

```bash
tdx tables "[database].*" --json
tdx describe "[database].[table]" --json
tdx show "[database].[table]" --limit 3 --json
```

If `tdx show` is unavailable, use a bounded query:

```bash
tdx query "SELECT * FROM \"[database]\".\"[table]\" LIMIT 3" --limit 3 --jsonl
```

Filter behavior-table names directly from structured table-list results using the runtime's JSON/object handling. Do not depend on shell pipelines such as `grep`; those are not portable. If structured filtering is unavailable, retain the full bounded table list and ask the user to identify the relevant behavior tables.

Protect data while sampling:

- Retrieve only the columns and rows needed.
- Avoid displaying raw email, phone, address, names, tokens, or other sensitive values.
- Prefer aggregate profiling for identifiers and PII-like fields.
- If sensitive data is returned, do not reproduce it in the artifact; record only schema and aggregate findings.

## 5. Ask Orienting Questions

Ask only for facts that tools cannot establish. Group closely related questions, and play back the answers before treating them as `USER_CONFIRMED`.

Cover:

- Which parent segments are production, test, deprecated, or planned?
- Which output database and customer table are authoritative?
- Which candidate identifiers represent a person, account, household, device, product, or another entity?
- Which events and behaviors matter to current use cases?
- How are markets or business units represented?
- What are expected refresh frequencies and source systems?
- Who owns the schema and who can approve technical mappings?

If the user supplies a business meaning based only on a name, capture it as `INFERRED` until an owner confirms it.

## 6. Run Production Validation

Skip this section only in Discovery mode and list every skipped check. In Production mode, profile candidate keys and important joins using aggregate queries appropriate to the available SQL engine.

### Key quality

```sql
SELECT
  COUNT(*) AS row_count,
  COUNT_IF(candidate_key IS NULL) AS null_key_rows,
  APPROX_DISTINCT(candidate_key) AS distinct_keys
FROM database.table
```

Record engine-specific substitutions if `COUNT_IF` or `APPROX_DISTINCT` is unavailable. Never claim uniqueness from a small sample. Calculate duplicate rate from full or clearly labelled sampled data.

### Cardinality and categorical fields

```sql
SELECT candidate_field, COUNT(*) AS rows
FROM database.table
GROUP BY 1
ORDER BY rows DESC
LIMIT 50
```

### Join coverage

```sql
SELECT
  COUNT(*) AS left_rows,
  COUNT_IF(r.join_key IS NOT NULL) AS matched_rows,
  COUNT_IF(r.join_key IS NULL) AS unmatched_rows
FROM database.left_table l
LEFT JOIN database.right_table r
  ON l.join_key = r.join_key
```

Before accepting a join, establish:

- Entity represented by each key.
- Null and uniqueness rates on both sides.
- Expected cardinality: 1:1, 1:N, N:1, or N:N.
- Match rate and denominator.
- Market or source-system scope.
- Temporal rules and late-arriving behavior.
- Whether joining can multiply rows.

If a query would scan excessive data, use partition filters, approximate functions, or a documented sample. Label estimated results and their limits.

## 7. Produce `[company]-architecture.md`

Use this structure:

```markdown
# [Company] — TD CDP Architecture Map

- Generated: [date/time]
- Runtime and model: [runtime] / [model or unknown]
- Mode: Discovery | Production
- Readiness: Draft | Pilot-ready input | Blocked
- TD account/site: [value or restricted]
- Technical owner: [name/team/TBD]
- Approver: [name/team/TBD]

## Scope and Limitations
[In-scope objects, sampling, inaccessible areas, and skipped checks]

## Parent Segments
| Name | ID | Lifecycle status | Output database | Evidence ID |

## Primary Output Database
### Customer Table
| Column | Type | Technical role | Business meaning status | Evidence ID |

### Behavior Tables
| Table | Approx. rows | Observed structure | Business meaning status | Evidence ID |

## Identifiers and Join Contracts
| Entity | Key | Table | Null rate | Uniqueness | Cardinality | Join coverage | Validation status | Evidence ID |

## Market Structure
[Observed representation plus unresolved business interpretation]

## Freshness and Lineage
| Object | Source | Expected refresh | Observed freshness | Status | Evidence ID |

## Quality Findings
[Duplicate, null, row-multiplication, stale-data, and sampling warnings]

## Schema Surprises and Gotchas
[Observed mismatch, deprecated/empty fields, or expected-but-missing fields]

## Open Questions and Conflicts
| ID | Question/conflict | Owner | Blocking? | Next action |

## Glossary Candidates
| Object | Candidate type | Why definition is needed | Evidence ID | Suggested owner |

Candidate types: opaque-name, categorical, score-field, behavior-table, metric-field,
identifier, consent-field, market-variant, surprise.

## Evidence Ledger
[ARC records]

## Validation and Approval
- Technical playback completed: [yes/no, date, participant]
- Technical owner approval: [approved/pending/rejected, date]
- Checks skipped: [list]
```

A Discovery artifact must say `Readiness: Draft`. Production can say `Pilot-ready input` only when identity, joins, freshness, provenance, and technical playback are complete. `Pilot-ready input` is an intermediate handoff status, not the final `Pilot-ready` status. This skill never grants final Pilot-ready or Production-ready status; `studio-context-file` applies those gates after the glossary, approvals, and acceptance tests are available.

## 8. Validate the Artifact

Before completion, verify:

- Every table, column, and relationship has evidence or is labelled unknown/inferred.
- No business definition was guessed from a name or sample value.
- Sensitive sample values are absent.
- Production-mode key and join claims include denominators and limitations.
- Conflicts remain explicit.
- Glossary Candidates includes every unresolved semantic field and schema surprise.
- Readiness matches completed checks and approvals.

Pass the architecture artifact, Evidence Ledger, and unresolved items to `business-glossary-build`.

## Structured Exceptions

When blocked, preserve valid work and return:

```text
EXECUTION BLOCKED
Code: [stable code]
Step: [step]
Runtime: [runtime]
Required capability: [capability/access]
Observed limitation: [exact limitation or sanitized error]
Work completed: [valid completed work]
Data not collected: [missing work]
Safe fallback: [sanitized export, smaller scope, stronger model, or none]
Recommended model tier: [1–4]
Example model class: [optional, non-binding]
User action: [specific next action]
```

Use these codes where applicable:

- `RUNTIME_UNSUPPORTED`
- `MODEL_CAPABILITY_INSUFFICIENT`
- `MODEL_UNAVAILABLE`
- `TD_AUTH_REQUIRED`
- `TD_ACCESS_DENIED`
- `TD_TOOL_UNAVAILABLE`
- `SOURCE_ARTIFACT_UNREADABLE`
- `CONTEXT_LIMIT_EXCEEDED`
- `VALIDATION_FAILED`
- `OUTPUT_WRITE_UNAVAILABLE`

Authentication failure, authorization failure, missing tool, model unavailability, and insufficient reasoning capability are distinct conditions. Do not substitute one for another or continue by inventing results.
